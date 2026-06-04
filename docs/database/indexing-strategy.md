# Indexing Strategy

## Overview

Indexes are one of the most important performance optimization techniques in relational databases. In ecommerce systems, database workloads are dominated by:

* Product browsing
* Search and filtering
* Cart retrieval
* Order lookups
* Inventory validation
* Coupon validation

Without proper indexing, query performance degrades rapidly as data volume grows.

This document explains the indexing architecture, optimization strategies, tradeoffs, and production lessons learned while operating high-volume ecommerce workloads.

---

# Objectives

The indexing strategy was designed to:

* Minimize query latency
* Support catalog growth
* Optimize checkout workflows
* Improve search performance
* Reduce database load
* Scale efficiently

---

# Core Principles

The platform follows several indexing principles.

### Index Frequently Queried Columns

Examples:

```text id="f1j4ma"
slug

sku

user_id

order_number
```

---

### Optimize For Read Traffic

Ecommerce workloads are typically:

```text id="p5x8ne"
90%+ Reads
```

Indexes primarily optimize read performance.

---

### Avoid Excessive Indexes

Too many indexes create:

* Slower writes
* Increased storage usage
* More maintenance overhead

---

### Design Around Query Patterns

Indexes are created for real application queries, not theoretical usage.

---

# Product Catalog Indexes

The catalog receives the highest read traffic.

---

## Product Slug Index

Query:

```sql id="o7u2lk"
SELECT *
FROM products
WHERE slug = ?
```

Index:

```sql id="y4n6rd"
INDEX(slug)
```

Benefits:

* Fast product page loading
* SEO-friendly lookups

---

## Category Index

Query:

```sql id="x8v1pq"
SELECT *
FROM products
WHERE category_id = ?
```

Index:

```sql id="g2m5ya"
INDEX(category_id)
```

---

## Featured Product Index

Query:

```sql id="n9r4tc"
SELECT *
FROM products
WHERE is_featured = 1
```

Index:

```sql id="j6k8mw"
INDEX(is_featured)
```

---

## Product Status Index

Query:

```sql id="d3w7pb"
SELECT *
FROM products
WHERE status = 'active'
```

Index:

```sql id="r5x2eq"
INDEX(status)
```

---

# Variant Indexes

Variants are central to inventory and checkout.

---

## Product Variant Lookup

Query:

```sql id="m8t4zn"
SELECT *
FROM variants
WHERE product_id = ?
```

Index:

```sql id="u2y6kc"
INDEX(product_id)
```

---

## SKU Lookup

Query:

```sql id="v7q1hb"
SELECT *
FROM variants
WHERE sku = ?
```

Index:

```sql id="c9m4rx"
UNIQUE(sku)
```

Benefits:

* Inventory lookups
* Fulfillment operations
* Warehouse integrations

---

# Inventory Indexes

Inventory validation occurs during checkout.

Performance is critical.

---

## Variant Inventory Lookup

Query:

```sql id="a4w9me"
SELECT *
FROM inventory
WHERE variant_id = ?
```

Index:

```sql id="k8x3tu"
INDEX(variant_id)
```

---

## Warehouse Inventory Lookup

Query:

```sql id="s1p6jq"
SELECT *
FROM inventory
WHERE warehouse_id = ?
```

Index:

```sql id="h5v2rn"
INDEX(warehouse_id)
```

---

## Composite Inventory Index

Query:

```sql id="e7n4mb"
WHERE warehouse_id = ?
AND variant_id = ?
```

Index:

```sql id="w2q8pa"
INDEX(warehouse_id, variant_id)
```

Benefits:

* Faster warehouse allocation
* Improved inventory validation

---

# Cart Indexes

Cart operations require low latency.

---

## User Cart Lookup

Query:

```sql id="p8m3yf"
SELECT *
FROM carts
WHERE user_id = ?
```

Index:

```sql id="x4v7tn"
INDEX(user_id)
```

---

## Cart Item Lookup

Query:

```sql id="q1w5rk"
SELECT *
FROM cart_items
WHERE cart_id = ?
```

Index:

```sql id="z6n2pb"
INDEX(cart_id)
```

---

## Variant Lookup In Cart

Query:

```sql id="m5x8je"
WHERE variant_id = ?
```

Index:

```sql id="g7q4tc"
INDEX(variant_id)
```

---

# Order Indexes

Order data grows rapidly.

Indexes become essential.

---

## Order Number Lookup

Query:

```sql id="v3k9xm"
SELECT *
FROM orders
WHERE order_number = ?
```

Index:

```sql id="b8p2wr"
UNIQUE(order_number)
```

---

## Customer Orders

Query:

```sql id="j4m7nt"
SELECT *
FROM orders
WHERE user_id = ?
```

Index:

```sql id="q6x1pe"
INDEX(user_id)
```

---

## Order Status

Query:

```sql id="u9n3rb"
SELECT *
FROM orders
WHERE status = ?
```

