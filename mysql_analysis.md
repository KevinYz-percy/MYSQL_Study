# MySQL `position()` and `rnd_pos()` Deep Analysis

## Overview
This document analyzes how MySQL uses `position()` and `rnd_pos()` methods, specifically focusing on scenarios where range scanning continues after positioning. This is critical for implementing federated-like storage engines that need to understand when complete result set preservation is required.

## Key Finding
MySQL **DOES** resume sequential scans after `rnd_pos()` calls. Storage engines must support the pattern:
```
rnd_init → rnd_pos(saved_position) → rnd_next → rnd_next → ...
```

## Code Locations for Deep Analysis

### 1. Filesort Range Continuation ✅ **ANALYZED**
**File**: `sql/filesort.cc`
**Key Location**: Line 986
```cpp
table->file->position(table->record[0]);
```

**Critical Discovery**: Filesort behavior depends on **addon fields vs row IDs**

#### **Two Distinct Filesort Patterns**:

**A) Using Addon Fields** (`param->using_addon_fields() == true`)
- **When**: Small records, no BLOBs, SELECT queries
- **Mechanism**: Complete row data stored in sort buffer
- **No `position()` calls**: Line 979 - `if (!param->using_addon_fields())`
- **Iterator**: `SortBufferIterator` - reads data directly from memory
- **Result**: **NO `rnd_pos()` calls needed**

**B) Using Row IDs** (`param->using_addon_fields() == false`)
- **When**: Large records, BLOBs, UPDATE/DELETE, forced by `force_sort_rowids`
- **Mechanism**: Only row positions stored in sort buffer
- **`position()` called**: Line 986 - `table->file->position(table->record[0])`
- **Iterator**: `SortBufferIndirectIterator` - **USES `rnd_pos()` TO FETCH ROWS**

#### **Critical Code Path for Federated Engines**:

**During Sort** (`sql/filesort.cc:986`):
```cpp
if (!param->using_addon_fields()) {
    table->file->position(table->record[0]);  // Store row position
}
```

**During Result Retrieval** (`sql/iterators/sorting_iterator.cc:380`):
```cpp
int tmp = table->file->ha_rnd_pos(table->record[0], cache_pos);
```

#### **Key Insight**: 
When filesort uses row IDs, `SortBufferIndirectIterator` calls `rnd_pos()` for **EVERY** sorted row retrieval. This is a **sequential scan through sorted positions**, not random access!

**Pattern**: `rnd_pos(pos1) → rnd_pos(pos2) → rnd_pos(pos3)...` (in sorted order)

**Impact for Federated Engines**: Must preserve complete result sets when `position()` is called during sorting, as each sorted row will be retrieved via `rnd_pos()`.

## **📋 SUMMARY: Filesort Range Continuation Analysis**

### **🔑 Key Discovery**: 
Filesort has **two completely different behaviors** that determine whether your federated engine needs to preserve result sets:

#### **✅ SAFE MODE: Addon Fields** (`using_addon_fields() == true`)
- **When**: Small records, no large BLOBs, typical SELECT queries
- **Behavior**: Complete row data stored in sort buffer, no `position()` calls
- **Result**: **NO `rnd_pos()` calls** → Your federated engine can stream safely
- **Iterator**: `SortBufferIterator` reads directly from memory

#### **⚠️ CRITICAL MODE: Row IDs** (`using_addon_fields() == false`)
- **When**: Large records, BLOBs >70KB, UPDATE/DELETE, fulltext search, weedout operations
- **Behavior**: Three-phase pattern with sequential `rnd_pos()` calls
- **Result**: **Must preserve complete result sets**
- **Iterator**: `SortBufferIndirectIterator` calls `rnd_pos()` for every row

### **📍 Three Phase Code Pattern**:

1. **Phase 1**: `sql/filesort.cc:986` → `table->file->position(table->record[0])`
2. **Phase 2**: `sql/filesort.cc:1595-1598` → Sort positions by ORDER BY criteria  
3. **Phase 3**: `sql/iterators/sorting_iterator.cc:380` → `table->file->ha_rnd_pos()`

