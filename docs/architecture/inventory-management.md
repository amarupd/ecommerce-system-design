# Inventory Management Architecture

## Overview

Inventory Management is one of the most critical and challenging domains in ecommerce. Revenue, customer trust, fulfillment operations, and product availability all depend on inventory accuracy.

Unlike product catalog data, inventory is highly transactional and changes continuously due to:

* Customer purchases
* Order cancellations
* Returns
* Refunds
* Warehouse adjustments
* Stock replenishments

The inventory architecture must ensure:

* Accurate stock counts
* Oversell prevention
* High concurrency support
* Fast validation
* Auditability
* Operational visibility

This document explains the architecture, reservation strategy, consistency model, concurrency controls, and production lessons involved in managing inventory at scale.

---

# Business Objectives

The inventory system must support:

### Customer Goals

* Accurate product availability
* Reliable checkout experience
* Prevent order cancellations caused by stock issues

### Operations Goals

* Warehouse visibility
* Stock adjustment workflows
* Inventory reconciliation
* Replenishment planning

### Engineering Goals

* Consistency
* Scalability
* Reliability
* Traceability
* Performance

---

# Inventory Domain Model

Inventory is intentionally separated from product catalog data.

```text
Product
   ↓
Variant
   ↓
Inventory
```

Supporting entities:

```text
Inventory Transaction
Inventory Reservation
Warehouse
Order
Return
Stock Adjustment
```

---

# Why Inventory Is Separate

Avoid storing inventory directly on products.

Incorrect:

```text
Product
 └── stock_quantity
```

Recommended:

```text
Product
   ↓
Variant
   ↓
Inventory
```

Benefits:

* Better scalability
* Easier auditing
* Multi-warehouse support
* Cleaner separation of concerns

---

# Inventory Data Model

## Inventory Table

```sql
id
variant_id
available_quantity
reserved_quantity
warehouse_id
updated_at
```

---

## Inventory Transaction Table

```sql
id
variant_id
transaction_type
quantity
reference_id
reference_type
created_at
```

---

## Inventory Reservation Table

```sql
id
variant_id
order_id
quantity
status
expires_at
created_at
```

---

# Inventory States

Inventory consists of multiple quantities.

---

## Available Quantity

Inventory available for purchase.

Example:

```text
100 Units
```

---

## Reserved Quantity

Inventory temporarily held.

Example:

```text
20 Units
```

---

## Total Inventory

Calculation:

```text
Available + Reserved
```

Example:

```text
Available = 80

Reserved = 20

Total = 100
```

---

# Inventory Validation Flow

Inventory validation occurs during checkout.

---

## Validation Sequence

```text
Checkout Request
       ↓
Locate Inventory
       ↓
Verify Availability
       ↓
Reserve Inventory
       ↓
Continue Checkout
```

---

# Stock Reservation System

Reservation is essential for preventing overselling.

---

## Why Reservation Exists

Scenario:

```text
Stock = 1

Customer A Checkout

Customer B Checkout
```

Without reservation:

Both may purchase.

---

## Reservation Flow

```text
Validate Inventory
        ↓
Create Reservation
        ↓
Reduce Available Stock
        ↓
Proceed To Payment
```

---

## Reservation Example

Before:

```text
Available = 10
Reserved = 0
```

Reserve 3:

```text
Available = 7
Reserved = 3
```

---

# Reservation Expiration

Not all checkouts succeed.

Inventory cannot remain locked indefinitely.

---

## Expiration Strategy

Example:

```text
Reservation TTL

15 Minutes
```

After expiration:

```text
Release Reservation
Restore Inventory
```

---

# Inventory Confirmation

When payment succeeds:

```text
Reservation
       ↓
Confirmed
       ↓
Inventory Deducted
```

---

## Example

Before:

```text
Available = 7

Reserved = 3
```

After purchase:

```text
Available = 7

Reserved = 0

Sold = 3
```

---

# Inventory Release Flow

When payment fails:

```text
Reservation
       ↓
Cancelled
       ↓
Inventory Restored
```

---

## Example

Before:

```text
Available = 7

Reserved = 3
```

After release:

```text
Available = 10

Reserved = 0
```

---

# Inventory Transactions

Every inventory movement must be recorded.

---

## Transaction Types

Examples:

```text
Purchase

Reservation

Release

Return

Adjustment

Restock
```

---

## Benefits

Provides:

* Audit trail
* Operational visibility
* Reconciliation support

---

# Inventory Audit Trail

Every stock movement is traceable.

Example:

```text
+100 Stock Received

-3 Customer Purchase

+1 Customer Return

-2 Manual Adjustment
```

---

# Overselling Prevention

Overselling damages customer trust.

The architecture prevents overselling through:

---

## Real-Time Validation

Inventory checked immediately before reservation.

---

## Reservation Layer

Stock reserved before payment.

---

## Database Transactions

Atomic updates prevent inconsistencies.

