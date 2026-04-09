# Performance Test

In this file we'll compare the performance of different types of tables using different types of table engines

## Experiment 1: Summing Use Case

Use case: transaction aggregation (e.g. total sales per product)
Recommended engine: SummingMergeTree

### Create tables

Create base schema:

```
CREATE TABLE exp_sales_summing_base_dataset
(
    transaction_id String,
    product_id String,
    order_date Date,
    quantity UInt32,
    amount Float64
)
ENGINE = MergeTree
ORDER BY (product_id, order_date);
```

Now, create same tables with different table engines for comparison:

SummingMergeTree:

```
CREATE TABLE exp_sales_summing_engine_summingmergetree
(
    product_id String,
    order_date Date,
    quantity UInt32,
    amount Float64
)
ENGINE = SummingMergeTree
ORDER BY (product_id, order_date);
```

MergeTree:

```
CREATE TABLE exp_sales_summing_engine_mergetree
AS exp_sales_summing_engine_summingmergetree
ENGINE = MergeTree
ORDER BY (product_id, order_date);
```

ReplacingMergeTree:

```
CREATE TABLE exp_sales_summing_engine_replacingmergetree
AS exp_sales_summing_engine_summingmergetree
ENGINE = ReplacingMergeTree
ORDER BY (product_id, order_date);
```

AggregatingMergeTree:

```
CREATE TABLE exp_sales_summing_engine_aggregatingmergetree
(
    product_id String,
    order_date Date,
    quantity AggregateFunction(sum, UInt32),
    amount AggregateFunction(sum, Float64)
)
ENGINE = AggregatingMergeTree
ORDER BY (product_id, order_date);
```

### Inserting Data

Now generate data:

```
INSERT INTO exp_sales_summing_base_dataset
SELECT
    toString(number) AS transaction_id,
    concat('P', toString(rand() % 100)) AS product_id,
    today() - (rand() % 30) AS order_date,
    rand() % 5 + 1 AS quantity,
    rand() % 100 AS amount
FROM numbers(1000000);
```

Populate each table:

```
INSERT INTO exp_sales_summing_engine_summingmergetree
SELECT product_id, order_date, quantity, amount
FROM exp_sales_summing_base_dataset;

INSERT INTO exp_sales_summing_engine_mergetree
SELECT product_id, order_date, quantity, amount
FROM exp_sales_summing_base_dataset;

INSERT INTO exp_sales_summing_engine_replacingmergetree
SELECT product_id, order_date, quantity, amount
FROM exp_sales_summing_base_dataset;

INSERT INTO exp_sales_summing_engine_aggregatingmergetree
SELECT
    product_id,
    order_date,
    sumState(quantity),
    sumState(amount)
FROM exp_sales_summing_base_dataset
GROUP BY product_id, order_date;
```

### Running Queries

SummingMergeTree:

```
SELECT
    product_id,
    sum(quantity) AS total_qty,
    sum(amount) AS total_amount
FROM exp_sales_summing_engine_summingmergetree
GROUP BY product_id;
```

MergeTree:

```
SELECT
    product_id,
    sum(quantity),
    sum(amount)
FROM exp_sales_summing_engine_mergetree
GROUP BY product_id;
```

AggregatingMergeTree:

```
SELECT
    product_id,
    sumMerge(quantity),
    sumMerge(amount)
FROM exp_sales_summing_engine_aggregatingmergetree
GROUP BY product_id;
```

Measure performance:


```
SELECT
    query,
    read_rows,
    read_bytes,
    result_rows,
    memory_usage,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 5;
```

<img width="1298" height="409" alt="image" src="https://github.com/user-attachments/assets/beb96f49-a989-4085-85ee-045437199997" />