### **🎯 Critical Insight**:
"Range scanning" = **Sequential access through sorted positions**, not random access!

### **💡 Optimization Strategy for Your Engine**:
- **Detect addon fields mode** → Stream safely without result set preservation
- **Detect row ID mode** → Preserve complete result sets for positioning
- **Use sequential nature** → Optimize with cursors/bookmarks instead of full memory copies

---

## **🔍 DEEP DIVE: Row ID Mode Analysis**

### **When Row ID Mode is FORCED** (`force_sort_rowids = true`):

#### **1. UPDATE/DELETE Operations with ORDER BY**
**Files**: `sql/sql_update.cc:673`, `sql/sql_delete.cc:538`
```cpp
// Force filesort to sort by position.
fsort.reset(new (thd->mem_root) Filesort(
    thd, {table}, /*keep_buffers=*/false, order, limit,
    /*remove_duplicates=*/false,
    /*force_sort_rowids=*/true, /*unwrap_rollup=*/false));
```
**Reason**: UPDATE/DELETE must position on exact rows to modify them

#### **2. Semi-join Weedout Operations** 
**File**: `sql/sql_select.cc:5124`
```cpp
if (weedout_end != NO_PLAN_IDX && weedout_end > static_cast<plan_idx>(idx)) {
    force_sort_rowids = true;
}
```
**Reason**: Duplicate elimination requires exact row positioning

#### **3. Multiple Consecutive Sorts**
**File**: `sql/sql_executor.cc:3342`
```cpp
// Thus, in this case, we force the first sort to use row IDs,
// so that the result comes from SortFileIndirectIterator or
// SortBufferIndirectIterator. These will both position the cursor
// on the underlying temporary table correctly before returning it,
// so that the successive filesort will save the right row ID
force_sort_rowids = true;
```
**Reason**: Chain of sorts needs consistent positioning

### **When Row ID Mode is NATURAL** (`SortWillBeOnRowId()` returns true):

#### **1. Fulltext Search Operations**
**File**: `sql/filesort.cc:2191-2194`
```cpp
if (table->pos_in_table_list->is_fulltext_searched()) {
    // MATCH() doesn't use the actual value,
    // it just goes and asks the handler directly for the current row.
    // Thus, we need row IDs, so that the row is positioned correctly.
    return true;
}
```

#### **2. Large BLOB Fields** 
**File**: `sql/filesort.cc:2226-2228`
```cpp
if (field->type() == MYSQL_TYPE_BLOB &&
    field->max_packed_col_length() > 70000u) {
    return true;  // mediumblob and longblob force row ID mode
}
```

### **The Sequential `rnd_pos()` Pattern**

**Key Pattern in `SortBufferIndirectIterator::Read()`**:
```cpp
int SortBufferIndirectIterator::Read() {
  for (;;) {
    if (m_cache_pos == m_cache_end) return -1; /* End of file */
    uchar *cache_pos = m_cache_pos;
    m_cache_pos += m_sum_ref_length;  // Move to next position

    for (TABLE *table : m_tables) {
        // HERE IS THE CRITICAL CALL:
        int tmp = table->file->ha_rnd_pos(table->record[0], cache_pos);
        cache_pos += table->file->ref_length;
        // Error handling...
    }
    return 0;  // Success - row retrieved
  }
}
```

### **🎯 Critical Insight for Federated Engines**:

**This is NOT random access!** It's **sequential access through a sorted list of positions**:

## **📍 Three Phase Code Locations**:

#### **Phase 1: Store Row Positions** (`position()` calls during sort)
**File**: `sql/filesort.cc:979-987`
```cpp
// Note where we are, for the case where we are not using addon fields.
if (!param->using_addon_fields()) {
  for (TABLE *table : tables) {
    if (!can_call_position(table)) {
      continue;
    }
    if (table->pos_in_table_list == nullptr ||
        (table->pos_in_table_list->map() & tables_to_get_rowid_for)) {
      table->file->position(table->record[0]);  // 🔑 PHASE 1: Store position
    }
  }
}
```
**Function**: `make_sortkey()` - called for each row during sorting
**Purpose**: Store row positions in sort buffer alongside sort keys

#### **Phase 2: Sort Positions by Criteria**
**File**: `sql/filesort.cc:1595-1598` (sort buffer processing)
```cpp
count = table_sort->sort_buffer(param, count, param->max_rows);
sort_result->found_records = count;

if (param->using_addon_fields()) {
  sort_result->sorted_result_in_fsbuf = true;
  return false;
}
// If not using addon fields, positions are now sorted
```
**Location**: Multiple sorting algorithms (`std::sort`, `std::stable_sort`)
**Purpose**: Sort the buffer containing sort keys + row positions by ORDER BY criteria

#### **Phase 3: Sequential `rnd_pos()` Retrieval**
**File**: `sql/iterators/sorting_iterator.cc:363-380`
```cpp
int SortBufferIndirectIterator::Read() {
  for (;;) {
    if (m_cache_pos == m_cache_end) return -1; /* End of file */
    uchar *cache_pos = m_cache_pos;
    m_cache_pos += m_sum_ref_length;  // 🔑 Move to next sorted position

    bool skip = false;
    for (TABLE *table : m_tables) {
      // 🔑 PHASE 3: Retrieve row using stored position
      int tmp = table->file->ha_rnd_pos(table->record[0], cache_pos);
      cache_pos += table->file->ref_length;
      if (tmp != 0) {
        return HandleError(thd(), table, tmp);
      }
    }
    return 0;  // Success - row retrieved in sorted order
  }
}
```
**Function**: Iterator pattern - called repeatedly to fetch sorted results
**Purpose**: Use stored positions to retrieve rows in sorted order

## **🔄 The Complete Flow**:

1. **During Sort**: `position()` called for each row → positions stored in sort buffer
2. **After Sort**: Positions are sorted according to ORDER BY criteria  
3. **During Retrieval**: `SortBufferIndirectIterator` calls `rnd_pos()` for each position **in sorted order**

**Example Flow**:
```
Original order: Row A, Row B, Row C
Sort positions: pos_A, pos_B, pos_C  
Sorted by column: pos_C, pos_A, pos_B
Retrieval calls: rnd_pos(pos_C) → rnd_pos(pos_A) → rnd_pos(pos_B)
```

### **⚠️ Implications for Your Federated Engine**:

1. **Cannot discard result sets** once `position()` is called
2. **Must support repositioning** to any previously saved position
3. **Sequential nature** means you might optimize by maintaining a cursor per result set
4. **Predictable access pattern** allows for some optimizations (prefetching, caching)

---

### 2. MEMORY-to-InnoDB Conversion ✅ **ANALYZED**
**File**: `sql/sql_derived.cc`
**Key Locations**: 
- Line 154: Documents the exact pattern
- Line 163: Explicit requirement

**Documented Pattern**:
```
"may be started from the first record (ha_rnd_init, ha_rnd_next) or from 
the record where the previous scan was ended (position(), ha_rnd_end,
[...], ha_rnd_init, ha_rnd_pos(saved position), ha_rnd_next)."
```

**Explicit Requirement**:
```
"A requirement is that InnoDB is able to start a scan like this: 
rnd_init, rnd_pos(some PK value), rnd_next."
```

**Implementation Function**: `create_ondisk_from_heap()`

## **📋 SUMMARY: MEMORY-to-InnoDB Conversion Analysis**

### **🔑 Key Discovery**: 
When MEMORY tables overflow and convert to InnoDB, active cursors must be repositioned to continue sequential scanning from the same logical position.