---

## Row-Level Locking

Protects against concurrent updates.

---

# Concurrency Challenges

Inventory is highly contested during promotions.

---

## Example

```text
Stock = 5

100 Customers
Attempt Purchase
```

---

## Risk

Without protection:

```text
Sold = 100

Available = 5
```

Impossible state.

---

# Locking Strategy

Inventory updates use row-level locking.

---

## Flow

```text
Acquire Lock
      ↓
Read Inventory
      ↓
Update Inventory
      ↓
Release Lock
```

---

## Benefits

Prevents:

* Double selling
* Lost updates
* Race conditions

---

# Inventory Synchronization

Inventory changes from multiple sources.

Examples:

```text
Checkout

Admin Update

Warehouse Update

Return Processing
```

All systems must remain synchronized.

---

# Inventory Consistency Model

The platform favors strong consistency.

Reason:

Inventory inaccuracies directly affect revenue.

---

## Principle

Inventory must always be correct.

Even if:

```text
Latency Slightly Increases
```

Correctness is prioritized.

---

# Multi-Warehouse Architecture

Large ecommerce systems often support multiple warehouses.

---

## Structure

```text
Warehouse A

Warehouse B

Warehouse C
```

Each maintains independent inventory.

---

## Inventory Selection

During checkout:

```text
Customer Location
       ↓
Warehouse Selection
       ↓
Inventory Allocation
```

---

# Warehouse Allocation Strategies

Examples:

---

## Nearest Warehouse

Benefits:

* Faster delivery
* Lower shipping cost

---

## Highest Inventory

Benefits:

* Better stock utilization

---

## Hybrid Strategy

Considers:

* Distance
* Availability
* Cost

---

# Returns and Inventory

Returns impact inventory.

---

## Return Flow

```text
Customer Return
       ↓
Inspection
       ↓
Inventory Update
```

---

## Return Outcomes

Possible outcomes:

```text
Restock

Discard

Refurbish
```

---

# Inventory Reconciliation

Periodic reconciliation ensures accuracy.

---

## Reconciliation Sources

Compare:

```text
System Inventory

Warehouse Inventory
```

---

## Goal

Detect:

```text
Missing Stock

Duplicate Stock

Incorrect Adjustments
```

---

# Redis Inventory Optimization

Inventory remains source-of-truth in MySQL.

Redis used for:

```text
Availability Checks

Popular Products

Inventory Metadata
```

---

## Benefits

* Faster product pages
* Reduced database load

---

## Important Rule

Never rely solely on cache for final inventory decisions.

Always validate against database.

---

# Failure Handling

Inventory systems must tolerate failures.

---

## Scenario 1

Payment Failure

Resolution:

Release reservation.

---

## Scenario 2

Application Crash

Resolution:

Recover from reservation records.

---

## Scenario 3

Expired Reservation

Resolution:

Scheduled cleanup workers.

---

## Scenario 4

Duplicate Purchase Requests

Resolution:

Idempotency protection.

---

# Monitoring Metrics

## Business Metrics

Track:

* Out-of-stock rate
* Inventory turnover
* Restock frequency

---

## Operational Metrics

Track:

* Reservation count
* Expired reservations
* Inventory adjustments

---

## Technical Metrics

Track:

* Inventory validation latency
* Lock wait times
* Reservation failures
* Database contention

---

# Production Incidents and Lessons

## Incident 1

Inventory Reserved But Never Released

Impact:

Artificial stock shortages.

Resolution:

Automated reservation cleanup jobs.

---

## Incident 2

Flash Sale Oversell

Cause:

Concurrent checkout race condition.

Resolution:

Row-level locking implementation.

---

## Incident 3

Manual Warehouse Adjustment Error

Impact:

Incorrect stock visibility.

Resolution:

Mandatory audit logging.

---

## Incident 4

Cache Displayed Incorrect Inventory

Impact:

Customer frustration.

Resolution:

Short inventory cache TTLs.

---

# Scalability Strategy

Inventory scales through:

### Horizontal Application Scaling

Multiple inventory service instances.

---

### Optimized Database Design

Indexes on:

```sql
variant_id

warehouse_id

status
```

---

### Reservation Architecture

Protects consistency during traffic spikes.

---

### Queue-Based Processing

Handles non-critical updates asynchronously.

---

# Reliability Principles

The inventory system follows:

* Validate before reserve
* Reserve before payment
* Release on failure
* Audit every movement
* Prefer consistency over speed
* Design for concurrency
* Recover automatically

---

# Engineering Outcomes

The Inventory Management Architecture provides:

* Accurate stock visibility
* Oversell prevention
* High-concurrency support
* Multi-warehouse readiness
* Complete auditability
* Operational transparency
* Scalable inventory processing
* Enterprise-grade consistency

This architecture enables ecommerce platforms to maintain reliable inventory operations even during large-scale sales events, rapid growth, and complex fulfillment workflows.
