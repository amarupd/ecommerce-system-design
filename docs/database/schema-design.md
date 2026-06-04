# Database Schema Design

## Overview

The database is the system of record for the ecommerce platform. Every critical business operation—including customer management, product catalog management, inventory tracking, cart persistence, order processing, payments, coupons, and fulfillment—ultimately relies on the database layer.

The schema was designed around the following goals:

* Data integrity
* Transactional consistency
* Scalability
* Query performance
* Maintainability
* Auditability

The platform uses **MySQL** as the primary relational database because ecommerce systems require strong consistency, transactional guarantees, and well-defined relationships between entities.

---

# Design Principles

The schema follows several architectural principles.

### Normalize Core Business Data

Avoid duplication where possible.

Benefits:

* Better consistency
* Easier maintenance
* Reduced storage overhead

---

### Denormalize Selectively

For high-volume read paths.

Examples:

```text id="f4h2xa"
Order Snapshots

Address Snapshots

Product Metadata
```

---

### Strong Referential Integrity

Use foreign keys for critical relationships.

Benefits:

* Prevent orphan records
* Maintain data quality
* Improve operational reliability

---

### Auditability

Important actions should be traceable.

Examples:

```text id="m7p8kv"
Inventory Updates

Payments

Refunds

Order Status Changes
```

---

# High-Level Domain Model

```text id="r9w4nt"
Users
  ↓
Addresses

Products
  ↓
Variants
  ↓
Inventory

Users
  ↓
Carts
  ↓
Cart Items

Users
  ↓
Orders
  ↓
Order Items
  ↓
Payments

Coupons
```

---

# Customer Domain

The customer domain stores identity and account information.

---

## Users Table

```sql
users
-----
id
name
email
phone
password_hash
status
email_verified
created_at
updated_at
```

---

## Addresses Table

```sql
addresses
---------
id
user_id
full_name
phone_number
address_line_1
address_line_2
city
state
country
postal_code
is_default
created_at
updated_at
```

---

## Relationship

```text id="t5y7mx"
User
  ↓
Many Addresses
```

---

# Product Catalog Domain

The catalog architecture supports categories, products, variants, and attributes.

---

## Categories Table

```sql
categories
----------
id
parent_id
name
slug
description
status
created_at
updated_at
```

---

## Category Hierarchy

Example:

```text id="g2w8pl"
Sports
 └── Cricket
      └── Jerseys
```

Uses:

```text id="a8m4qv"
parent_id
```

For hierarchical relationships.

---

## Products Table

```sql
products
--------
id
category_id
name
slug
description
brand
status
is_featured
created_at
updated_at
```

---

## Variants Table

Purchasable entities.

```sql
variants
--------
id
product_id
sku
price
compare_price
status
created_at
updated_at
```

---

## Relationship

```text id="k3n7wr"
Product
   ↓
Many Variants
```

---

# Attribute Architecture

Attributes support flexible product configurations.

---

## Attributes Table

```sql
attributes
----------
id
name
created_at
```

Examples:

```text id="p6m2xe"
Size

Color

Material
```

---

## Attribute Values Table

```sql
attribute_values
----------------
id
attribute_id
value
created_at
```

---

## Variant Attributes Table

```sql
variant_attributes
------------------
id
variant_id
attribute_value_id
```

---

# Product Images

Products support multiple images.

---

## Product Images Table

```sql
product_images
--------------
id
product_id
image_url
is_primary
sort_order
created_at
```

---

# Inventory Domain

Inventory is separated from products.

---

## Inventory Table

```sql
inventory
---------
id
variant_id
available_quantity
reserved_quantity
warehouse_id
updated_at
```

---

## Inventory Transactions Table

```sql
inventory_transactions
----------------------
id
variant_id
transaction_type
quantity
reference_id
reference_type
created_at
```

---

## Relationship

```text id="n9w5jb"
Variant
   ↓
Inventory
```

---

# Cart Domain

Cart data persists independently of checkout.

---

## Carts Table

```sql
carts
-----
id
user_id
status
created_at
updated_at
```

---

## Cart Items Table

```sql
cart_items
----------
id
cart_id
variant_id
quantity
created_at
updated_at
```

---

## Relationship

```text id="r4p8yt"
Cart
  ↓
Many Cart Items
```

---

# Coupon Domain

Coupons support promotions and discounts.