#### **⚠️ Critical Scenario: Recursive CTEs with Table Spill**
- **When**: MEMORY table exceeds `tmp_table_size` during recursive CTE processing
- **Behavior**: Active `FollowTailIterator` cursors must maintain scan position
- **Result**: **Must support `rnd_pos()` followed by continued `rnd_next()` calls**

### **📍 Three Phase Code Pattern**:

#### **Phase 1: Track Read Position** (`m_read_rows` counter)
**File**: `sql/iterators/basic_row_iterators.cc:478-483`
```cpp
int err = table()->file->ha_rnd_next(m_record);
if (err) {
  return HandleError(err);
}
++m_read_rows;  // 🔑 PHASE 1: Track how many rows we've read
```
**Function**: `FollowTailIterator::Read()` - increments counter for each row read
**Purpose**: Know logical position in MEMORY table before conversion

#### **Phase 2: Convert Table** (`create_ondisk_from_heap()`)
**File**: `sql/sql_tmp_table.cc:2851-2859`
```cpp
/*
  The table just changed from MEMORY to INNODB. 'table' is a reader and
  had an open cursor to the MEMORY table. We closed the cursor, now need
  to open it to InnoDB and re-position it at the same row as before.
  Row positions (returned by handler::position()) are different in
  MEMORY and InnoDB - so the MEMORY row and InnoDB row have differing
  positions.
  We had read N rows of the MEMORY table, need to re-position our
  cursor after the same N rows in the InnoDB table.
*/
```
**Function**: MEMORY table replaced with InnoDB table containing same data
**Purpose**: Convert storage engine while preserving logical row order

#### **Phase 3: Reposition and Continue** (`reposition_innodb_cursor()`)
**File**: `sql/sql_tmp_table.cc:2940-2951`
```cpp
bool reposition_innodb_cursor(TABLE *table, ha_rows row_num) {
  assert(table->s->db_type() == innodb_hton);
  if (table->file->ha_rnd_init(false)) return true;
  // Per the explanation above, the wanted InnoDB row has PK=row_num.
  uchar rowid_bytes[6];
  encode_innodb_position(rowid_bytes, sizeof(rowid_bytes), row_num);
  /*
    Go to the row, and discard the row. That places the cursor at
    the same row as before the engine conversion, so that rnd_next() will
    read the (row_num+1)th row.
  */
  return table->file->ha_rnd_pos(table->record[0], rowid_bytes);
  // 🔑 PHASE 3: Position cursor, then caller continues with rnd_next()
}
```
**Called from**: `sql/iterators/basic_row_iterators.cc:498`
```cpp
bool FollowTailIterator::RepositionCursorAfterSpillToDisk() {
  if (!m_inited) {
    return false;  // No repositioning needed if not started
  }
  return reposition_innodb_cursor(table(), m_read_rows);
  // After this call, subsequent Read() calls use ha_rnd_next()
}
```

### **🔄 The Complete Flow**:

1. **During MEMORY scan**: `FollowTailIterator` tracks `m_read_rows` (e.g., read 1000 rows)
2. **Table conversion**: MEMORY→InnoDB via `create_ondisk_from_heap()`
3. **Repositioning**: `reposition_innodb_cursor()` positions to row 1000 using `rnd_pos()`
4. **Continue scanning**: Next `Read()` call uses `ha_rnd_next()` to get row 1001

### **🎯 Critical Insight for Federated Engines**:

**This is the EXACT `rnd_pos() → rnd_next()` pattern!**

**Pattern**: `rnd_init() → rnd_pos(row_N_position) → rnd_next() → rnd_next()...`

**Example Flow**:
```
MEMORY table: Read rows 1,2,3...1000 → MEMORY overflow
InnoDB conversion: Copy all 1000 rows to InnoDB  
Repositioning: rnd_pos(InnoDB_position_for_row_1000)
Continue: rnd_next() reads row 1001, rnd_next() reads row 1002...
```

### **⚠️ Implications for Your Federated Engine**:

