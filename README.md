

We outline the approach to migrate from one ClickHouse instance to another, and to do first-load of Postgres data into ClickHouse.

## Why this matters

**ClickHouse-to-ClickHouse migration** is non-trivial due to scale and consistency challenges. ClickHouse tables often contain billions of rows across hundreds of partitions, making naive approaches prohibitively slow or memory-intensive. Additionally, different table engines (ReplacingMergeTree, AggregatingMergeTree) require special handling to maintain data integrity during migration, especially when the source continues receiving writes.

**Postgres-to-ClickHouse first-load** requires extracting massive volumes from a production database without degrading online performance and while staying withing SLA. A naive `SELECT *` would lock tables, spike CPU/memory, and slow down production queries. The challenge is streaming large result sets with extremely low latency while minimizing impact on the online system.

---

## ClickHouse-to-ClickHouse migration

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
3. Convert data to Parquet format (Parquet lends performance gains and Clickhouse has extensive support for Parquet)
4. Ingest data into target instance

**Advantages:** Better parallelism and more control over data transfer.

---

## Ingestion Strategy

### Data Export
- Use `clickhouse-connect` to select data, using the Arrow APIs
- Write to Parquet files for efficient storage and transfer

#### Parallelising Export
- Partition source tables to parallelize export, with a worker handling each partition
- Use checkpointing strategy to avoid rework when workers fail

### Data Import
- Use **async insert** mode
- Let the server handle batching internally for optimal performance

#### Configuration for Import Optimization

```
max_insert_threads = ~half of available CPU cores
peak_memory_usage_in_bytes = entire RAM for isolated ingestion / 50% RAM or less if running concurrent tasks
min_insert_block_size_bytes = peak_memory_usage_in_bytes / (~3 × max_insert_threads)
```
- Reserves remaining cores for background merges. Parts are merged to achieve 1.5 GB per part.
- Prevents resource contention as more parts will require more merges

---

## Table-Specific Considerations

### MergeTree Tables

**Strategy:**
- Partition on sort key
- Execute series of SELECT queries per partition

**Sort Key Types:**

1. **Timestamp-based** (`ts`)
   - Natural chronological partitioning
   - Fine-grained partition size based on timestamp granularity

2. **Low cardinality column** (e.g., country, state)
   - Requires additional partitioning logic to control export-partition size

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

### Additional Migration Checklist

- **Indexes** (primary, secondary, skip indexes)
- **Materialized views**

---

## Postgres-to-ClickHouse first-load
Use `postgres_scanner` DuckDB extension, which is *extremely* efficient (parallelizes internally, uses binary protocol to reduce serde cost) in downloading a Postgres table into Parquet files on filesystem

1. Execute on DuckDB: `COPY (SELECT * FROM postgres_scan('dbname=mydb', 'public', 'mytable')) TO 'mytable.parquet' (FORMAT parquet);` [Ref: https://duckdb.org/2022/09/30/postgres-scanner]
2. Ingest the Parquet files into Clickhouse

