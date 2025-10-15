# VtTablet Design Document

## Executive Summary

VtTablet is the query execution layer in Vitess that sits between VtGate (the query router) and MySQL. Each VtTablet instance manages a single MySQL database and is responsible for:
- Query execution and planning (single-shard scope)
- Connection pooling and management
- Transaction coordination
- Schema tracking and caching
- Query result caching
- Replication lag monitoring
- Health checking and streaming

**Key Distinction**: VtTablet performs **single-shard, table-level query planning**, while VtGate performs **multi-shard, distributed query planning with cross-shard joins and aggregations**.

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                           VtTablet                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Query Service (gRPC API)                        │  │
│  │  - Execute(), StreamExecute()                                │  │
│  │  - Begin(), Commit(), Rollback()                             │  │
│  └──────────────────┬───────────────────────────────────────────┘  │
│                     │                                               │
│  ┌──────────────────▼───────────────────────────────────────────┐  │
│  │              TabletServer                                    │  │
│  │  - State management (SERVING/NOT_SERVING)                    │  │
│  │  - Request routing and validation                            │  │
│  └───┬──────────┬──────────┬──────────┬──────────┬─────────────┘  │
│      │          │          │          │          │                 │
│  ┌───▼───┐  ┌───▼───┐  ┌───▼───┐  ┌───▼───┐  ┌─▼────────┐        │
│  │Schema │  │Query  │  │  Tx   │  │Repl   │  │VStreamer │        │
│  │Engine │  │Engine │  │Engine │  │Tracker│  │(Binlog)  │        │
│  └───┬───┘  └───┬───┘  └───┬───┘  └───────┘  └──────────┘        │
│      │          │          │                                       │
│  ┌───▼──────────▼──────────▼───────────────────────────────────┐  │
│  │              Connection Pools                                │  │
│  │  - OLTP pool (stateless queries)                             │  │
│  │  - OLAP pool (long-running analytics)                        │  │
│  │  - Transaction pool (stateful connections)                   │  │
│  │  - DBA pool (admin operations)                               │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
│                             │                                       │
│  ┌──────────────────────────▼───────────────────────────────────┐  │
│  │                     MySQL                                    │  │
│  │  - Single database instance                                  │  │
│  │  - InnoDB tables                                             │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. TabletServer (Main Orchestrator)

