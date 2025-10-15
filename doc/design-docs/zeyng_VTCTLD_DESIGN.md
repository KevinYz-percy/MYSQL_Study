# VtCtld Design Document

## Executive Summary

VtCtld (Vitess Control Daemon) is the control plane component of Vitess responsible for cluster management operations. It exposes both a gRPC API (modern) and REST API (legacy) for administrative tasks such as schema changes, reparenting, backup management, workflow orchestration, and topology manipulation.

Unlike VtGate (which handles query routing) and VtTablet (which executes queries), VtCtld does **not** participate in the query path. It is purely an administrative service.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         VtCtld                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          gRPC API (VtctldServer)                      │  │
│  │  - Modern interface                                   │  │
│  │  - Protocol buffer based                              │  │
│  │  - Used by vtctldclient                               │  │
│  └─────────────────┬─────────────────────────────────────┘  │
│                    │                                         │
│  ┌─────────────────▼─────────────────────────────────────┐  │
│  │          Wrangler (Orchestration Layer)               │  │
│  │  - Manages complex topology operations                │  │
│  │  - Coordinates multi-step workflows                   │  │
│  │  - Thread-safe, reusable across operations            │  │
│  └─────────────┬───────────────┬─────────────────────────┘  │
│                │               │                             │
│  ┌─────────────▼───────┐  ┌───▼──────────────────────────┐  │
│  │  Tablet Manager     │  │   Topology Server            │  │
│  │  Client (TMC)       │  │   - etcd / Zookeeper / etc   │  │
│  │  - RPC to tablets   │  │   - Metadata storage         │  │
│  └─────────────────────┘  └──────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          REST API (ActionRepository)                  │  │
│  │  - Legacy web interface                               │  │
│  │  - Action-based pattern                               │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. VtctldServer (gRPC Interface)