1. **Must preserve result sets** during table conversions (recursive CTEs)
2. **Cannot assume positioning is random** - it's continuation of sequential scan
3. **InnoDB-specific optimization**: Uses auto-increment PK for positioning
4. **Your engine needs**: Ability to position at logical row number and continue

### **💡 Optimization Strategy**:
- **Detect recursive CTE patterns** → Prepare for potential spill-to-disk
- **Use logical row counters** instead of position() calls when possible
- **Maintain cursors/bookmarks** for efficient repositioning
- **Consider row number-based positioning** like InnoDB's approach

**🔍 Key Files Analyzed**:
- `sql/sql_derived.cc:150-165` - Pattern documentation
- `sql/sql_tmp_table.cc:2606-2951` - Conversion implementation  
- `sql/iterators/basic_row_iterators.cc:470-499` - Iterator repositioning
- `sql/iterators/composite_iterators.cc:1950-1962` - MaterializeIterator coordination

---

### 3. Multi-table UPDATE Range Operations
**File**: `sql/sql_update.cc`
**Key Location**: Line 2704
```cpp
/* call ha_rnd_pos() using rowids from temporary table */
int field_num = 0;
if (PositionScanOnRow(table, table, tmp_table, field_num++)) goto err;
```

**Function**: `PositionScanOnRow()` at line 2157
**Pattern**: Position on specific row using stored row ID from temporary table

**Related Analysis**:
- How row IDs are stored in temporary tables
- Subsequent operations after positioning

**Status**: 🔍 **Ready for Analysis**

---

### 4. Range Optimization with Row ID Retrieval
**File**: `sql/range_optimizer/rowid_ordered_retrieval.h`
**Key Location**: Lines 112-114
```cpp
table->file->ha_rnd_pos(table->record[0], rowid);
```

**Notes from Code**:
- "index merge uses position() instead of ha_rnd_pos()"
- "fetch after the fact" pattern
- Used for retrieving rows after index intersection

**Status**: 🔍 **Ready for Analysis**

---

### 5. Iterator Repositioning Patterns
**File**: `sql/iterators/composite_iterators.cc`
**Key Areas**:
- Hash join spill-to-disk operations (around lines 1940-1950)
- Window function repositioning (around lines 160-170)

**Pattern**: Complex iterators that may need to reposition and continue scanning

**Status**: 🔍 **Ready for Analysis**

---

## Analysis Notes

### Current Understanding
- **Direct `rnd_pos()` calls**: Only found in `sql_insert.cc` (duplicate handling) and `sql_update.cc` (multi-table updates)
- **Indirect usage**: Filesort and other operations call `position()`, then iterators use `rnd_pos()` + `rnd_next()`
- **Critical constraint**: Storage engines must support continued scanning after positioning

### Questions to Investigate
1. How do iterators use stored positions from filesort?
2. What triggers the MEMORY-to-InnoDB conversion and how does repositioning work?
3. In multi-table updates, what happens after `PositionScanOnRow()`?
4. How does range optimization combine index access with row positioning?
5. What are the exact scenarios where `rnd_pos()` → `rnd_next()` is required?

### Implications for Federated-like Engines
- Cannot safely discard result sets if `position()` was called
- Must support positioning followed by continued sequential access
- Need to understand query patterns that trigger positioning requirements

---

## Deep Dive Plan
1. Start with `sql/sql_derived.cc:150-165` for clearest documentation
2. Trace `create_ondisk_from_heap` implementation in `sql/sql_tmp_table.cc`
3. Examine iterator classes in `sql/iterators/basic_row_iterators.h`
4. Follow UPDATE flow in `sql/sql_update.cc:2704`
5. Analyze range optimization patterns

---

## Investigation Status
- [x] Filesort Range Continuation ✅ **COMPLETED**
- [x] MEMORY-to-InnoDB Conversion ✅ **COMPLETED**
- [ ] Multi-table UPDATE Operations  
- [ ] Range Optimization Patterns
- [ ] Iterator Repositioning Patterns

**Next Step**: Choose which area to investigate first