---

## Coupons Table

```sql
coupons
-------
id
code
type
value
status
usage_limit
minimum_order_value
starts_at
expires_at
created_at
```

---

## Coupon Usage Table

```sql
coupon_usage
------------
id
coupon_id
user_id
order_id
discount_amount
created_at
```

---

# Order Domain

The order domain represents completed purchases.

---

## Orders Table

```sql
orders
------
id
order_number
user_id
status
subtotal
discount_amount
shipping_amount
tax_amount
grand_total
payment_status
created_at
updated_at
```

---

## Order Items Table

```sql
order_items
-----------
id
order_id
variant_id
sku
quantity
price
created_at
```

---

## Relationship

```text id="v7m1pk"
Order
  ↓
Many Order Items
```

---

# Order Address Snapshot

Order addresses are immutable.

---

## Order Addresses Table

```sql
order_addresses
---------------
id
order_id
full_name
phone_number
address_line_1
address_line_2
city
state
country
postal_code
```

---

# Payment Domain

Payments are stored independently.

---

## Payments Table

```sql
payments
--------
id
order_id
gateway
transaction_id
amount
status
created_at
updated_at
```

---

## Payment Statuses

Examples:

```text id="x5n8ql"
Pending

Authorized

Captured

Failed

Refunded
```

---

# Shipment Domain

Supports order fulfillment.

---

## Shipments Table

```sql
shipments
---------
id
order_id
carrier
tracking_number
status
shipped_at
delivered_at
created_at
```

---

# Returns Domain

Supports post-purchase workflows.

---

## Returns Table

```sql
returns
-------
id
order_id
status
reason
created_at
updated_at
```

---

## Refunds Table

```sql
refunds
-------
id
return_id
payment_id
amount
status
created_at
```

---

# Audit Domain

Tracks critical operational events.

---

## Audit Logs Table

```sql
audit_logs
----------
id
user_id
entity_type
entity_id
action
metadata
created_at
```

---

# Relationship Overview

```text id="c6m4wa"
User
 ├── Addresses
 ├── Carts
 ├── Orders
 └── Coupon Usage

Category
 └── Products

Product
 ├── Variants
 └── Images

Variant
 ├── Inventory
 ├── Cart Items
 └── Order Items

Order
 ├── Order Items
 ├── Payment
 ├── Shipment
 ├── Return
 └── Address Snapshot
```

---

# Normalization Decisions

The schema primarily follows Third Normal Form (3NF).

---

## Benefits

* Reduced duplication
* Improved consistency
* Easier updates

---

## Exceptions

Selective denormalization used for:

```text id="w8k2je"
Order Addresses

Historical Product Data

Reporting Tables
```

---

# Transaction Boundaries

Transactions are critical for:

### Checkout

```text id="q3v7pr"
Reserve Inventory

Create Order

Create Payment
```

---

### Refunds

```text id="t1m6yb"
Update Refund

Update Payment

Update Order
```

---

# Scalability Considerations

The schema was designed to support:

### Millions of Customers

Through:

* Indexed relationships
* Efficient lookups

---

### Large Catalogs

Through:

* Variant separation
* Attribute modeling

---

### High Order Volume

Through:

* Optimized order tables
* Snapshot strategy

---

# Common Schema Tradeoffs

## Product vs Variant Pricing

Decision:

Store price on variant.

Reason:

Customers purchase variants.

---

## Address Snapshotting

Decision:

Copy address into order.

Reason:

Historical accuracy.

---

## Inventory Separation

Decision:

Dedicated inventory tables.

Reason:

Scalability and auditability.

---

# Production Lessons Learned

## Lesson 1

Never store inventory directly on products.

---

## Lesson 2

Always model purchasable entities as variants.

---

## Lesson 3

Order data becomes historical data.

Optimize for preservation, not modification.

---

## Lesson 4

Snapshot important checkout information.

---

## Lesson 5

Design auditability from the beginning.

Adding it later is expensive.

---

# Engineering Outcomes

The database schema provides:

* Strong consistency
* Relational integrity
* Scalable product modeling
* Reliable order processing
* Inventory traceability
* Efficient querying
* Historical accuracy
* Enterprise-grade maintainability

This schema forms the foundation of the ecommerce platform and supports the operational, transactional, and analytical needs of a modern production-grade commerce system.