**Location**: [go/vt/vttablet/tabletserver/tabletserver.go:104-145](go/vt/vttablet/tabletserver/tabletserver.go#L104)

```go
type TabletServer struct {
    exporter               *servenv.Exporter
    config                 *tabletenv.TabletConfig
    stats                  *tabletenv.Stats
    QueryTimeout           atomic.Int64
    topoServer             *topo.Server

    // Core engines
    statelessql  *QueryList      // Stateless query tracking
    statefulql   *QueryList      // Stateful query tracking
    olapql       *QueryList      // OLAP query tracking
    se           *schema.Engine  // Schema management
    rt           *repltracker.ReplTracker  // Replication lag tracking
    vstreamer    *vstreamer.Engine         // Binlog streaming
    tracker      *schema.Tracker           // Schema change tracking
    watcher      *BinlogWatcher            // Binlog event monitoring
    qe           *QueryEngine              // Query execution and caching
    txThrottler  txthrottler.TxThrottler   // Transaction throttling
    te           *TxEngine                 // Transaction management
    messager     *messager.Engine          // Message table support
    hs           *healthStreamer           // Health broadcasting
    lagThrottler *throttle.Throttler       // Lag-based throttling
    tableGC      *gc.TableGC               // Table garbage collection

    sm                *stateManager         // State transitions
    onlineDDLExecutor *onlineddl.Executor  // Online DDL execution

    alias *topodatapb.TabletAlias
    env   *vtenv.Environment
}
```

**Initialization Sequence** ([go/vt/vttablet/tabletserver/tabletserver.go:161-200](go/vt/vttablet/tabletserver/tabletserver.go#L161)):
1. Create sub-engines (schema engine, query engine, tx engine)
2. Initialize connection pools
3. Set up health streamer
4. Configure throttlers
5. Initialize state manager
6. Register debug handlers

**State Management**:
- `NOT_SERVING`: Initial state, tablet not ready
- `SERVING`: Tablet is healthy and serving queries
- `NOT_SERVING_TX_ONLY`: Can only serve existing transactions (during graceful shutdown)
- State transitions managed by `stateManager`

### 2. Query Engine (Plan Caching & Execution)

**Location**: [go/vt/vttablet/tabletserver/query_engine.go:144-150](go/vt/vttablet/tabletserver/query_engine.go#L144)

```go
type QueryEngine struct {
    isOpen atomic.Bool
    env    tabletenv.Env
    se     *schema.Engine

    // Plan caching
    plans         *PlanCache      // LRU cache of query plans
    settings      *SettingsCache  // Connection settings cache

    // Query tracking
    streamQList *QueryList
    queryRuleSources *rules.Map

    // Consolidation
    consolidator      *sync2.Consolidator  // Deduplicates identical in-flight queries
    streamConsolidator *StreamConsolidator

    // Throttling
    maxResultSize     atomic.Int64
    warnResultSize    atomic.Int64
    consolidatorMode  atomic.Value
}
```

**Responsibilities**:
1. **Plan Generation**: Parse SQL and create execution plans
2. **Plan Caching**: LRU cache to avoid repeated parsing
3. **Query Consolidation**: Merge identical concurrent queries
4. **Connection Management**: Get connections from appropriate pool
5. **Result Size Limiting**: Enforce max result set sizes

**Plan Cache Key**: Query string + reserved connection flag + system settings

### 3. Query Executor (Execution Pipeline)

**Location**: [go/vt/vttablet/tabletserver/query_executor.go:54-70](go/vt/vttablet/tabletserver/query_executor.go#L54)

```go
type QueryExecutor struct {
    query          string
    marginComments sqlparser.MarginComments
    bindVars       map[string]*querypb.BindVariable
    connID         int64                    // Transaction/reserved conn ID
    options        *querypb.ExecuteOptions
    plan           *TabletPlan              // Execution plan
    ctx            context.Context
    logStats       *tabletenv.LogStats
    tsv            *TabletServer
    targetTabletType topodatapb.TabletType
    setting        *smartconnpool.Setting
}
```

**Execute() Flow** ([go/vt/vttablet/tabletserver/query_executor.go:125-200](go/vt/vttablet/tabletserver/query_executor.go#L125)):

```
1. checkPermissions()
   └─> Verify table ACLs and user permissions

2. queryThrottler.Throttle()
   └─> Apply rate limiting if configured

3. Route by PlanID:
   ├─> PlanSelect/PlanShow → execSelect()
   │   ├─> Apply row limit (#maxLimit bind var)
   │   ├─> Get connection from pool
   │   ├─> Execute query
   │   └─> Check result size limits
   │
   ├─> PlanInsert/PlanUpdate/PlanDelete → txConnExec()
   │   ├─> Require transaction or reserved connection
   │   ├─> Execute via StatefulConnection
   │   └─> Update row counts
   │
   ├─> PlanDDL → execDDL()
   │   ├─> Apply to MySQL
   │   └─> Reload schema
   │
   ├─> PlanSet → execSet()
   │   └─> Update connection settings
   │
   └─> PlanNextval → execNextval()
       └─> Auto-increment sequence handling

4. Record stats
   ├─> Query count
   ├─> Execution time
   ├─> MySQL time
   ├─> Rows affected/returned
   └─> Error codes
```

### 4. Plan Builder (Single-Shard Query Planning)

**Location**: [go/vt/vttablet/tabletserver/planbuilder/plan.go:155-181](go/vt/vttablet/tabletserver/planbuilder/plan.go#L155)

```go
type Plan struct {
    PlanID PlanType          // Type of query plan
    Table *schema.Table      // Primary table (if single table)
    AllTables []*schema.Table // All accessed tables

    Permissions []Permission  // ACL requirements
    FullQuery *sqlparser.ParsedQuery  // Rewritten query with bind vars

    NextCount evalengine.Expr  // For sequence queries
    WhereClause *sqlparser.ParsedQuery  // For hot row protection
    FullStmt sqlparser.Statement

    NeedsReservedConn bool  // Requires stateful connection
}
```

**PlanTypes** ([go/vt/vttablet/tabletserver/planbuilder/plan.go:46-84](go/vt/vttablet/tabletserver/planbuilder/plan.go#L46)):
```go
const (
    PlanSelect           // SELECT queries
    PlanNextval          // Sequence next value
    PlanSelectImpossible // SELECT with impossible WHERE (e.g., WHERE 1=0)
    PlanSelectLockFunc   // SELECT with GET_LOCK() function
    PlanInsert           // INSERT
    PlanInsertMessage    // INSERT into message table
    PlanUpdate           // UPDATE
    PlanUpdateLimit      // UPDATE with LIMIT
    PlanDelete           // DELETE
    PlanDeleteLimit      // DELETE with LIMIT
    PlanDDL              // DDL statements
    PlanSet              // SET statements
    PlanOtherRead        // SHOW, DESCRIBE, etc.
    PlanOtherAdmin       // REPAIR, OPTIMIZE, etc.
    PlanSelectStream     // Streaming SELECT
    PlanMessageStream    // Message table streaming
    PlanSavepoint        // SAVEPOINT
    PlanRelease          // RELEASE SAVEPOINT
    PlanSRollback        // ROLLBACK TO SAVEPOINT
    PlanFlush            // FLUSH
    PlanLoad             // LOAD DATA
    PlanCallProc         // CALL procedure
    PlanAlterMigration   // ALTER VITESS_MIGRATION
    ...
)
```

**Build() Function** ([go/vt/vttablet/tabletserver/planbuilder/plan.go:205-265](go/vt/vttablet/tabletserver/planbuilder/plan.go#L205)):
```go
func Build(env *vtenv.Environment, statement sqlparser.Statement, tables map[string]*schema.Table, dbName string, noRowsLimit bool) (plan *Plan, err error) {
    switch stmt := statement.(type) {
    case *sqlparser.Select:
        plan, err = analyzeSelect(env, stmt, tables, noRowsLimit)
    case *sqlparser.Insert:
        plan, err = analyzeInsert(stmt, tables)
    case *sqlparser.Update:
        plan, err = analyzeUpdate(stmt, tables)
    case *sqlparser.Delete:
        plan, err = analyzeDelete(stmt, tables)
    case *sqlparser.Set:
        plan = analyzeSet(stmt)
    case sqlparser.DDLStatement:
        plan, err = analyzeDDL(stmt)
    // ... more cases
    }

    // Set permissions and table references
    plan.AllTables = lookupAllTables(statement, tables)
    plan.Permissions = BuildPermissions(statement)
    return plan, nil
}
```

**Key Characteristics**:
- **Table-scoped**: Plans operate on specific tables in the schema
- **Simple rewrites**: Convert to parameterized queries with bind variables
- **No distributed logic**: No joins across shards, aggregations across tablets
- **Direct execution**: Plans map 1:1 to MySQL queries (with minor rewrites)

### 5. TabletPlan (Wrapped Plan with Stats)

**Location**: [go/vt/vttablet/tabletserver/query_engine.go:58-72](go/vt/vttablet/tabletserver/query_engine.go#L58)

```go
type TabletPlan struct {
    *planbuilder.Plan         // Embedded plan
    Original   string         // Original query text
    Rules      *rules.Rules   // Query rules applied
    Authorized []*tableacl.ACLResult  // ACL results

    // Statistics (atomically updated)
    QueryCount   uint64  // Number of executions
    Time         uint64  // Total execution time (ns)
    MysqlTime    uint64  // Time spent in MySQL (ns)
    RowsAffected uint64  // Total rows affected
    RowsReturned uint64  // Total rows returned
    ErrorCount   uint64  // Number of errors
}
```

**Stats Tracking** ([go/vt/vttablet/tabletserver/query_engine.go:75-82](go/vt/vttablet/tabletserver/query_engine.go#L75)):
```go
func (ep *TabletPlan) AddStats(queryCount uint64, duration, mysqlTime time.Duration, rowsAffected, rowsReturned, errorCount uint64) {
    atomic.AddUint64(&ep.QueryCount, queryCount)
    atomic.AddUint64(&ep.Time, uint64(duration))
    atomic.AddUint64(&ep.MysqlTime, uint64(mysqlTime))
    atomic.AddUint64(&ep.RowsAffected, rowsAffected)
    atomic.AddUint64(&ep.RowsReturned, rowsReturned)
    atomic.AddUint64(&ep.ErrorCount, errorCount)
}
```

## VtTablet Query Planning vs VtGate Query Planning

### Architectural Difference

```
┌─────────────────────────────────────────────────────────────┐
│                      VtGate                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Distributed Query Planning                            │ │
│  │  - Multi-shard awareness                               │ │
│  │  - Cross-shard joins                                   │ │
│  │  - Distributed aggregations                            │ │
│  │  - Scatter/gather operations                           │ │
│  │  - Vindex-based routing                                │ │
│  └────────────────────────────────────────────────────────┘ │
│           ▼              ▼              ▼                    │
│      [Shard -80]    [Shard 80-]    [Shard unsharded]        │
└─────────────────────────────────────────────────────────────┘
                 ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  VtTablet 1  │ │  VtTablet 2  │ │  VtTablet 3  │
│ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │
│ │ Single-  │ │ │ │ Single-  │ │ │ │ Single-  │ │
│ │ Shard    │ │ │ │ Shard    │ │ │ │ Shard    │ │
│ │ Planning │ │ │ │ Planning │ │ │ │ Planning │ │
│ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │
│  MySQL DB 1  │ │  MySQL DB 2  │ │  MySQL DB 3  │
└──────────────┘ └──────────────┘ └──────────────┘
```

### VtTablet Plan Builder

**Scope**: Single shard, single database
**Input**: SQL statement, local schema
**Output**: Simple execution plan

**Example**:
```sql
SELECT * FROM users WHERE user_id = 123
```

**VtTablet Processing**:
1. Parse SQL → AST
2. Identify table: `users`
3. Check table exists in local schema
4. Determine plan type: `PlanSelect`
5. Rewrite with bind var: `SELECT * FROM users WHERE user_id = :vtg1`
6. Store bind var: `{":vtg1": 123}`
7. Check permissions on `users` table
8. Return plan with:
   - PlanID: `PlanSelect`
   - FullQuery: `SELECT * FROM users WHERE user_id = :vtg1`
   - Table: `users` (schema.Table reference)
   - Permissions: `[{TableName: "users", Role: READER}]`

**Execution**:
- Get connection from OLTP pool
- Execute: `SELECT * FROM users WHERE user_id = 123`
- Return results directly to VtGate

### VtGate Plan Builder

**Location**: [go/vt/vtgate/planbuilder/](go/vt/vtgate/planbuilder/)
**Scope**: Multi-shard, distributed
**Input**: SQL statement, VSchema (virtual schema with sharding info)
**Output**: Complex execution primitive tree

**Example** (same query):
```sql
SELECT * FROM users WHERE user_id = 123
```

**VtGate Processing**:
1. Parse SQL → AST
2. Consult VSchema: `users` is sharded by `user_id` using `hash` vindex
3. Compute vindex value: `hash(123)` → determines shard (e.g., `-80`)
4. Create execution plan:
   ```go
   &engine.Route{
       RoutingParameters: &engine.RoutingParameters{
           Opcode: EqualUnique,  // Single shard lookup
           Keyspace: "main",
           Vindex: hashVindex,
           Values: []sqltypes.Value{sqltypes.NewInt64(123)},
       },
       Query: "SELECT * FROM users WHERE user_id = :vtg1",
   }
   ```
5. Plan type: `PlanPassthrough` (single shard)

**Execution**:
- Route to shard `-80`
- Send to VtTablet managing that shard
- VtTablet does its own planning (as above)
- Return results to VtGate
- VtGate returns to client

### Complex Query Example (Multi-Shard Join)

```sql
SELECT u.name, o.order_id
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.country = 'US'
```

**VtGate Planning** ([go/vt/vtgate/engine/](go/vt/vtgate/engine/)):

```go
&engine.Join{
    Opcode: LeftJoin,
    Left: &engine.Route{  // Scatter to all shards for users
        RoutingParameters: &engine.RoutingParameters{
            Opcode: Scatter,
            Keyspace: "main",
        },
        Query: "SELECT u.user_id, u.name FROM users u WHERE u.country = 'US'",
    },
    Right: &engine.Route{  // Routed lookups for orders
        RoutingParameters: &engine.RoutingParameters{
            Opcode: IN,
            Keyspace: "main",
            Vindex: hashVindex,
            Values: []sqltypes.Value{/* user_ids from left */},
        },
        Query: "SELECT o.user_id, o.order_id FROM orders o WHERE o.user_id IN ::user_ids",
    },
    Vars: map[string]int{"user_ids": 0},  // Bind left results to right query
}
```

**VtGate Execution**:
1. Execute left Route (scatter to all user shards)
2. Collect all `user_id` values from results
3. Compute shard routing for each `user_id`
4. Execute right Route with batched IN query per shard
5. Perform in-memory join on `user_id`
6. Return combined results

**VtTablet Role**:
- VtTablet on shard `-80` receives: `SELECT u.user_id, u.name FROM users u WHERE u.country = 'US'`
- Plans as `PlanSelect`
- Executes against local MySQL
- Returns partial results
- **No knowledge** this is part of a join!

### Plan Type Comparison

| Aspect | VtTablet PlanBuilder | VtGate PlanBuilder |
|--------|----------------------|---------------------|
| **Scope** | Single shard, single table | Multi-shard, distributed |
| **Schema** | Actual MySQL schema | VSchema (virtual schema) |
| **Plan Types** | 30+ simple types (Select, Insert, Update, DDL, etc.) | 13 high-level types + primitives |
| **Routing** | N/A (already at destination) | Vindex-based shard routing |
| **Joins** | Local joins only | Cross-shard joins |
| **Aggregations** | Direct MySQL execution | Distributed aggregation (scatter-gather) |
| **Complexity** | Simple 1:1 query mapping | Complex primitive trees |
| **Caching** | Plan cache by query string | Plan cache by query + keyspace + tablet type |
| **Output** | `*planbuilder.Plan` → direct MySQL query | `engine.Primitive` → execution tree |

### VtGate Engine Primitives

**Location**: [go/vt/vtgate/engine/](go/vt/vtgate/engine/)

**Key Primitives**:
- **Route**: Send query to specific shard(s)
- **Join**: Perform distributed join
- **Aggregate**: Distributed aggregation (SUM, COUNT, GROUP BY)
- **OrderedAggregate**: Aggregation with ordering
- **Limit**: Apply LIMIT across shards
- **Distinct**: Remove duplicates across shards
- **Insert/Update/Delete**: DML with vindex updates
- **Send**: Low-level shard targeting
- **Concatenate**: Union results from multiple routes
- **MergeSort**: Merge sorted results from shards
- **VindexLookup**: Lookup routing via vindex

**Plan Structure** ([go/vt/vtgate/engine/plan.go:42-60](go/vt/vtgate/engine/plan.go#L42)):
```go
type Plan struct {
    Type         PlanType              // Passthrough, Scatter, Join, Complex
    QueryType    sqlparser.StatementType
    Original     string
    Instructions Primitive             // Tree of execution primitives
    BindVarNeeds *sqlparser.BindVarNeeds
    Warnings     []*query.QueryWarning
    TablesUsed   []string

    // Stats
    ExecCount    uint64
    ExecTime     uint64
    ShardQueries uint64
    RowsReturned uint64
    RowsAffected uint64
    Errors       uint64
}
```

**VtGate PlanTypes** ([go/vt/vtgate/engine/plan.go:75-88](go/vt/vtgate/engine/plan.go#L75)):
```go
const (
    PlanLocal        // No database access (e.g., SELECT 1)
    PlanPassthrough  // Single shard, direct passthrough
    PlanMultiShard   // Multiple shards, simple routing
    PlanLookup       // Requires vindex lookup
    PlanScatter      // All shards
    PlanJoinOp       // Cross-shard join
    PlanForeignKey   // Foreign key operation
    PlanComplex      // Complex multi-primitive plan
    PlanOnlineDDL    // Online DDL workflow
    PlanDirectDDL    // Direct DDL execution
    PlanTransaction  // Transaction management
    PlanTopoOp       // Topology operation
)
```

## Transaction Management

### TxEngine (Transaction Coordinator)

**Location**: [go/vt/vttablet/tabletserver/tx_engine.go](go/vt/vttablet/tabletserver/tx_engine.go)

```go
type TxEngine struct {
    env    tabletenv.Env
    txPool *TxPool           // Pool of transaction connections

    preparedPool     *TxPrepPool  // Prepared transactions (2PC)
    twoPC            *TwoPC       // Two-phase commit coordinator
    txThrottler      txthrottler.TxThrottler
}
```

**Transaction Flow**:
1. **Begin**: Allocate connection from TxPool, assign transaction ID
2. **Execute Queries**: All queries on same connection
3. **Commit/Rollback**: Execute transaction end, return connection to pool

**Two-Phase Commit** (for distributed transactions):
1. **Prepare Phase**: Write transaction to _vt.dt_state table
2. **Commit Phase**: Execute MySQL COMMIT
3. **Resolve Phase**: Mark transaction resolved in _vt.dt_state

### TxPool (Stateful Connections)

**Location**: [go/vt/vttablet/tabletserver/tx_pool.go](go/vt/vttablet/tabletserver/tx_pool.go)

```go
type TxPool struct {
    env           tabletenv.Env
    pool          *connpool.Pool      // Underlying connection pool
    activePool    *pools.Numbered[*StatefulConnection]  // Active transactions

    limiter       txlimiter.TxLimiter  // Rate limiting
    timeout       time.Duration         // Transaction timeout
}
```

**StatefulConnection**:
- Wraps MySQL connection
- Tracks transaction state
- Records query history (for redo on failure)
- Supports reserved connections (LOCK TABLES, GET_LOCK, etc.)

## Connection Pooling

**Pools** ([go/vt/vttablet/tabletserver/tabletserver.go:161-200](go/vt/vttablet/tabletserver/tabletserver.go#L161)):

1. **OLTP Pool** (`qe.conns`):
   - Stateless queries
   - Short-lived
   - Size: configurable (default 16)
   - Timeout: query timeout (default 30s)

2. **OLAP Pool** (`qe.streamConns`):
   - Long-running analytics
   - Streaming results
   - Separate sizing
   - Longer timeout

3. **Transaction Pool** (`te.txPool`):
   - Stateful transaction connections
   - Reserved for duration of transaction
   - Timeout-based cleanup

4. **DBA Pool**:
   - Administrative operations
   - Schema changes, repairs
   - Privileged access

**Connection Lifecycle**:
```
1. GetConnection()
   ├─> Check pool availability
   ├─> Wait if pool full (with timeout)
   └─> Establish MySQL connection if new

2. Use connection
   └─> Execute queries

3. Return to pool
   ├─> Reset connection state (if needed)
   └─> Make available for reuse
```

## Schema Engine

**Location**: [go/vt/vttablet/tabletserver/schema/](go/vt/vttablet/tabletserver/schema/)

```go
type Engine struct {
    env       tabletenv.Env
    cp        *connpool.Pool
    tables    map[string]*Table    // Schema metadata cache

    historian *historian           // Schema history tracking
    notifiers []notifier           // Change notification handlers

    lastChange int64               // Last schema change timestamp
}
```

**Responsibilities**:
1. **Schema Loading**: Fetch table definitions from MySQL INFORMATION_SCHEMA
2. **Caching**: Maintain in-memory schema representation
3. **Change Detection**: Monitor for schema changes
4. **Notification**: Notify planbuilder of schema changes to invalidate plan cache

**Table Metadata**:
```go
type Table struct {
    Name        sqlparser.IdentifierCS
    Type        TableType  // Normal, View, Sequence, Message
    Columns     []Column
    PKColumns   []int      // Primary key column indices

    CreateTime  int64
    FileSize    uint64
    RowCount    uint64
}
```

**Schema Reload Triggers**:
- Periodic polling (configurable interval)
- Explicit `ReloadSchema` RPC call (from VtCtld)
- DDL execution detection
- Binlog schema change events

## Health Streaming

**Location**: [go/vt/vttablet/tabletserver/health_streamer.go](go/vt/vttablet/tabletserver/health_streamer.go)

```go
type healthStreamer struct {
    env              tabletenv.Env
    ts               *TabletServer
    se               *schema.Engine
    alias            *topodatapb.TabletAlias

    state            *querypb.StreamHealthResponse
    broadcastCh      chan *querypb.StreamHealthResponse
}
```

**Health Metrics**:
- **Tablet Type**: PRIMARY, REPLICA, RDONLY
- **Serving State**: SERVING, NOT_SERVING
- **Replication Lag**: Seconds behind primary
- **QPS**: Queries per second
- **Transaction Pool Availability**: % free connections

**Health Broadcast**:
- Continuously streams health to VtGate
- VtGate uses health to make routing decisions
- Unhealthy tablets excluded from query routing

## Replication Tracker

**Location**: [go/vt/vttablet/tabletserver/repltracker/](go/vt/vttablet/tabletserver/repltracker/)

```go
type ReplTracker struct {
    env       tabletenv.Env
    alias     *topodatapb.TabletAlias
    config    *tabletenv.TabletConfig

    mu        sync.Mutex
    lag       time.Duration     // Current replication lag
    lastCheck time.Time

    hw        *heartbeatWriter  // Heartbeat injection
}
```

**Lag Measurement**:
1. **Heartbeat Method**:
   - Primary writes heartbeat to `_vt.heartbeat` table
   - Replica reads heartbeat table
   - Lag = current time - heartbeat timestamp

2. **SHOW SLAVE STATUS Method**:
   - Query MySQL replication status
   - Use `Seconds_Behind_Master`

**Use Cases**:
- Health reporting to VtGate
- Lag-based throttling (prevent overwhelming replicas)
- Replica selection (prefer low-lag replicas)

## Query Consolidation

**Location**: [go/vt/vttablet/tabletserver/query_engine.go](go/vt/vttablet/tabletserver/query_engine.go)

**Consolidator** (powered by `go/sync2/consolidator.go`):
- Deduplicates identical queries executing concurrently
- First query executes normally
- Subsequent identical queries wait for first to complete
- All waiters receive same result

**Example**:
```
Time  Query 1: SELECT * FROM users WHERE id=1  [executing]
  │   Query 2: SELECT * FROM users WHERE id=1  [waiting]
  │   Query 3: SELECT * FROM users WHERE id=1  [waiting]
  ▼
Time  Query 1: [completes] → Result X
      Query 2: [receives Result X immediately]
      Query 3: [receives Result X immediately]
```

**Configuration**:
- `--queryserver-consolidator`: `enable`, `disable`, `notOnPrimary`
- Can be overridden per-query via ExecuteOptions

**Benefit**: Reduces redundant work during cache stampedes

## Throttling Mechanisms

### 1. Lag-Based Throttling

**Location**: [go/vt/vttablet/tabletserver/throttle/](go/vt/vttablet/tabletserver/throttle/)

- Monitors replication lag
- Rejects queries when lag exceeds threshold
- Prevents replica overload
- Configurable per-app

### 2. Transaction Throttling

**Location**: [go/vt/vttablet/tabletserver/txthrottler/](go/vt/vttablet/tabletserver/txthrottler/)

- Monitors transaction pool availability
- Throttles new transactions when pool near capacity
- Configurable target (e.g., keep 20% free)

### 3. Query Throttling

**Location**: [go/vt/vttablet/tabletserver/querythrottler/](go/vt/vttablet/tabletserver/querythrottler/)

- File-based configuration
- Per-query-pattern rate limits
- Regex matching on queries

## Tablet Manager (Admin RPC Interface)

**Location**: [go/vt/vttablet/tabletmanager/](go/vt/vttablet/tabletmanager/)

Separate component from TabletServer, handles administrative operations:

**Key RPCs**:
- `SetReadOnly` / `SetReadWrite`: Change MySQL read-only state
- `ChangeType`: Change tablet type (PRIMARY ↔ REPLICA)
- `ReloadSchema`: Force schema reload
- `ApplySchema`: Execute schema change
- `Backup` / `RestoreFromBackup`: Backup operations
- `InitPrimary` / `InitReplica`: Replication setup
- `StartReplication` / `StopReplication`: Replication control

**Separation of Concerns**:
- **TabletServer**: Query path (hot path, performance-critical)
- **TabletManager**: Admin path (control plane, less frequent)

## VStreamer (Binlog Streaming)

**Location**: [go/vt/vttablet/tabletserver/vstreamer/](go/vt/vttablet/tabletserver/vstreamer/)

```go
type Engine struct {
    env          tabletenv.Env
    ts           srvtopo.Server
    se           *schema.Engine
    cell         string

    mu           sync.Mutex
    streamers    map[int]*uvstreamer  // Active streams
    streamIdx    int                  // Stream ID counter
}
```

**Purpose**: Stream binlog events for:
- VReplication (MoveTables, Reshard workflows)
- Change data capture (CDC)
- External consumers

**Event Types**:
- `FIELD`: Schema information
- `ROW`: Row changes (INSERT, UPDATE, DELETE)
- `VGTID`: Global transaction ID position
- `DDL`: DDL statements
- `HEARTBEAT`: Keep-alive events

## Monitoring and Debug Endpoints

**Metrics**:
- `/debug/vars`: Go runtime + custom stats
- `/debug/queryz`: Active query list
- `/debug/querylogz`: Query log (sampled)
- `/debug/txlogz`: Transaction log
- `/debug/query_plans`: Cached query plans
- `/debug/query_stats`: Per-plan statistics
- `/debug/schema`: Current schema
- `/debug/health`: Health check endpoint

**Query Stats Export**:
```json
{
  "PlanSelect": {
    "Query": "SELECT * FROM users WHERE id = :vtg1",
    "Table": "users",
    "Count": 12345,
    "Time": 1234567890,
    "MysqlTime": 987654321,
    "RowsAffected": 0,
    "RowsReturned": 12345,
    "ErrorCount": 0
  }
}
```

## Summary: VtTablet vs VtGate Query Planning

### VtTablet Query Planning

**Nature**: **Table-Level, Single-Shard Execution**

✓ **What VtTablet Plans**:
- Simple SELECT, INSERT, UPDATE, DELETE on local tables
- DDL execution
- Stored procedure calls
- SET variable statements
- SHOW/DESCRIBE commands
- Transactions (BEGIN/COMMIT/ROLLBACK)

✗ **What VtTablet Does NOT Plan**:
- Cross-shard queries
- Distributed joins
- Multi-shard aggregations
- Vindex routing decisions
- Scatter-gather operations

**Planning Strategy**:
1. Parse SQL
2. Identify table(s) from local schema
3. Classify into PlanType (30+ types)
4. Rewrite with bind variables
5. Attach permissions
6. Cache plan
7. Execute directly against MySQL

**Key Files**:
- [go/vt/vttablet/tabletserver/planbuilder/plan.go](go/vt/vttablet/tabletserver/planbuilder/plan.go)
- [go/vt/vttablet/tabletserver/query_engine.go](go/vt/vttablet/tabletserver/query_engine.go)
- [go/vt/vttablet/tabletserver/query_executor.go](go/vt/vttablet/tabletserver/query_executor.go)

### VtGate Query Planning

**Nature**: **Distributed, Multi-Shard Orchestration**

✓ **What VtGate Plans**:
- Shard routing via vindexes
- Cross-shard joins (merge join, hash join)
- Distributed aggregations (scatter-gather)
- Multi-shard DML with vindex updates
- Subqueries across shards
- UNION across shards
- Complex WHERE clause routing

✗ **What VtGate Does NOT Plan**:
- Actual MySQL query execution (delegates to VtTablet)
- Connection pooling to MySQL
- Schema management

**Planning Strategy**:
1. Parse SQL
2. Consult VSchema (virtual schema with sharding metadata)
3. Build primitive execution tree (Route → Join → Aggregate → etc.)
4. Determine shard routing via vindex calculations
5. Cache plan
6. Execute primitives (sending subqueries to VtTablets)
7. Merge results in-memory

**Key Files**:
- [go/vt/vtgate/planbuilder/](go/vt/vtgate/planbuilder/)
- [go/vt/vtgate/engine/plan.go](go/vt/vtgate/engine/plan.go)
- [go/vt/vtgate/engine/route.go](go/vt/vtgate/engine/route.go)
- [go/vt/vtgate/engine/join.go](go/vt/vtgate/engine/join.go)
- [go/vt/vtgate/executor.go](go/vt/vtgate/executor.go)

### Division of Responsibility

```
Client Query: SELECT u.name, COUNT(o.id)
              FROM users u JOIN orders o ON u.id = o.user_id
              WHERE u.country = 'US'
              GROUP BY u.name

┌────────────────────────────────────────────────────┐
│                 VtGate Planning                    │
│  1. Parse query                                    │
│  2. VSchema: users sharded by hash(id)             │
│              orders sharded by hash(user_id)       │
│  3. Plan:                                          │
│     Join(                                          │
│       Route(scatter to all shards):                │
│         "SELECT id, name FROM users                │
│          WHERE country='US'",                      │
│       Route(IN routing by user_ids):               │
│         "SELECT user_id, COUNT(*) FROM orders      │
│          WHERE user_id IN (...) GROUP BY user_id"  │
│     )                                              │
│  4. Aggregate(SUM counts, GROUP BY name)           │
└────────────────────────────────────────────────────┘
         │                            │
         ▼                            ▼
┌──────────────────┐         ┌──────────────────┐
│ VtTablet Shard 1 │         │ VtTablet Shard 2 │
│ ┌──────────────┐ │         │ ┌──────────────┐ │
│ │   Planning   │ │         │ │   Planning   │ │
│ │ 1. Parse SQL │ │         │ │ 1. Parse SQL │ │
│ │ 2. Table:    │ │         │ │ 2. Table:    │ │
│ │    users     │ │         │ │    orders    │ │
│ │ 3. PlanType: │ │         │ │ 3. PlanType: │ │
│ │    Select    │ │         │ │    Select    │ │
│ │ 4. Execute   │ │         │ │ 4. Execute   │ │
│ │    on MySQL  │ │         │ │    on MySQL  │ │
│ └──────────────┘ │         │ └──────────────┘ │
│      MySQL 1     │         │      MySQL 2     │
└──────────────────┘         └──────────────────┘
```

**VtGate**: "How do I execute this query across multiple shards?"
**VtTablet**: "How do I execute this query on my local MySQL?"

---

## Conclusion

VtTablet is the **execution layer** that:
1. Manages individual MySQL instances
2. Performs **simple, table-level query planning** for single-shard queries
3. Executes queries efficiently with connection pooling, caching, and throttling
4. Tracks schema, replication lag, and health
5. Streams results and binlog events
6. Provides administrative RPC interface for cluster management

The key insight is the **separation of concerns**:
- **VtGate** handles distributed query planning and routing
- **VtTablet** handles single-shard query execution and MySQL management
- **VtTablet's planner is deliberately simple** because complex distributed logic belongs in VtGate
