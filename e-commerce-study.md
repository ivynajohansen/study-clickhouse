# E-Commerce Study Project

In this study project, I'm building the analytics backend for an e-commerce platform.
The platform generates massive event logs:
- Page views
- Product clicks
- Add-to-cart
- Purchases
- Search queries
- Seller performance
- Delivery status

The system must answer questions like:
- “Top selling products in the last 5 minutes”
- “Revenue by category today”
- “Conversion rate per campaign”
- “Live visitor dashboard”

## Table Creations

```
CREATE TABLE events
(
    event_time DateTime,
    event_date Date MATERIALIZED toDate(event_time),

    user_id UInt64,
    session_id String,

    event_type LowCardinality(String),

    product_id UInt64,
    category_id UInt32,

    price Float32,

    country LowCardinality(String),
    device LowCardinality(String),

    traffic_source LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_time, user_id)
SETTINGS index_granularity = 8192;
```

```
CREATE TABLE orders
(
    order_id UInt64,
    order_time DateTime,
    order_date Date MATERIALIZED toDate(order_time),

    user_id UInt64,
    product_id UInt64,
    seller_id UInt64,

    quantity UInt16,
    price Float32,

    total_amount Float32 MATERIALIZED quantity * price,

    payment_method LowCardinality(String),
    order_status LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(order_date)
ORDER BY (order_time, order_id);
```

```
CREATE TABLE products
(
    product_id UInt64,
    name String,
    category_id UInt32,
    brand String,

    price Float32,

    updated_at DateTime
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY product_id;
```

> ReplacingMergeTree engine was chosen because product data may change over time

```
CREATE TABLE categories
(
    category_id UInt32,
    category_name String
)
ENGINE = TinyLog;
```

> TinyLog engine was chosen because it's very small, rarely updated and rarely queried

```
CREATE TABLE sellers
(
    seller_id UInt64,
    seller_name String,
    country String,
    joined_date Date
)
ENGINE = MergeTree
ORDER BY seller_id;
```

```
CREATE TABLE daily_revenue
(
    revenue_date Date,
    product_id UInt64,
    seller_id UInt64,

    total_orders UInt64,
    total_quantity UInt64,
    revenue Float64
)
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(revenue_date)
ORDER BY (revenue_date, product_id, seller_id);
```

```
CREATE MATERIALIZED VIEW product_clicks_mv
TO daily_revenue
AS
SELECT
    toDate(event_time) AS revenue_date,
    product_id,
    0 AS seller_id,

    count() AS total_orders,
    0 AS total_quantity,
    0 AS revenue
FROM events
WHERE event_type = 'click'
GROUP BY
    revenue_date,
    product_id;
```

## Inserting Data

```
INSERT INTO categories VALUES
(1,'Electronics'),
(2,'Fashion'),
(3,'Home'),
(4,'Sports');
```

```
INSERT INTO sellers VALUES
(101,'TechNoblade','Indonesia','2024-01-01'),
(102,'FashionHub','Indonesia','2024-03-15'),
(103,'HomeStyle','Singapore','2023-09-10');
```

```
INSERT INTO products VALUES
(1001,'Wireless Mouse',1,'Logitech',25,'2026-03-01 10:00:00'),
(1002,'Gaming Keyboard',1,'Razer',120,'2026-03-01 10:00:00'),
(2001,'Running Shoes',4,'Nike',80,'2026-03-01 10:00:00'),
(3001,'Coffee Maker',3,'Philips',150,'2026-03-01 10:00:00');
```

```
INSERT INTO events
(event_time,user_id,session_id,event_type,product_id,category_id,price,country,device,traffic_source)
VALUES
('2026-03-12 10:00:01',1,'sess1','view',1001,1,25,'ID','mobile','ads'),
('2026-03-12 10:01:10',1,'sess1','click',1001,1,25,'ID','mobile','ads'),
('2026-03-12 10:03:45',2,'sess2','view',2001,4,80,'ID','desktop','organic');
```

```
INSERT INTO orders
(order_id,order_time,user_id,product_id,seller_id,quantity,price,payment_method,order_status)
VALUES
(5001,'2026-03-12 10:05:00',1,1001,101,1,25,'card','paid'),
(5002,'2026-03-12 10:07:00',2,2001,103,2,80,'ewallet','paid');
```

## Running Analytical Queries

Top-clicked products

```
SELECT
    product_id,
    count() AS clicks
FROM events
WHERE event_type='click'
GROUP BY product_id
ORDER BY clicks DESC
```

<img width="1290" height="292" alt="image" src="https://github.com/user-attachments/assets/d925f1bf-7a05-474b-8536-f6543099ac26" />

Revenue by seller

```
SELECT
    seller_id,
    sum(total_amount) AS revenue
FROM orders
GROUP BY seller_id
ORDER BY revenue DESC;
```

<img width="1283" height="321" alt="image" src="https://github.com/user-attachments/assets/a3e7a1e9-0e93-44a6-94ab-a9e3290af74a" />

Conversion Rate

```
SELECT
    product_id,
    countIf(event_type='view') AS views,
    countIf(event_type='click') AS clicks
FROM events
GROUP BY product_id;
```

## Data Modeling and Performance Optimization

Inspecting partitions:

```
SELECT
    table,
    partition,
    rows,
    bytes_on_disk
FROM system.parts
WHERE table = 'events'
AND active = 1;
```

<img width="1289" height="61" alt="image" src="https://github.com/user-attachments/assets/99b160e2-f0ef-436e-b755-61be16703289" />

Inspecting how a query is going to be executed using `EXPLAIN`:

```
EXPLAIN
SELECT *
FROM events
WHERE event_time >= now() - INTERVAL 1 HOUR;
```

<img width="1288" height="181" alt="image" src="https://github.com/user-attachments/assets/11ccfb96-6ee6-47e1-99f2-3d8d413ac7e9" />

```
SELECT *
FROM events
PREWHERE event_type = 'click'
WHERE event_time >= now() - INTERVAL 1 HOUR;
```

> PREWHERE = an optimization of WHERE that filters rows earlier while reading data from disk, reducing the amount of data loaded.

<img width="1294" height="299" alt="image" src="https://github.com/user-attachments/assets/5cdd4a7b-a3bb-4540-859e-3f0c89636d15" />

Primary key only helps with the ORDER BY columns, but we may frequently filter by event_type. So create a data skipping index:

```
ALTER TABLE events
ADD INDEX idx_event_type event_type TYPE set(100) GRANULARITY 4;

ALTER TABLE events MATERIALIZE INDEX idx_event_type;

SELECT *
FROM events
WHERE event_type='click';
```

<img width="1291" height="528" alt="image" src="https://github.com/user-attachments/assets/f152a5b9-5575-4bb2-9608-e2efce712ef0" />

Test compression codecs:

```
ALTER TABLE events
MODIFY COLUMN price Float32 CODEC(ZSTD);
```

Build materialized view:

```
CREATE MATERIALIZED VIEW product_views_mv
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, product_id)
AS
SELECT
    toDate(event_time) AS event_date,
    product_id,
    count() AS views
FROM events
WHERE event_type = 'view'
GROUP BY
    event_date,
    product_id;
```

## Monitor Internal Behavior:

```
SELECT *
FROM system.merges;
```

```
SELECT *
FROM system.mutations;
```

Recent queries:

```
SELECT
    query,
    query_duration_ms
FROM system.query_log
ORDER BY event_time DESC
LIMIT 10;
```
