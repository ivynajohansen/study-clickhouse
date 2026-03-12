# ClickHouse Table Engines

## MergeTree Family (Primary Analytical Engines)

-   **MergeTree** -- Core engine for large analytical datasets with
    partitioning, sorting, and indexing.
-   **ReplacingMergeTree** -- Automatically replaces rows with the same
    primary key during merges (useful for deduplication).
-   **SummingMergeTree** -- Automatically sums numeric columns with the
    same primary key during merges.
-   **AggregatingMergeTree** -- Stores aggregate states for advanced
    aggregation queries.
-   **CollapsingMergeTree** -- Collapses rows using a sign column to
    support update/delete patterns.
-   **VersionedCollapsingMergeTree** -- Improved collapsing engine that
    handles row versions safely.
-   **GraphiteMergeTree** -- Specialized engine optimized for Graphite
    time‑series data.

### Replicated MergeTree Variants (for clusters)

-   **ReplicatedMergeTree** -- Replicated version of MergeTree for high
    availability clusters.
-   **ReplicatedReplacingMergeTree** -- Replicated version of
    ReplacingMergeTree.
-   **ReplicatedSummingMergeTree** -- Replicated version of
    SummingMergeTree.
-   **ReplicatedAggregatingMergeTree** -- Replicated version of
    AggregatingMergeTree.
-   **ReplicatedCollapsingMergeTree** -- Replicated version of
    CollapsingMergeTree.
-   **ReplicatedVersionedCollapsingMergeTree** -- Replicated version of
    VersionedCollapsingMergeTree.
-   **ReplicatedGraphiteMergeTree** -- Replicated version of
    GraphiteMergeTree.

## Log Family (Simple Storage Engines)

-   **TinyLog** -- Extremely simple storage engine without indexes, best
    for very small datasets.
-   **Log** -- Lightweight append-only engine without indexing.
-   **StripeLog** -- Similar to Log but stores columns separately for
    slightly better read performance.

## Memory and Buffer Engines

-   **Memory** -- Stores all data in RAM. Very fast but data disappears
    after restart.
-   **Buffer** -- Buffers incoming data in memory and periodically
    flushes it to another table.

## Special Purpose Engines

-   **Distributed** -- Allows querying data across multiple servers in a
    cluster.
-   **Merge** -- Provides a virtual table that reads from multiple
    tables with the same structure.
-   **Null** -- Discards all inserted data (useful for testing or
    benchmarking).
-   **Set** -- Stores data in memory for use in IN queries.
-   **Join** -- Stores pre-built JOIN tables for fast lookup joins.
-   **File** -- Reads and writes data from files on disk.
-   **URL** -- Reads data from an HTTP/HTTPS endpoint.
-   **View** -- A virtual table defined by a query.
-   **MaterializedView** -- Automatically populates a table using
    results from another table.
-   **Dictionary** -- Provides access to external dictionaries as
    tables.
-   **KeeperMap** -- Persistent key-value storage backed by ClickHouse
    Keeper.

## Integration Engines (External Systems)

-   **Kafka** -- Consumes streaming data from Apache Kafka.
-   **RabbitMQ** -- Consumes streaming data from RabbitMQ.
-   **MySQL** -- Allows querying MySQL tables directly.
-   **PostgreSQL** -- Allows querying PostgreSQL tables directly.
-   **MongoDB** -- Allows querying MongoDB collections.
-   **JDBC** -- Connects to databases using JDBC drivers.
-   **ODBC** -- Connects to databases using ODBC drivers.
-   **HDFS** -- Reads and writes data stored in Hadoop HDFS.
-   **S3** -- Reads and writes data stored in Amazon S3 or compatible
    storage.
-   **EmbeddedRocksDB** -- Key-value storage backed by RocksDB embedded
    inside ClickHouse.

------------------------------------------------------------------------

## Summary

ClickHouse engines define how data is stored, processed, and integrated
with other systems.\
Most production analytics workloads rely on **MergeTree-based engines**,
while others are used for integrations, streaming ingestion, or special
storage behavior.
