# What is Clickhouse?

- Column-oriented SQL DBMS for online analytical processing (OLAP).

## OLAP vs OLTP

<img width="631" height="418" alt="image" src="https://github.com/user-attachments/assets/8362825d-3873-4818-af81-51c4cc6d28f0" />

Benefit of column-oriented DB:
- Harder to restore single rows, but column operations like filters/aggregation become much faster

### Row-oriented
<img width="630" height="258" alt="image" src="https://github.com/user-attachments/assets/42740876-4353-48f1-88ad-2eb68d36fd06" />

### Column-oriented
<img width="630" height="258" alt="image" src="https://github.com/user-attachments/assets/84a00a5e-c913-45b7-84ee-06ceb8ab2dd8" />

## Architecture

<img width="1024" height="655" alt="image" src="https://github.com/user-attachments/assets/48546b78-19e3-4c25-a973-7888a766c715" />

**Main Layers**:
1. Query Processing: SQL dialect/ PRQL / KQL
2. Storage
3. Integration

**Access Layer**: manages user sessions and communication with applications

**Storage Layer Table Engines**:
1. MergeTree: LSM Tree -> horizontal sorted parts which are continuously merged
2. Special-purpose: speed up or distribute query execution. Example: Dictionary for caching
3. Virtual table engines: bidirectional data exchange with external systems (relational databases, publish/subscribe systems, key/value stores, data lakes, files in object storage)

**Sharding**: Partitions a table into shards based on expression, to process data sets which exceed the capacity of individual nodes

## Use Cases

1. Real-time analytics

**Example: Vimeo**
- Their original stack used Apache Phoenix on HBase, but it increasingly suffered from GC pauses, server failures, and slow queries as use-cases and data volume grew.
- Clickhouse: ~10× faster queries and 2–3× better storage efficiency
- Real-time session events (loads, plays, etc.) are fed via Apache Spark streaming, which computes deltas before insertion to ClickHouse, reducing complexity and improving performance.
- Materialized views pre-aggregate data (e.g., per video or time window), dramatically speeding up queries

2. Observability

**Example: Ebay**
- Needed a scalable way to handle huge volumes of OLAP to perform log aggregation
- ClickHouse clusters are deployed across multiple regions/data centers using custom Kubernetes operators for automated deployment and management

3. Data warehousing

**Example: Microsoft**
- 

5. Machine learning and GenAI