Index:

```sql id="t2v5mk"
INDEX(status)
```

---

## Order Date Queries

Query:

```sql id="n8p4xy"
WHERE created_at >= ?
```

Index:

```sql id="r1m7qa"
INDEX(created_at)
```

---

# Composite Order Indexes

Very common ecommerce query:

```sql id="k7w2vn"
WHERE user_id = ?
AND status = ?
```

Index:

```sql id="m3x8pt"
INDEX(user_id, status)
```

Benefits:

* Faster order history pages
* Better dashboard performance

---

# Coupon Indexes

Coupon validation occurs frequently.

---

## Coupon Code Lookup

Query:

```sql id="s4v9ke"
SELECT *
FROM coupons
WHERE code = ?
```

Index:

```sql id="f8n2ra"
UNIQUE(code)
```

---

## Coupon Status

Query:

```sql id="p2m6wt"
WHERE status = 'active'
```

Index:

```sql id="x5q1nb"
INDEX(status)
```

---

## Coupon Expiration

Query:

```sql id="v8r4py"
WHERE expires_at < NOW()
```

Index:

```sql id="c3m7tk"
INDEX(expires_at)
```

---

# Payment Indexes

Payment reconciliation requires fast lookups.

---

## Transaction Lookup

Query:

```sql id="a7w2qe"
WHERE transaction_id = ?
```

Index:

```sql id="n4x9pm"
UNIQUE(transaction_id)
```

---

## Order Payment Lookup

Query:

```sql id="r6m3kb"
WHERE order_id = ?
```

Index:

```sql id="t8p5nx"
INDEX(order_id)
```

---

# Shipment Indexes

Shipping operations rely heavily on tracking searches.

---

## Tracking Number Lookup

Query:

```sql id="q2v7rw"
WHERE tracking_number = ?
```

Index:

```sql id="m9n4pt"
UNIQUE(tracking_number)
```

---

# Audit Log Indexes

Audit data grows rapidly.

---

## Entity Lookup

Query:

```sql id="p7x3ma"
WHERE entity_type = ?
AND entity_id = ?
```

Index:

```sql id="w5m8rc"
INDEX(entity_type, entity_id)
```

---

# Composite Index Strategy

Composite indexes improve performance for multi-column filters.

---

## Example

Query:

```sql id="h1q7vn"
WHERE category_id = ?
AND status = 'active'
```

Index:

```sql id="r4x2pe"
INDEX(category_id, status)
```

---

## Benefits

* Faster filtering
* Lower scan cost
* Better query plans

---

# Covering Indexes

Certain queries can be served entirely from indexes.

Example:

```sql id="u8m5ky"
SELECT id, slug
FROM products
```

Benefits:

* Reduced disk reads
* Faster execution

---

# Over-Indexing Risks

Indexes are not free.

---

## Problems

Too many indexes cause:

```text id="n2p6wa"
Slower Inserts

Slower Updates

More Storage Usage
```

---

## Example

Product update:

```text id="c7v3rm"
Update Product
       ↓
Update Every Related Index
```

Write cost increases.

---

# Read vs Write Tradeoff

Ecommerce systems are read-heavy.

Therefore:

```text id="m5q8xe"
Read Optimization
```

Usually takes priority.

However:

Inventory and checkout paths require careful balance.

---

# Query Analysis Strategy

Indexes should be validated using:

```sql id="k9x4nt"
EXPLAIN
```

Benefits:

* Understand execution plans
* Detect table scans
* Optimize performance

---

# Monitoring Index Effectiveness

Track:

### Slow Queries

Identify:

```text id="y3m7rb"
Missing Indexes
```

---

### Index Usage

Monitor:

```text id="f6p2vk"
Unused Indexes
```

---

### Query Latency

Track:

```text id="x8n5qt"
Read Performance
```

---

# Production Lessons Learned

## Lesson 1

Indexes should be based on actual query patterns.

---

## Lesson 2

SKU lookups become critical as catalog size grows.

---

## Lesson 3

Composite indexes often outperform multiple single-column indexes.

---

## Lesson 4

Audit tables require dedicated indexing strategies.

---

## Lesson 5

Over-indexing creates hidden write performance problems.

---

# Common Ecommerce Index Set

Minimum recommended indexes:

```sql id="t7m3wp"
products(slug)

products(category_id)

products(status)

variants(sku)

variants(product_id)

inventory(variant_id)

orders(order_number)

orders(user_id)

orders(status)

coupons(code)

payments(transaction_id)

shipments(tracking_number)
```

---

# Engineering Outcomes

The indexing strategy provides:

* Fast product retrieval
* Efficient filtering
* Low-latency checkout
* Reliable inventory validation
* Faster order searches
* Improved reporting performance
* Reduced database load
* Scalable query execution

This indexing architecture enables the ecommerce platform to maintain predictable performance as catalog size, customer volume, and order traffic continue to grow.
