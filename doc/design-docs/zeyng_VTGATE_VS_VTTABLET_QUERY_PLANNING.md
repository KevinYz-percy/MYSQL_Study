# VtGate vs VtTablet: Query Planning & Execution Comparison

## Executive Summary

**VtGate** and **VtTablet** have fundamentally different responsibilities in query planning and execution:

- **VtGate**: Distributed query planner and coordinator - determines **WHERE** and **HOW** to execute across multiple shards
- **VtTablet**: Single-shard query executor - determines **WHAT** to execute on local MySQL

This document provides a deep dive into their planning strategies, execution flows, and the division of responsibility.

---

## Table of Contents

1. [Architectural Overview](#architectural-overview)
2. [Query Planning Philosophy](#query-planning-philosophy)
3. [VtGate Query Planning](#vtgate-query-planning)
4. [VtTablet Query Planning](#vttablet-query-planning)
5. [Side-by-Side Comparison](#side-by-side-comparison)
6. [Execution Flow Examples](#execution-flow-examples)
7. [Plan Caching Strategies](#plan-caching-strategies)
8. [Performance Implications](#performance-implications)

---

## Architectural Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                          Client                                 │
│                            │                                     │
│                            │ SQL Query                           │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     VtGate                              │   │
│  │  ┌───────────────────────────────────────────────────┐  │   │
│  │  │  DISTRIBUTED QUERY PLANNING                       │  │   │
│  │  │  ────────────────────────────────────────────     │  │   │
│  │  │  • Parse SQL → AST                                │  │   │
│  │  │  • Consult VSchema (virtual schema)               │  │   │
│  │  │  • Determine shard routing via Vindexes           │  │   │
│  │  │  • Build execution primitive tree:                │  │   │
│  │  │    - Route (which shards?)                        │  │   │
│  │  │    - Join (cross-shard joins)                     │  │   │
│  │  │    - Aggregate (scatter-gather)                   │  │   │
│  │  │    - OrderBy, Limit (merge results)               │  │   │
│  │  │  • Generate per-shard subqueries                  │  │   │
│  │  │  • Cache plan (key: query+keyspace+tablettype)    │  │   │
│  │  └───────────────────────────────────────────────────┘  │   │
│  │                                                           │   │
│  │  EXECUTION COORDINATION                                  │   │
│  │  ────────────────────────                                │   │
│  │  • Execute primitive tree                                │   │
│  │  • Send subqueries to VtTablets                          │   │
│  │  • Collect results                                       │   │
│  │  • Merge/join/aggregate in memory                        │   │
│  │  • Return to client                                      │   │
│  └─────────────┬────────────┬────────────┬──────────────────┘   │
│                │            │            │                       │
│      Subquery1 │  Subquery2 │  Subquery3 │                       │
│                ▼            ▼            ▼                       │
│     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│     │  VtTablet 1  │ │  VtTablet 2  │ │  VtTablet 3  │         │
│     │  (Shard -40) │ │  (Shard 40-80)│ │  (Shard 80-) │         │
│     │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │         │
│     │ │ SINGLE-  │ │ │ │ SINGLE-  │ │ │ │ SINGLE-  │ │         │
│     │ │ SHARD    │ │ │ │ SHARD    │ │ │ │ SHARD    │ │         │
│     │ │ PLANNING │ │ │ │ PLANNING │ │ │ │ PLANNING │ │         │
│     │ │────────  │ │ │ │────────  │ │ │ │────────  │ │         │
│     │ │• Parse   │ │ │ │• Parse   │ │ │ │• Parse   │ │         │
│     │ │• Lookup  │ │ │ │• Lookup  │ │ │ │• Lookup  │ │         │
│     │ │  table in│ │ │ │  table in│ │ │ │  table in│ │         │
│     │ │  local   │ │ │ │  local   │ │ │ │  local   │ │         │
│     │ │  schema  │ │ │ │  schema  │ │ │ │  schema  │ │         │
│     │ │• Classify│ │ │ │• Classify│ │ │ │• Classify│ │         │
│     │ │  plan    │ │ │ │  plan    │ │ │ │  plan    │ │         │
│     │ │  type    │ │ │ │  type    │ │ │ │  type    │ │         │
│     │ │• Rewrite │ │ │ │• Rewrite │ │ │ │• Rewrite │ │         │
│     │ │  w/bind  │ │ │ │  w/bind  │ │ │ │  w/bind  │ │         │
│     │ │  vars    │ │ │ │  vars    │ │ │ │  vars    │ │         │
│     │ │• Check   │ │ │ │• Check   │ │ │ │• Check   │ │         │
│     │ │  ACLs    │ │ │ │  ACLs    │ │ │ │  ACLs    │ │         │
│     │ │• Execute │ │ │ │• Execute │ │ │ │• Execute │ │         │
│     │ │  on      │ │ │ │  on      │ │ │ │  on      │ │         │
│     │ │  MySQL   │ │ │ │  MySQL   │ │ │ │  MySQL   │ │         │
│     │ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │         │
│     │              │ │              │ │              │         │
│     │   MySQL DB   │ │   MySQL DB   │ │   MySQL DB   │         │
│     └──────────────┘ └──────────────┘ └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Query Planning Philosophy

### VtGate Planning Philosophy

**Question**: "How do I distribute this query across multiple shards and combine the results?"

**Responsibilities**:
- ✅ Shard routing (which shards to query?)
- ✅ Query decomposition (how to break up the query?)
- ✅ Cross-shard operations (joins, aggregations, ordering)
- ✅ Result merging (combining data from multiple shards)
- ✅ Vindex lookups (routing based on sharding keys)
- ❌ Direct MySQL execution
- ❌ Connection pooling to MySQL
- ❌ Transaction management with MySQL

**Input**:
- SQL query from client
- VSchema (virtual schema with sharding metadata)
- Target keyspace and tablet type

**Output**:
- Execution primitive tree (plan.Instructions)
- Per-shard subqueries
- Merge/join/aggregate strategy

### VtTablet Planning Philosophy

**Question**: "How do I execute this query on my local MySQL instance?"

**Responsibilities**:
- ✅ Parse and validate SQL
- ✅ Table lookup in local schema
- ✅ Query classification (SELECT, INSERT, UPDATE, etc.)
- ✅ ACL checking
- ✅ Query rewriting with bind variables
- ✅ Direct MySQL execution
- ✅ Connection pool management
- ✅ Transaction coordination
- ❌ Shard routing (already at the right shard)
- ❌ Cross-shard joins
- ❌ Distributed aggregations

**Input**:
- SQL query (usually a subquery from VtGate)
- Local MySQL schema
- Transaction/connection context

**Output**:
- Simple plan (PlanType + rewritten query)
- Direct execution against MySQL
- Result rows

---

## VtGate Query Planning

### VtGate Planner Location

**Code**: [go/vt/vtgate/planbuilder/](go/vt/vtgate/planbuilder/)

### Planning Process

```
┌────────────────────────────────────────────────────────────┐
│  VtGate Planning Pipeline                                  │
└────────────────────────────────────────────────────────────┘

1. PARSE SQL
   ├─> sqlparser.Parse(query)
   └─> Generate AST

2. LOAD VSCHEMA
   ├─> Get keyspace VSchema
   ├─> VSchema contains:
   │   ├─> Sharded tables (users, orders)
   │   ├─> Vindexes (hash, lookup, consistent_lookup)
   │   ├─> Routing rules
   │   └─> Column vindexes (e.g., user_id → hash vindex)

3. SEMANTIC ANALYSIS
   ├─> Resolve table names to keyspaces
   ├─> Identify sharding keys in WHERE clauses
   ├─> Detect cross-shard operations
   └─> Build semantic graph

4. BUILD LOGICAL PLAN (Operators)
   ├─> Create operator tree:
   │   ├─> Route (table access)
   │   ├─> Join (if multiple tables)
   │   ├─> Aggregate (GROUP BY, COUNT, SUM, etc.)
   │   ├─> Sort (ORDER BY)
   │   ├─> Limit (LIMIT clause)
   │   └─> Projection (SELECT columns)
   └─> Optimize operator tree

5. DETERMINE ROUTING
   ├─> For each Route operator:
   │   ├─> Extract vindex values from WHERE
   │   ├─> Compute shard mapping:
   │   │   ├─> EqualUnique: hash(value) → single shard
   │   │   ├─> IN: multiple values → multiple shards
   │   │   ├─> Scatter: no vindex → all shards
   │   │   └─> Lookup: vindex lookup → determine shards
   │   └─> Set routing opcode

6. GENERATE SUBQUERIES
   ├─> For each shard in routing:
   │   ├─> Rewrite query for that shard
   │   ├─> Remove cross-shard logic
   │   └─> Add bind variables
   └─> Store in Route.Query

7. BUILD EXECUTION PRIMITIVES
   ├─> Convert operators to engine.Primitives:
   │   ├─> Route → engine.Route
   │   ├─> Join → engine.Join
   │   ├─> Aggregate → engine.Aggregate
   │   ├─> Sort → engine.MemorySort
   │   └─> Limit → engine.Limit
   └─> Create primitive tree

8. CLASSIFY PLAN TYPE
   ├─> Analyze primitive tree
   ├─> Determine complexity:
   │   ├─> PlanPassthrough: single shard
   │   ├─> PlanScatter: all shards
   │   ├─> PlanJoinOp: cross-shard join
   │   └─> PlanComplex: complex multi-primitive
   └─> Set plan.Type

9. CACHE PLAN
   ├─> Generate cache key:
   │   ├─> Query string
   │   ├─> Keyspace
   │   ├─> Tablet type (PRIMARY/REPLICA)
   │   ├─> Collation
   │   └─> SET_VAR hints
   └─> Store in plan cache
```

### VtGate Plan Structure

**Code**: [go/vt/vtgate/engine/plan.go:42-60](go/vt/vtgate/engine/plan.go#L42)

```go
type Plan struct {
    Type         PlanType              // Passthrough, Scatter, Join, Complex
    QueryType    sqlparser.StatementType
    Original     string                // Original query
    Instructions Primitive             // Tree of execution primitives
    BindVarNeeds *sqlparser.BindVarNeeds
    TablesUsed   []string

    // Stats
    ExecCount    uint64
    ExecTime     uint64
    ShardQueries uint64
    RowsReturned uint64
}
```

### VtGate Execution Primitives

**Code**: [go/vt/vtgate/engine/](go/vt/vtgate/engine/)

| Primitive | Purpose | Example |
|-----------|---------|---------|
| **Route** | Execute query on specific shard(s) | `SELECT * FROM users WHERE id=1` |
| **Join** | Cross-shard join | `SELECT * FROM users u JOIN orders o ON u.id=o.user_id` |
| **Aggregate** | Distributed aggregation | `SELECT COUNT(*) FROM users` (scatter-gather) |
| **OrderedAggregate** | Aggregation with ordering | `SELECT country, COUNT(*) FROM users GROUP BY country` |
| **Limit** | Limit across shards | `SELECT * FROM users LIMIT 10` |
| **Distinct** | Deduplicate across shards | `SELECT DISTINCT country FROM users` |
| **MemorySort** | Merge-sort results | `SELECT * FROM users ORDER BY created_at` |
| **MergeSort** | Streaming merge-sort | Large ORDER BY results |
| **Concatenate** | UNION results | `SELECT * FROM users UNION SELECT * FROM archived_users` |
| **VindexLookup** | Vindex-based routing | Lookup vindex to find target shards |
| **Insert/Update/Delete** | DML with vindex updates | Modify rows and update vindex tables |
| **Send** | Low-level shard targeting | Direct shard specification |

### VtGate Routing Opcodes

**Code**: [go/vt/vtgate/engine/routing.go](go/vt/vtgate/engine/routing.go)

```go
const (
    // No routing needed
    None         // SELECT 1 (no table access)

    // Single shard routing
    EqualUnique  // WHERE id = 1 (hash vindex → single shard)

    // Multi-shard routing
    Equal        // WHERE id IN (1,2,3) → multiple shards
    IN           // Explicitly multiple values
    MultiEqual   // Multiple column equality

    // All shards
    Scatter      // No vindex in WHERE → query all shards

    // Special routing
    ByDestination // Explicit shard targeting (USE `keyspace:shard`)
    DBA          // Database admin queries
    Reference    // Reference table (replicated to all shards)
    Unsharded    // Unsharded keyspace
)
```

---

## VtTablet Query Planning

### VtTablet Planner Location

**Code**: [go/vt/vttablet/tabletserver/planbuilder/](go/vt/vttablet/tabletserver/planbuilder/)

### Planning Process

```
┌────────────────────────────────────────────────────────────┐
│  VtTablet Planning Pipeline                                │
└────────────────────────────────────────────────────────────┘

1. PARSE SQL
   ├─> sqlparser.Parse(query)
   └─> Generate AST

2. LOAD LOCAL SCHEMA
   ├─> Get table definitions from schema engine
   ├─> Schema contains:
   │   ├─> Table structure (columns, types)
   │   ├─> Indexes
   │   ├─> Primary key
   │   └─> Table type (normal, view, sequence, message)

3. CLASSIFY STATEMENT
   ├─> Determine statement type:
   │   ├─> SELECT → analyzeSelect()
   │   ├─> INSERT → analyzeInsert()
   │   ├─> UPDATE → analyzeUpdate()
   │   ├─> DELETE → analyzeDelete()
   │   ├─> DDL → analyzeDDL()
   │   ├─> SET → analyzeSet()
   │   └─> SHOW → analyzeShow()

4. ANALYZE & REWRITE
   ├─> Extract table references
   ├─> Check if tables exist in schema
   ├─> Rewrite query with bind variables:
   │   ├─> WHERE id = 123 → WHERE id = :vtg1
   │   ├─> INSERT VALUES (1,'foo') → INSERT VALUES (:vtg1, :vtg2)
   │   └─> Store bind var mapping
   ├─> Add row limit for SELECT (if needed):
   │   └─> Add WHERE ... LIMIT :maxLimit
   ├─> Extract WHERE clause (for hot row protection)

5. DETERMINE PLAN TYPE
   ├─> Classify into one of 30+ plan types:
   │   ├─> PlanSelect, PlanSelectStream
   │   ├─> PlanInsert, PlanInsertMessage
   │   ├─> PlanUpdate, PlanUpdateLimit
   │   ├─> PlanDelete, PlanDeleteLimit
   │   ├─> PlanDDL
   │   ├─> PlanSet
   │   ├─> PlanShow, PlanOtherRead
   │   └─> etc.

6. BUILD PERMISSIONS
   ├─> For each table accessed:
   │   ├─> Determine required role (READER/WRITER)
   │   └─> Create Permission{TableName, Role}

7. CHECK RESERVED CONNECTION NEED
   ├─> Certain operations need stateful connection:
   │   ├─> GET_LOCK() function
   │   ├─> DDL statements
   │   ├─> FLUSH statements
   │   └─> SET statements
   └─> Set NeedsReservedConn flag

8. CREATE PLAN
   └─> Return Plan{
         PlanID: PlanType,
         Table: primary table,
         AllTables: all referenced tables,
         Permissions: ACL requirements,
         FullQuery: rewritten query with bind vars,
         WhereClause: extracted WHERE (for locking),
         NeedsReservedConn: bool
       }

(NO CACHING AT THIS LEVEL - caching done by QueryEngine)
```

### VtTablet Plan Structure

**Code**: [go/vt/vttablet/tabletserver/planbuilder/plan.go:155-181](go/vt/vttablet/tabletserver/planbuilder/plan.go#L155)

```go
type Plan struct {
    PlanID PlanType                    // Simple classification
    Table *schema.Table                // Primary table
    AllTables []*schema.Table           // All accessed tables
    Permissions []Permission            // ACL requirements
    FullQuery *sqlparser.ParsedQuery    // Rewritten query
    WhereClause *sqlparser.ParsedQuery  // For hot row protection
    FullStmt sqlparser.Statement        // Original statement
    NeedsReservedConn bool              // Requires stateful connection
}
```

### VtTablet Plan Types (30+)

**Code**: [go/vt/vttablet/tabletserver/planbuilder/plan.go:46-84](go/vt/vttablet/tabletserver/planbuilder/plan.go#L46)

```go
const (
    PlanSelect           // Basic SELECT
    PlanNextval          // Sequence: SELECT NEXT VALUE
    PlanSelectImpossible // SELECT with WHERE 1=0
    PlanSelectLockFunc   // SELECT with GET_LOCK()
    PlanInsert           // Basic INSERT
    PlanInsertMessage    // INSERT into message table
    PlanUpdate           // Basic UPDATE
    PlanUpdateLimit      // UPDATE with LIMIT
    PlanDelete           // Basic DELETE
    PlanDeleteLimit      // DELETE with LIMIT
    PlanDDL              // DDL (CREATE, ALTER, DROP)
    PlanSet              // SET variables
    PlanOtherRead        // SHOW, DESCRIBE
    PlanOtherAdmin       // REPAIR, OPTIMIZE
    PlanSelectNoLimit    // SELECT without limit enforcement
    PlanSelectStream     // Streaming SELECT
    PlanMessageStream    // Stream from message table
    PlanSavepoint        // SAVEPOINT
    PlanRelease          // RELEASE SAVEPOINT
    PlanSRollback        // ROLLBACK TO SAVEPOINT
    PlanShow             // SHOW statements
    PlanLoad             // LOAD DATA
    PlanFlush            // FLUSH TABLES
    PlanUnlockTables     // UNLOCK TABLES
    PlanCallProc         // CALL stored_proc()
    PlanAlterMigration   // ALTER VITESS_MIGRATION
    PlanRevertMigration  // REVERT VITESS_MIGRATION
    PlanShowMigrations   // SHOW VITESS_MIGRATIONS
    ...
)
```

### TabletPlan (Wrapped with Stats)

**Code**: [go/vt/vttablet/tabletserver/query_engine.go:58-72](go/vt/vttablet/tabletserver/query_engine.go#L58)

```go
type TabletPlan struct {
    *planbuilder.Plan              // Embedded plan
    Original   string              // Original query
    Rules      *rules.Rules        // Applied query rules
    Authorized []*tableacl.ACLResult

    // Statistics (atomically updated)
    QueryCount   uint64
    Time         uint64
    MysqlTime    uint64
    RowsAffected uint64
    RowsReturned uint64
    ErrorCount   uint64
}
```

---

## Side-by-Side Comparison

### Planning Comparison Table

| Aspect | VtGate | VtTablet |
|--------|--------|----------|
| **Scope** | Multi-shard, distributed | Single-shard, local |
| **Input Schema** | VSchema (virtual, sharding metadata) | MySQL schema (actual tables) |
| **Routing Logic** | ✅ Determines target shards via vindexes | ❌ Already at correct shard |
| **Cross-Shard Joins** | ✅ Builds Join primitives | ❌ Only local joins |
| **Aggregations** | ✅ Scatter-gather (e.g., `SUM` across shards) | ❌ Direct MySQL execution |
| **Plan Complexity** | Complex primitive trees | Simple classification |
| **Plan Types** | 13 high-level types | 30+ granular types |
| **Plan Output** | `engine.Primitive` tree | `planbuilder.Plan` |
| **Query Rewriting** | Subquery generation per shard | Bind variable substitution |
| **Cache Key** | Query + Keyspace + TabletType + Collation | Query + ReservedConn + Settings |
| **Execution** | Coordinates multiple VtTablets | Executes on local MySQL |
| **Result Merging** | ✅ In-memory merge/join/aggregate | ❌ Returns raw MySQL results |

### Code Comparison

#### VtGate Plan Example

```go
// VtGate plan for: SELECT * FROM users WHERE user_id IN (1, 2, 3)
&engine.Route{
    RoutingParameters: &engine.RoutingParameters{
        Opcode: IN,                    // Multi-shard routing
        Keyspace: &vindexes.Keyspace{
            Name: "commerce",
            Sharded: true,
        },
        Vindex: hashVindex,            // Hash vindex for user_id
        Values: []sqltypes.Value{
            sqltypes.NewInt64(1),
            sqltypes.NewInt64(2),
            sqltypes.NewInt64(3),
        },
    },
    Query: "SELECT * FROM users WHERE user_id IN ::__vals",
    TableName: "users",
}
// This will:
// 1. Compute hash(1), hash(2), hash(3)
// 2. Determine which shards each hash maps to
// 3. Group by shard
// 4. Send subquery to each shard
// 5. Merge results
```

#### VtTablet Plan Example

```go
// VtTablet plan for: SELECT * FROM users WHERE user_id = 1
&planbuilder.Plan{
    PlanID: PlanSelect,                // Simple classification
    Table: &schema.Table{
        Name: "users",
        Columns: [...],
        PKColumns: [0],
    },
    AllTables: []*schema.Table{...},
    Permissions: []Permission{
        {TableName: "users", Role: tableacl.READER},
    },
    FullQuery: &sqlparser.ParsedQuery{
        Query: "SELECT * FROM users WHERE user_id = :vtg1 LIMIT :maxLimit",
        BindLocations: [...],
    },
    NeedsReservedConn: false,
}
// This will:
// 1. Get connection from OLTP pool
// 2. Substitute bind vars: user_id=1, maxLimit=10001
// 3. Execute: SELECT * FROM users WHERE user_id = 1 LIMIT 10001
// 4. Return results directly
```

---

## Execution Flow Examples

### Example 1: Simple Single-Shard Query

**Query**: `SELECT name FROM users WHERE user_id = 123`

#### VtGate Processing

```
1. Parse query → AST

2. Consult VSchema:
   └─> users table is sharded by user_id (hash vindex)

3. Build plan:
   ├─> Create Route primitive
   ├─> Opcode: EqualUnique (single shard)
   ├─> Compute: hash(123) → determines shard "-40"
   └─> Subquery: "SELECT name FROM users WHERE user_id = :vtg1"

4. Execute:
   ├─> Send to VtTablet managing shard "-40"
   ├─> Pass bind var: {":vtg1": 123}
   └─> Wait for results

5. Return results to client (no merging needed)
```

#### VtTablet Processing

```
1. Receive query: "SELECT name FROM users WHERE user_id = :vtg1"
   Bind vars: {":vtg1": 123}

2. Check plan cache:
   └─> Cache miss (first time)

3. Build plan:
   ├─> Parse SQL
   ├─> Lookup "users" in local schema ✓
   ├─> PlanType: PlanSelect
   ├─> Rewrite: "SELECT name FROM users WHERE user_id = :vtg1 LIMIT :maxLimit"
   ├─> Permissions: [{users, READER}]
   └─> Cache plan

4. Execute:
   ├─> Check ACLs ✓
   ├─> Get connection from OLTP pool
   ├─> Execute on MySQL: SELECT name FROM users WHERE user_id = 123 LIMIT 10001
   ├─> Fetch results: [{name: "Alice"}]
   └─> Return connection to pool

5. Return results to VtGate
```

**Key Observations**:
- VtGate: Routing decision (which shard?)
- VtTablet: Execution decision (how to query MySQL?)
- VtGate plan: Route primitive with vindex calculation
- VtTablet plan: Simple PlanSelect with table lookup

---

### Example 2: Cross-Shard Join

**Query**:
```sql
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.country = 'US'
```

**Assumptions**:
- `users` sharded by `user_id` (hash)
- `orders` sharded by `user_id` (hash)
- Both tables in keyspace "commerce"

#### VtGate Processing

```
1. Parse query → AST

2. Consult VSchema:
   ├─> users: sharded by user_id
   └─> orders: sharded by user_id (co-located!)

3. Build plan:
   ├─> Operator tree:
   │   └─> Join(
   │         Left: Route(users, scatter),
   │         Right: Route(orders, IN routing)
   │       )
   │
   ├─> Routing analysis:
   │   ├─> users: No user_id in WHERE → must scatter
   │   └─> orders: Will use user_id from join → IN routing
   │
   └─> Generate execution plan:

&engine.Join{
    Opcode: LeftJoin,

    // Phase 1: Scatter to all shards for users
    Left: &engine.Route{
        RoutingParameters: &engine.RoutingParameters{
            Opcode: Scatter,
            Keyspace: commerce,
        },
        Query: "SELECT u.user_id, u.name FROM users u WHERE u.country = 'US'",
    },

    // Phase 2: Lookup orders using user_ids from phase 1
    Right: &engine.Route{
        RoutingParameters: &engine.RoutingParameters{
            Opcode: IN,
            Keyspace: commerce,
            Vindex: hashVindex,
            Values: []sqltypes.Value{/* populated at runtime */},
        },
        Query: "SELECT o.user_id, o.total FROM orders o WHERE o.user_id IN ::user_ids",
    },

    // Join configuration
    Vars: map[string]int{
        "user_ids": 0,  // Extract user_id from left results
    },
    Cols: []int{1, 3},  // u.name (col 1), o.total (col 3)
}

4. Execute:
   ├─> Execute Left Route:
   │   ├─> Send to ALL shards (shard -40, 40-80, 80-)
   │   ├─> VtTablet-40 returns: [{user_id:1, name:"Alice"}, {user_id:5, name:"Bob"}]
   │   ├─> VtTablet-80 returns: [{user_id:2, name:"Carol"}]
   │   └─> VtTablet+80 returns: [{user_id:3, name:"Dave"}]
   │
   ├─> Collect user_ids: [1, 2, 3, 5]
   │
   ├─> Execute Right Route:
   │   ├─> Compute routing:
   │   │   ├─> hash(1) → shard -40
   │   │   ├─> hash(2) → shard 40-80
   │   │   ├─> hash(3) → shard 40-80
   │   │   └─> hash(5) → shard -40
   │   │
   │   ├─> Batch by shard:
   │   │   ├─> Shard -40: WHERE user_id IN (1, 5)
   │   │   └─> Shard 40-80: WHERE user_id IN (2, 3)
   │   │
   │   ├─> Send queries:
   │   │   ├─> VtTablet-40: SELECT ... WHERE user_id IN (1,5)
   │   │   └─> VtTablet-80: SELECT ... WHERE user_id IN (2,3)
   │   │
   │   └─> Collect results:
   │       ├─> VtTablet-40: [{user_id:1, total:100}, {user_id:5, total:200}]
   │       └─> VtTablet-80: [{user_id:2, total:150}]
   │
   └─> Perform in-memory join:
       ├─> Join on user_id
       └─> Result: [
             {name:"Alice", total:100},
             {name:"Bob", total:200},
             {name:"Carol", total:150}
           ]

5. Return joined results to client
```

#### VtTablet Processing (Shard -40)

**Received Query 1** (from Left Route):
```sql
SELECT u.user_id, u.name FROM users u WHERE u.country = 'US'
```

```
1. Check plan cache → Cache miss

2. Build plan:
   ├─> PlanType: PlanSelect
   ├─> Table: users
   ├─> Rewrite: Add LIMIT :maxLimit
   └─> Cache plan

3. Execute on MySQL:
   └─> SELECT u.user_id, u.name FROM users u WHERE u.country = 'US' LIMIT 10001

4. Return: [{user_id:1, name:"Alice"}, {user_id:5, name:"Bob"}]
```

**Received Query 2** (from Right Route):
```sql
SELECT o.user_id, o.total FROM orders o WHERE o.user_id IN (1, 5)
```

```
1. Check plan cache → Cache miss

2. Build plan:
   ├─> PlanType: PlanSelect
   ├─> Table: orders
   ├─> Rewrite: Add LIMIT :maxLimit
   └─> Cache plan

3. Execute on MySQL:
   └─> SELECT o.user_id, o.total FROM orders o WHERE o.user_id IN (1,5) LIMIT 10001

4. Return: [{user_id:1, total:100}, {user_id:5, total:200}]
```

**Key Observations**:
- **VtGate performs the join**: VtTablet has no idea it's part of a join!
- **VtTablet sees two independent SELECTs**: Just executes against local MySQL
- **VtGate coordinates**: Scatter → collect → route → join
- **Complexity**: VtGate (high), VtTablet (low)

---

### Example 3: Distributed Aggregation

**Query**: `SELECT country, COUNT(*) FROM users GROUP BY country`

**Assumptions**: `users` sharded by `user_id` across 3 shards

#### VtGate Processing

```
1. Parse query → AST

2. Consult VSchema:
   └─> users: sharded by user_id
   └─> No user_id in WHERE/GROUP BY → must scatter

3. Build plan:

&engine.OrderedAggregate{
    PreProcess: true,
    Aggregates: []*engine.AggregateParams{
        {Opcode: AggregateCount, Col: 1},  // COUNT(*)
    },
    GroupByKeys: []*engine.GroupByParams{
        {KeyCol: 0},  // country
    },

    Input: &engine.Route{
        RoutingParameters: &engine.RoutingParameters{
            Opcode: Scatter,
            Keyspace: commerce,
        },
        // Each shard does partial aggregation
        Query: "SELECT country, COUNT(*) FROM users GROUP BY country ORDER BY country",
    },
}

4. Execute:
   ├─> Send to ALL shards:
   │   ├─> Shard -40: "SELECT country, COUNT(*) ... GROUP BY country"
   │   ├─> Shard 40-80: "SELECT country, COUNT(*) ... GROUP BY country"
   │   └─> Shard 80-: "SELECT country, COUNT(*) ... GROUP BY country"
   │
   ├─> Collect partial results:
   │   ├─> Shard -40: [{country:"US", count:100}, {country:"UK", count:50}]
   │   ├─> Shard 40-80: [{country:"US", count:150}, {country:"UK", count:75}]
   │   └─> Shard 80-: [{country:"US", count:200}, {country:"UK", count:25}]
   │
   ├─> Merge aggregations (in VtGate):
   │   ├─> Group by country
   │   ├─> Sum counts:
   │   │   ├─> US: 100 + 150 + 200 = 450
   │   │   └─> UK: 50 + 75 + 25 = 150
   │   └─> Result: [{country:"US", count:450}, {country:"UK", count:150}]
   │
   └─> Return to client

5. Return aggregated results
```

#### VtTablet Processing (Each Shard)

**Received Query**:
```sql
SELECT country, COUNT(*) FROM users GROUP BY country ORDER BY country
```

```
1. Check plan cache → Cache hit (likely)

2. Execute plan (PlanSelect):
   ├─> Get connection from OLTP pool
   ├─> Execute on MySQL:
   │   └─> SELECT country, COUNT(*) FROM users GROUP BY country ORDER BY country
   │   └─> MySQL does the local aggregation
   ├─> Return partial results to VtGate

3. Each shard returns its local aggregation:
   ├─> Shard -40: Local users with country counts
   ├─> Shard 40-80: Local users with country counts
   └─> Shard 80-: Local users with country counts
```

**Key Observations**:
- **Scatter-Gather Pattern**: VtGate scatters to all shards, gathers partial results
- **Two-Level Aggregation**:
  1. VtTablet: MySQL does local GROUP BY
  2. VtGate: Merges partial aggregations
- **Push-Down Optimization**: VtGate pushes GROUP BY to MySQL where possible
- **VtTablet sees normal SELECT**: No knowledge of distributed aggregation

---

## Plan Caching Strategies

### VtGate Plan Cache

**Location**: [go/vt/vtgate/engine/plan.go](go/vt/vtgate/engine/plan.go)

**Cache Key**:
```go
type PlanKey struct {
    CurrentKeyspace string                // Target keyspace
    TabletType      topodatapb.TabletType // PRIMARY, REPLICA, RDONLY
    Destination     string                // Explicit shard routing (if any)
    Query           string                // Normalized query
    SetVarComment   string                // SET_VAR hints
    Collation       collations.ID         // Collation ID
}
```

**Cache Implementation**:
- **Type**: LRU cache (Theine library)
- **Size**: Configurable (default: 5000 plans)
- **Eviction**: Least recently used
- **Key Hashing**: 256-bit hash for fast lookup

**Cache Invalidation**:
- VSchema changes (schema reload)
- Manual cache clear

**Example**:
```
Query: "SELECT * FROM users WHERE user_id = ?"

Cache Key:
  Keyspace: "commerce"
  TabletType: REPLICA
  Destination: ""
  Query: "select * from users where user_id = :vtg1"
  SetVarComment: ""
  Collation: 33 (utf8mb4_general_ci)

Cache Hit: Return cached plan
Cache Miss: Build plan → cache → return
```

### VtTablet Plan Cache

**Location**: [go/vt/vttablet/tabletserver/query_engine.go](go/vt/vttablet/tabletserver/query_engine.go)

**Cache Key**:
```go
type PlanCacheKey = theine.StringKey  // Just the query string + flags

// Computed as:
func computeCacheKey(query string, hasReservedConn bool, hasSysSettings bool) string {
    if hasReservedConn {
        query = query + " /*reserved*/"
    }
    if hasSysSettings {
        query = query + " /*settings*/"
    }
    return query
}
```

**Cache Implementation**:
- **Type**: LRU cache (Theine library)
- **Size**: Configurable (default: 5000 plans)
- **Eviction**: Least recently used

**Cache Invalidation**:
- Schema changes (table structure modified)
- Manual cache clear
- Query rule changes

**Example**:
```
Query: "SELECT * FROM users WHERE user_id = :vtg1"

Cache Key: "select * from users where user_id = :vtg1"

Cache Hit: Return TabletPlan (includes stats)
Cache Miss: Build plan → cache → return
```

**Difference from VtGate**:
- Simpler key (just query string)
- No keyspace/tablet type (VtTablet knows its own keyspace)
- No vindex metadata needed

---

## Performance Implications

### VtGate Planning Cost

**Expensive Operations**:
- ✓ VSchema lookup and parsing
- ✓ Vindex computations (especially lookup vindexes)
- ✓ Primitive tree construction
- ✓ Query rewriting for each shard

**Optimizations**:
- Aggressive plan caching
- VSchema caching (rebuilt only on changes)
- Vindex result caching (for lookup vindexes)

**Typical Planning Time**: 1-5ms (cached: <0.1ms)

### VtTablet Planning Cost

**Expensive Operations**:
- ✓ SQL parsing
- ✓ Schema table lookup
- ✓ Query rewriting with bind vars

**Optimizations**:
- Plan caching
- Schema caching (in-memory)
- Consolidated schema reloads

**Typical Planning Time**: 0.5-2ms (cached: <0.05ms)

### Execution Cost Comparison

**VtGate Execution**:
- Network RTT to multiple VtTablets (1-10ms per shard)
- In-memory merging/joining (0.5-5ms for small results)
- Result serialization overhead

**VtTablet Execution**:
- Connection pool acquisition (0.1-1ms)
- MySQL execution (varies by query)
- Result serialization

**Cross-Shard Join Example**:
```
Simple Query (single shard):
  VtGate planning: 1ms
  → VtTablet planning: 0.5ms
  → MySQL execution: 10ms
  → Total: ~12ms

Cross-Shard Join (3 shards):
  VtGate planning: 3ms
  → VtTablet planning (scatter): 3 × 0.5ms = 1.5ms
  → MySQL execution (scatter): 3 × 10ms = 30ms (parallel)
  → VtTablet planning (IN routing): 2 × 0.5ms = 1ms
  → MySQL execution (IN routing): 2 × 5ms = 10ms (parallel)
  → VtGate join: 2ms
  → Total: ~47ms
```

---

## Summary: Key Differences

| Dimension | VtGate | VtTablet |
|-----------|--------|----------|
| **Planning Question** | "How do I route and combine this query across shards?" | "How do I execute this query on local MySQL?" |
| **Scope** | Distributed, multi-shard | Single-shard, local |
| **Schema Source** | VSchema (virtual, routing metadata) | MySQL schema (actual tables) |
| **Routing** | Vindex-based shard selection | N/A (already routed) |
| **Complexity** | High (primitive trees) | Low (simple classification) |
| **Join Support** | Cross-shard joins | Local joins only |
| **Aggregation** | Scatter-gather merge | Direct MySQL |
| **Query Rewriting** | Per-shard subquery generation | Bind variable substitution |
| **Result Handling** | Merge, join, aggregate in memory | Direct MySQL results |
| **Plan Cache Key** | Query + Keyspace + TabletType + Collation | Query + Connection flags |
| **Primary Concern** | Distribution strategy | Execution efficiency |

---

## Conclusion

The query planning division between VtGate and VtTablet is a **fundamental architectural separation**:

**VtGate** = **"Distributed Query Orchestrator"**
- Knows about sharding
- Makes routing decisions
- Coordinates cross-shard operations
- Merges results
- Complex planning

**VtTablet** = **"Single-Shard Query Executor"**
- Knows about local MySQL
- Executes queries efficiently
- Manages connections and transactions
- Returns raw results
- Simple planning

This separation enables:
1. **Scalability**: VtTablets focus on fast local execution
2. **Simplicity**: VtTablets don't need distributed logic
3. **Flexibility**: Change sharding strategy without touching VtTablet
4. **Performance**: Each layer optimized for its role

The VtGate plan is **WHERE and HOW to distribute**, while the VtTablet plan is **WHAT to execute locally**.
