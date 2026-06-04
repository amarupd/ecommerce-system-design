# Cart Architecture

## Overview

The shopping cart is one of the most critical components in an ecommerce platform. It acts as the bridge between product discovery and checkout conversion.

A well-designed cart system must provide:

* Fast user interactions
* Inventory awareness
* Cross-device synchronization
* Fault tolerance
* High scalability
* Seamless checkout integration

In production ecommerce systems, cart traffic is often significantly higher than order traffic. A customer may update their cart dozens of times before completing a purchase.

This document explains how a production-grade cart system is architected to handle large user volumes, maintain inventory accuracy, and deliver low-latency customer experiences.

---

# Business Requirements

The cart system must support:

### Customer Requirements

* Add products quickly
* Update quantities instantly
* Remove items easily
* Preserve carts across sessions
* Continue shopping on multiple devices

### Business Requirements

* Reduce cart abandonment
* Improve checkout conversion
* Support promotions and coupons
* Enable cart recovery campaigns

### Engineering Requirements

* Low latency
* High concurrency
* Inventory validation
* Horizontal scalability
* Cache efficiency

---

# Cart Domain Model

The cart architecture consists of:

```text
User
  ↓
Cart
  ↓
Cart Item
  ↓
Variant
  ↓
Inventory
```

Supporting entities:

```text
Guest Session
Coupon
Address
Inventory Reservation
Checkout Session
```

---

# Cart Types

The platform supports two cart models.

## Guest Cart

Used when the customer is not authenticated.

Stored using:

* Browser storage
* Temporary session identifiers
* Redis cache

Characteristics:

* No account required
* Short-lived lifecycle
* Can be merged after login

---

## Authenticated Cart

Used for logged-in customers.

Stored in:

* MySQL
* Redis

Benefits:

* Persistent storage
* Cross-device access
* Cart recovery support

---

# Cart Data Model

## Cart Table

```sql
id
user_id
status
created_at
updated_at
```

---

## Cart Item Table

```sql
id
cart_id
variant_id
quantity
created_at
updated_at
```

---

# Add To Cart Flow

Adding products to the cart follows a validation-first approach.

Flow:

```text
Customer Action
      ↓
Validate Product
      ↓
Validate Variant
      ↓
Validate Inventory
      ↓
Create/Update Cart
      ↓
Update Cache
      ↓
Return Response
```

---

## Validation Rules

Before adding an item:

### Product Validation

Verify:

* Product exists
* Product is active
* Product is purchasable

---

### Variant Validation

Verify:

* Variant exists
* Variant is active
* SKU is available

---

### Inventory Validation

Verify:

* Inventory available
* Quantity permitted
* Stock not reserved

---

# Cart Update Flow

When quantity changes:

```text
Update Request
      ↓
Inventory Check
      ↓
Quantity Validation
      ↓
Cart Update
      ↓
Cache Update
```

---

# Cart Removal Flow

Removing items requires:

```text
Customer Request
      ↓
Locate Cart Item
      ↓
Delete Cart Item
      ↓
Update Cache
      ↓
Return Updated Cart
```

---

# Redis Cart Strategy

Cart operations are extremely read-heavy.

To reduce database pressure, carts are cached in Redis.

---

## Cache Structure

Example:

```text
cart:user:123
```

Stored value:

```json
{
  "items": [
    {
      "variant_id": 101,
      "quantity": 2
    }
  ]
}
```

---

## Benefits

Redis provides:

* Sub-millisecond access
* Reduced MySQL load
* Faster cart rendering
* Better scalability

---

# Cart Synchronization

Customers frequently switch devices.

Example:

```text
Mobile
   ↓
Tablet
   ↓
Laptop
```

All devices must display the same cart state.

---

## Synchronization Flow

```text
User Login
      ↓
Fetch Persistent Cart
      ↓
Merge Local Cart
      ↓
Resolve Conflicts
      ↓
Update Cache
```

---

# Guest Cart Merge Strategy

A common ecommerce scenario:

```text
Guest User
      ↓
Adds Products
      ↓
Creates Account
      ↓
Logs In
```

Guest cart must be merged safely.

---

## Merge Rules

### New Product

Add directly.

---

### Existing Product

