# clickhouse-migration
migrate from one clickhouse instance to another


## Migration Approaches

### Approach 1: Simple Partition-Based Migration

**Steps:**
1. Create table schema in target instance
2. Execute `SELECT * FROM <partition>` query
3. Insert data into target

**Limitation:** Partitions can be very large, which limits load parallelism and can cause performance bottleneck.

---

### Approach 2: Explicit Partitioning (Recommended)

**Steps:**
1. Copy table schema to target instance
2. Use cursor-paginated `SELECT *` queries
3. Convert data to Arrow/Parquet format (enables parallelization)
4. Ingest data into target instance

**Advantages:** Better parallelism and more control over data transfer.

---

## Ingestion Strategy

### Data Export
- Use `clickhouse-connect` to select data
- Convert to Arrow format
- Write to Parquet files for efficient storage and transfer

### Data Import
- Use **async insert** mode
- Let the server handle batching internally for optimal performance

---

## Configuration Guidelines

### Insert Thread and Block Size Configuration

**Recommended settings:**

```
max_insert_threads = ~half of available CPU cores
```
- Reserves remaining cores for background merges
- Prevents resource contention

**Memory-based block sizing formula:**

```
min_insert_block_size_bytes = peak_memory_usage_in_bytes / (~3 × max_insert_threads)
```

Where:
- `peak_memory_usage_in_bytes`: 
  - Use all available RAM for isolated ingestion
  - Use 50% or less if running concurrent tasks

**Implementation:**
- Set `min_insert_block_size_rows = 0` (disables row-based threshold)
- Set `max_insert_threads` to chosen value
- Set `min_insert_block_size_bytes` to calculated result from formula

---

## Table-Specific Considerations

### MergeTree Tables

**Strategy:**
- Partition on sort key
- Execute series of SELECT queries per partition

**Sort Key Options:**

1. **Timestamp-based** (`ts`)
   - Natural chronological partitioning
   - Good for time-series data

2. **Low cardinality column** (e.g., country, state)
   - Requires dynamic partitioning logic
   - Useful for geographically distributed data

---

### ReplacingMergeTree & AggregatingMergeTree

**Challenge:** Ensuring data consistency during migration while table continues to receive writes.

**Approaches:**

1. **Using FINAL clause**
   - Limitation: Does not capture data added after query execution
   - Not recommended for active tables

2. **Parking strategy** (Recommended)
   - Temporarily redirect newly ingested data to a separate staging location
   - Migrate historical data
   - Merge parked data after migration completes

---

## Additional Migration Checklist

Don't forget to copy:
- **Indexes** (primary, secondary, skip indexes)
- **Materialized views**
- **Functions** (user-defined functions)
- **Table schemas** (including column defaults, codecs, TTLs)
- **Database schemas**
- **Dictionaries**