**Location**: [go/vt/vtctl/grpcvtctldserver/server.go:94-110](go/vt/vtctl/grpcvtctldserver/server.go#L94)

The primary interface for VtCtld operations. Implements the protobuf-defined VtctldServer service.

```go
type VtctldServer struct {
    vtctlservicepb.UnimplementedVtctldServer
    ts  *topo.Server                    // Topology server connection
    tmc tmclient.TabletManagerClient    // Client for tablet RPC calls
    ws  *workflow.Server                 // Workflow orchestration
}
```

**Key Methods Include**:
- **Topology Operations**: `AddCellInfo`, `AddCellsAlias`, `GetCellInfo`, `DeleteCellInfo`
- **Tablet Management**: `GetTablet`, `DeleteTablet`, `ChangeTabletType`, `RefreshState`
- **Schema Management**: `ApplySchema`, `GetSchema`, `ValidateSchemaKeyspace`
- **Reparenting**: `PlannedReparentShard`, `EmergencyReparentShard`, `InitShardPrimary`
- **Backup/Restore**: `BackupShard`, `RestoreFromBackup`, `GetBackups`
- **Workflow Operations**: `Workflow`, `MoveTables`, `Reshard`, `VDiff`
- **VSchema Management**: `ApplyVSchema`, `GetVSchema`, `RebuildVSchemaGraph`
- **Routing Rules**: `ApplyRoutingRules`, `GetRoutingRules`

**Design Pattern**: Each RPC method:
1. Validates input parameters
2. Creates a context with appropriate timeout
3. Delegates to Wrangler for complex operations
4. Returns protobuf response

**Example** ([go/vt/vtctl/grpcvtctldserver/server.go:129-152](go/vt/vtctl/grpcvtctldserver/server.go#L129)):
```go
func (s *VtctldServer) AddCellInfo(ctx context.Context, req *vtctldatapb.AddCellInfoRequest) (resp *vtctldatapb.AddCellInfoResponse, err error) {
    span, ctx := trace.NewSpan(ctx, "VtctldServer.AddCellInfo")
    defer span.Finish()
    defer panicHandler(&err)

    if req.CellInfo.Root == "" {
        err = vterrors.Errorf(vtrpcpb.Code_FAILED_PRECONDITION, "CellInfo.Root must be non-empty")
        return nil, err
    }

    ctx, cancel := context.WithTimeout(ctx, topo.RemoteOperationTimeout)
    defer cancel()

    if err = s.ts.CreateCellInfo(ctx, req.Name, req.CellInfo); err != nil {
        return nil, err
    }

    return &vtctldatapb.AddCellInfoResponse{}, nil
}
```

### 2. Wrangler (Orchestration Engine)

**Location**: [go/vt/wrangler/wrangler.go:51-64](go/vt/wrangler/wrangler.go#L51)

Wrangler is the workhorse of VtCtld, managing complex multi-step topology operations.

```go
type Wrangler struct {
    env      *vtenv.Environment          // Environment (parser, collations)
    logger   logutil.Logger               // Logging interface
    ts       *topo.Server                 // Primary topology server
    tmc      tmclient.TabletManagerClient // Tablet communication
    vtctld   vtctlservicepb.VtctldServer  // Back-reference to VtctldServer
    sourceTs *topo.Server                 // Optional source topo (for migrations)
    sem      *semaphore.Weighted          // Concurrency limiting
}
```

**Key Responsibilities**:
- **Reparenting Coordination**: Orchestrates planned and emergency reparents across multiple tablets
- **Schema Change Execution**: Applies DDL across shards with proper validation
- **Backup Workflows**: Coordinates backup/restore operations
- **Tablet Lifecycle**: Manages tablet initialization, deletion, type changes
- **Validation**: Cross-checks topology consistency

**Thread Safety**: Multiple goroutines can use the same Wrangler instance safely, sharing logger/topo/timeout settings.

**Typical Usage Pattern**:
```go
wr := wrangler.New(env, logger, topoServer, tmClient)
err := wr.ValidateKeyspace(ctx, keyspaceName, pingTablets)
```

### 3. Tablet Manager Client (TMC)

**Location**: [go/vt/vttablet/tmclient/](go/vt/vttablet/tmclient/)

TMC provides the RPC interface to communicate with VtTablet's TabletManager component. VtCtld uses TMC to:
- Execute schema changes on individual tablets
- Trigger backups and restores
- Manage replication (start/stop, change source)
- Get tablet status and metadata
- Execute administrative queries

**Interface** ([go/vt/vttablet/tabletmanager/rpc_agent.go:35-150](go/vt/vttablet/tabletmanager/rpc_agent.go#L35)):
```go
type RPCTM interface {
    // Read-only operations
    Ping(ctx context.Context, args string) string
    GetSchema(ctx context.Context, request *tabletmanagerdatapb.GetSchemaRequest) (*tabletmanagerdatapb.SchemaDefinition, error)
    GetPermissions(ctx context.Context) (*tabletmanagerdatapb.Permissions, error)

    // Write operations
    SetReadOnly(ctx context.Context, rdonly bool) error
    ChangeType(ctx context.Context, tabletType topodatapb.TabletType, semiSync bool) error
    RefreshState(ctx context.Context) error
    ReloadSchema(ctx context.Context, waitPosition string) error
    ApplySchema(ctx context.Context, change *tmutils.SchemaChange) (*tabletmanagerdatapb.SchemaChangeResult, error)

    // Replication operations
    PrimaryStatus(ctx context.Context) (*replicationdatapb.PrimaryStatus, error)
    ReplicationStatus(ctx context.Context) (*replicationdatapb.Status, error)
    StopReplication(ctx context.Context) error
    StartReplication(ctx context.Context, semiSync bool) error

    // Reparenting operations
    InitPrimary(ctx context.Context, semiSync bool) (string, error)
    InitReplica(ctx context.Context, parent *topodatapb.TabletAlias, replicationPosition string, timeCreatedNS int64, semiSync bool) error
    ...
}
```

### 4. Action Repository (Legacy REST API)

**Location**: [go/vt/vtctld/action_repository.go:84-102](go/vt/vtctld/action_repository.go#L84)

The legacy web-based interface using an action-based pattern.

```go
type ActionRepository struct {
    env             *vtenv.Environment
    keyspaceActions map[string]actionKeyspaceMethod  // Actions on keyspaces
    shardActions    map[string]actionShardMethod     // Actions on shards
    tabletActions   map[string]actionTabletRecord    // Actions on tablets
    ts              *topo.Server
}
```

**Action Types**:
- **Keyspace Actions**: `ValidateKeyspace`, `ValidateSchemaKeyspace`, `ValidateVersionKeyspace`
- **Shard Actions**: `ValidateShard`, `ValidateSchemaShard`, `ValidateVersionShard`
- **Tablet Actions**: `Ping`, `RefreshState`, `DeleteTablet`, `ReloadSchema`

**Execution Pattern** ([go/vt/vtctld/action_repository.go:123-142](go/vt/vtctld/action_repository.go#L123)):
```go
func (ar *ActionRepository) ApplyKeyspaceAction(ctx context.Context, actionName, keyspace string) *ActionResult {
    result := &ActionResult{Name: actionName, Parameters: keyspace}

    action, ok := ar.keyspaceActions[actionName]
    if !ok {
        result.error("Unknown keyspace action")
        return result
    }

    ctx, cancel := context.WithTimeout(ctx, actionTimeout)
    wr := wrangler.New(ar.env, logutil.NewConsoleLogger(), ar.ts, tmclient.NewTabletManagerClient())
    output, err := action(ctx, wr, keyspace)
    cancel()

    if err != nil {
        result.error(err.Error())
        return result
    }
    result.Output = output
    return result
}
```

**Note**: This is being phased out in favor of the gRPC API.

## Key Operations

### Reparenting

**Planned Reparent** (`PlannedReparentShard`):
1. Validate new primary candidate is healthy
2. Stop writes on current primary
3. Wait for all replicas to catch up to primary's position
4. Promote new primary
5. Point all replicas to new primary
6. Update topology with new primary information

**Emergency Reparent** (`EmergencyReparentShard`):
1. Determine most up-to-date replica (via GTID position)
2. Promote that replica to primary
3. Reconfigure other replicas
4. Update topology

**Implementation**: Uses `reparentutil` package for policy-based decisions.

### Schema Management

**ApplySchema Workflow**:
1. Parse and validate SQL DDL
2. Check if online DDL or direct DDL
3. For direct DDL:
   - Apply to primary tablet first (via TMC)
   - Wait for replication to propagate
   - Optionally apply to all replicas in parallel
4. For online DDL:
   - Submit to Online DDL scheduler
   - Track migration progress
   - Coordinate cutover

**Schema Validation**:
- `ValidateSchemaKeyspace`: Ensures all shards have identical schemas
- `ValidateSchemaShard`: Checks all tablets in shard have same schema

### Workflow Orchestration

**MoveTables/Reshard Operations**:
1. Create VReplication streams on target tablets
2. Monitor copy phase progress
3. Handle schema differences
4. Coordinate traffic cutover (reads then writes)
5. Clean up after completion

**VDiff** (Vertical Diff):
- Compares data between source and target during migrations
- Runs on-demand or scheduled
- Reports discrepancies

### Backup Management

**BackupShard**:
1. Select appropriate tablet for backup (usually replica)
2. Trigger backup via TMC.Backup()
3. Tablet streams backup to configured storage (S3, GCS, Ceph, etc.)
4. Update topology with backup metadata

**RestoreFromBackup**:
1. Fetch backup from storage
2. Restore to tablet's MySQL instance
3. Apply binary logs to reach desired point
4. Mark tablet as ready

## Topology Service Integration

VtCtld is the primary writer to the topology service (topo.Server). It manages:

**Keyspace Information**:
- Keyspace configuration
- Sharding scheme
- Served types per shard

**Shard Information**:
- Shard ranges
- Primary tablet assignment
- Shard state (serving/non-serving)

**Tablet Records**:
- Tablet aliases and hostnames
- Tablet types (PRIMARY, REPLICA, RDONLY)
- Health status

**VSchema**:
- Virtual schema definitions
- Vindexes and routing rules
- Table to keyspace mappings

**SrvVSchema** (Serving VSchema):
- Cached, optimized version of VSchema
- Built per cell for VtGate consumption
- Rebuilt via `RebuildVSchemaGraph`

## VtCtld vs Legacy vtctl

| Aspect | VtCtld | vtctl (CLI) |
|--------|---------|-------------|
| **Interface** | gRPC API (server daemon) | Command-line tool (client) |
| **Lifetime** | Long-running process | Executes single command then exits |
| **State** | Maintains connections to topo | Creates new topo connection per invocation |
| **Performance** | Better for repeated operations | Overhead per command |
| **Use Case** | Programmatic access, automation, UI | Manual administration, scripts |
| **Modern Approach** | `vtctldclient` talks to `vtctld` | Direct topo manipulation |

**Migration Path**:
- Old: `vtctl ApplySchema ...` (direct topo access)
- New: `vtctldclient ApplySchema ...` → gRPC → `vtctld` → orchestration

## Initialization and Startup

**Sequence** ([go/vt/vtctld/vtctld.go:54-138](go/vt/vtctld/vtctld.go#L54)):

```go
func InitVtctld(env *vtenv.Environment, ts *topo.Server) error {
    // 1. Create action repository for REST API
    actionRepo := NewActionRepository(env, ts)

    // 2. Register keyspace actions
    actionRepo.RegisterKeyspaceAction("ValidateKeyspace",
        func(ctx context.Context, wr *wrangler.Wrangler, keyspace string) (string, error) {
            return "", wr.ValidateKeyspace(ctx, keyspace, false)
        })

    // 3. Register shard actions
    actionRepo.RegisterShardAction("ValidateShard",
        func(ctx context.Context, wr *wrangler.Wrangler, keyspace, shard string) (string, error) {
            return "", wr.ValidateShard(ctx, keyspace, shard, false)
        })

    // 4. Register tablet actions
    actionRepo.RegisterTabletAction("Ping", "",
        func(ctx context.Context, wr *wrangler.Wrangler, tabletAlias *topodatapb.TabletAlias) (string, error) {
            ti, err := wr.TopoServer().GetTablet(ctx, tabletAlias)
            if err != nil {
                return "", err
            }
            return "", wr.TabletManagerClient().Ping(ctx, ti.Tablet)
        })

    // 5. Initialize REST API endpoints
    initAPI(context.Background(), ts, actionRepo)

    // 6. Initialize topology explorer endpoint
    initExplorer(ts)

    return nil
}
```

**Main Binary** ([go/cmd/vtctld/cli/](go/cmd/vtctld/cli/)):
1. Parse flags
2. Connect to topology service
3. Initialize VtctldServer
4. Start gRPC server
5. Start HTTP server (for REST API and /debug endpoints)
6. Register with service discovery

## Security and ACL

**Access Control**:
- ACL checks on tablet actions (via `acl` package)
- Role-based permissions (ADMIN, READ, etc.)
- Configurable via `--security_policy` flag

**Authentication**:
- gRPC: mTLS, SASL, or custom auth
- HTTP: Basic auth or custom handlers

## Monitoring and Observability

**Metrics Exposed**:
- Operation counts per RPC method
- Operation latencies
- Error rates
- Topology read/write statistics

**Debug Endpoints**:
- `/debug/status`: Server health
- `/debug/vars`: Go runtime stats
- `/api/keyspaces`: Keyspace list (REST)
- `/api/tablets`: Tablet list (REST)
- `/api/topodata/`: Topology browser (REST)

## Error Handling

**Panic Recovery** ([go/vt/vtctl/grpcvtctldserver/server.go:122-126](go/vt/vtctl/grpcvtctldserver/server.go#L122)):
```go
func panicHandler(err *error) {
    if x := recover(); x != nil {
        *err = fmt.Errorf("uncaught panic: %v from: %v", x, string(debug.Stack()))
    }
}
```

Every RPC method uses `defer panicHandler(&err)` to ensure panics are converted to gRPC errors.

**Timeout Handling**:
- Default action timeout: `topo.RemoteOperationTimeout * 4`
- Per-operation timeouts via context
- Configurable via `--action-timeout` flag

## Summary

VtCtld serves as the **control plane orchestrator** for Vitess clusters:

1. **No Query Path Involvement**: Unlike VtGate/VtTablet, VtCtld never handles user queries
2. **Topology Authority**: Primary writer to topology service
3. **Orchestration Hub**: Coordinates complex multi-step operations via Wrangler
4. **RPC Gateway**: Uses TMC to communicate with individual tablets
5. **Dual APIs**: Modern gRPC (preferred) and legacy REST
6. **Workflow Manager**: Handles migrations, resharding, schema changes
7. **Safety First**: Validation, ACLs, panic recovery, timeouts

**When to use VtCtld**:
- Schema changes across keyspace
- Reparenting operations
- Backup/restore orchestration
- Topology modifications
- Workflow management (MoveTables, Reshard)
- Cluster validation

**When NOT to use VtCtld**:
- Query execution → Use VtGate
- Tablet-local operations → Use Tablet Manager directly
- High-frequency operations → Performance overhead