Combine quantities.

---

### Inventory Limit

Respect available stock.

---

# Multi-Device Cart Management

Modern customers use multiple devices.

Example:

```text
Phone
Laptop
Tablet
```

All cart updates must remain synchronized.

---

## Architecture

```text
Device
   ↓
API
   ↓
Redis
   ↓
MySQL
```

Redis acts as the source of fast-access state.

MySQL remains the source of truth.

---

# Inventory Validation Strategy

Inventory changes constantly.

Cart quantities cannot be trusted indefinitely.

---

## Validation Points

Inventory is checked:

### Add To Cart

Verify stock exists.

---

### Quantity Update

Verify requested quantity.

---

### Checkout

Perform final validation.

---

# Inventory Drift Protection

Scenario:

```text
Inventory = 5

User A adds 5

User B adds 5
```

Only one checkout should succeed.

---

Solution:

Inventory validation during checkout.

Inventory reservation before payment.

---

# Cart Recovery Architecture

Cart recovery improves revenue.

---

## Abandoned Cart Definition

Example:

```text
Customer adds items

No checkout activity

30+ minutes inactivity
```

Cart becomes recoverable.

---

## Recovery Workflow

```text
Cart Abandoned
      ↓
Recovery Job
      ↓
Notification Trigger
      ↓
Customer Returns
      ↓
Cart Restored
```

---

# Coupon Integration

Cart must support coupon validation.

Validation includes:

* Expiry checks
* Usage limits
* Product eligibility
* Category eligibility

---

Flow:

```text
Apply Coupon
      ↓
Validate Rules
      ↓
Calculate Discount
      ↓
Update Cart Totals
```

---

# Cart Pricing Architecture

Cart pricing is recalculated continuously.

Components:

```text
Subtotal
Discount
Shipping
Tax
Total
```

Never trust client-side calculations.

All calculations occur on the server.

---

# Cart Cache Invalidation

One of the most important engineering concerns.

---

## Invalidation Events

Cache refresh triggered by:

* Cart updates
* Product changes
* Inventory updates
* Coupon updates

---

Example:

```text
Product Updated
      ↓
Invalidate Cart Cache
      ↓
Refresh Pricing
```

---

# Cart Scalability Challenges

## Challenge 1

Massive Concurrent Activity

Problem:

Thousands of customers updating carts simultaneously.

Solution:

* Redis caching
* Stateless services
* Horizontal scaling

---

## Challenge 2

Inventory Accuracy

Problem:

Inventory changes after items enter carts.

Solution:

* Final checkout validation
* Inventory reservation system

---

## Challenge 3

Cross-Device Consistency

Problem:

Different cart states across devices.

Solution:

* Centralized persistence
* Redis synchronization

---

# Monitoring Metrics

Critical metrics include:

### Business Metrics

* Add-to-cart rate
* Cart abandonment rate
* Checkout initiation rate
* Conversion rate

---

### Technical Metrics

* Cart API latency
* Redis hit ratio
* Cart merge failures
* Inventory validation failures

---

# Production Incidents and Lessons

## Incident 1

Inventory Changed After Cart Addition

Impact:

Customer saw available product but checkout failed.

Resolution:

Added final inventory validation during checkout.

---

## Incident 2

Redis Cache Expiration

Impact:

Cart temporarily appeared empty.

Resolution:

Implemented cache rebuild strategy from MySQL.

---

## Incident 3

Guest Cart Merge Conflict

Impact:

Duplicate quantities after login.

Resolution:

Introduced deterministic merge rules.

---

# Performance Optimizations

Key optimizations include:

### Redis Caching

Fast cart retrieval.

---

### Batched Queries

Reduce database round trips.

---

### Lazy Product Loading

Load only required product metadata.

---

### Efficient Indexing

Indexes on:

```sql
cart_id
user_id
variant_id
```

---

# Engineering Outcomes

The cart architecture delivers:

* Fast customer interactions
* Inventory-aware operations
* Multi-device consistency
* High scalability
* Reliable checkout preparation
* Efficient caching
* Operational simplicity

The resulting design supports both growing ecommerce brands and high-traffic enterprise commerce environments while maintaining excellent user experience and conversion performance.
