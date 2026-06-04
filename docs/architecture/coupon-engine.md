# Coupon Engine Architecture

## Overview

The Coupon Engine is responsible for managing promotions, discount campaigns, customer incentives, and pricing rules across the ecommerce platform.

While discounts appear simple from a customer perspective, the underlying architecture must handle:

* Complex eligibility rules
* Real-time validation
* Abuse prevention
* High concurrency
* Pricing consistency
* Usage tracking
* Campaign management

A poorly designed coupon system can directly impact revenue through incorrect discounts, fraud, duplicate redemptions, and checkout failures.

This document explains the architecture, validation workflows, pricing calculations, scaling strategies, and operational lessons learned from building a production-grade coupon engine.

---

# Business Objectives

The promotion system must support:

### Marketing Goals

* Customer acquisition
* Customer retention
* Seasonal campaigns
* Product promotions
* Category promotions

### Customer Goals

* Easy coupon redemption
* Transparent discount calculation
* Fast validation

### Engineering Goals

* Accurate pricing
* Abuse prevention
* Scalability
* Reliability
* Auditability

---

# Promotion Architecture

The coupon engine operates as an independent domain.

```text
Customer
    ↓
Checkout
    ↓
Coupon Engine
    ↓
Validation Layer
    ↓
Pricing Engine
    ↓
Order Service
```

---

# Promotion Types

The platform supports multiple discount models.

---

## Fixed Amount Discount

Example:

```text
₹500 OFF
```

Calculation:

```text
Subtotal = ₹5000

Discount = ₹500

Final = ₹4500
```

---

## Percentage Discount

Example:

```text
10% OFF
```

Calculation:

```text
Subtotal = ₹5000

Discount = ₹500

Final = ₹4500
```

---

## Category Discount

Example:

```text
15% OFF Jerseys
```

Applied only to eligible products.

---

## Product-Specific Discount

Example:

```text
20% OFF Product XYZ
```

---

## Free Shipping

Example:

```text
Free Delivery Above ₹999
```

---

## First Order Discount

Example:

```text
WELCOME10
```

Available only for first-time customers.

---

# Coupon Data Model

## Coupons Table

```sql
id
code
type
value
status
starts_at
expires_at
usage_limit
minimum_order_value
maximum_discount
created_at
updated_at
```

---

## Coupon Usage Table

```sql
id
coupon_id
user_id
order_id
discount_amount
created_at
```

---

# Coupon Validation Flow

Coupons must be validated before any discount is applied.

---

## Validation Sequence

```text
Coupon Applied
       ↓
Coupon Exists
       ↓
Coupon Active
       ↓
Date Validation
       ↓
Usage Validation
       ↓
Eligibility Validation
       ↓
Pricing Calculation
       ↓
Discount Applied
```

---

# Basic Validation Rules

---

## Coupon Exists

Verify:

```text
Coupon Found
```

---

## Coupon Active

Verify:

```text
status = active
```

---

## Date Validation

Verify:

```text
Current Time
Within
Campaign Window
```

---

## Minimum Order Value

Example:

```text
Minimum Order = ₹2000

Cart = ₹1500

Validation Fails
```

---

## Maximum Discount

Example:

```text
20% OFF

Maximum Discount = ₹1000
```

Even if calculation exceeds ₹1000, cap is enforced.

---

# Customer Eligibility Rules

Certain promotions target specific customers.

---

## First Order Coupons

Validation:

```text
Customer Orders = 0
```

Only then allow redemption.

---

## User-Specific Campaigns

Example:

```text
VIP Users
```

Only eligible customers receive discounts.

---

## Segment-Based Promotions

Examples:

```text
Premium Customers

Loyalty Members

Referral Customers
```

---

# Product Eligibility Rules

Promotions often apply selectively.

---

## Product Restrictions

Coupon may apply only to:

```text
Product A
Product B
Product C
```

---

## Category Restrictions

Coupon applies only to:

```text
Jerseys

Accessories
```

---

## Brand Restrictions

Coupon applies only to:

```text
Brand X
```

---

# Discount Calculation Engine

The pricing engine performs calculations after validation.

---

## Calculation Flow

```text
Validate Coupon
        ↓
Identify Eligible Products
        ↓
Calculate Discount
        ↓
Apply Limits
        ↓
Update Cart Total
```

---

## Example

Cart:

```text
Jersey = ₹2000

Cap = ₹1000
```

Coupon:

```text
20% OFF
```

Discount:

```text
₹400
```

Final:

```text
₹1600
```

---

# Coupon Stacking Strategy

Stacking determines whether multiple coupons can be combined.

---

## Strategy 1

Single Coupon Only

Example:

```text
One coupon per order
```

Benefits:

* Simpler pricing
* Lower abuse risk

---

## Strategy 2

Stackable Promotions

Example:

```text
Product Discount

+

Free Shipping
```

Requires conflict resolution.

---

# Coupon Priority System

When multiple promotions apply:

Priority determines execution order.

---

Example:

```text
Campaign Priority

100 = Highest

10 = Lowest
```

Higher priority executes first.

---

# Abuse Prevention Architecture

Coupon abuse directly impacts profitability.

---

## Common Abuse Patterns

Examples:

```text
Multiple Account Creation

Referral Abuse

Repeated Redemption

Automation Scripts
```

---

# Redemption Limits

---

## Global Usage Limit

Example:

```text
First 10,000 Redemptions
```

After limit reached:

Coupon expires automatically.

---

## Per User Limit

Example:

```text
1 Redemption Per Customer
```

---

## Daily Limits

Example:

```text
Maximum 500 Redemptions Per Day
```

---

# Fraud Protection

Additional safeguards include:

---

## Account Verification

Require:

```text
Email Verification

Phone Verification
```

---

## Device Monitoring

Track:

```text
Device Fingerprints

IP Patterns
```

---

## Velocity Checks

Detect:

```text
Excessive Redemption Attempts
```

---

# High-Concurrency Coupon Handling

Large campaigns create traffic spikes.

Example:

```text
Festival Sale

Flash Sale

Launch Event
```

---

## Challenges

* Simultaneous validations
* Usage limit contention
* Database pressure

---

# Redis Optimization

Frequently accessed coupon metadata stored in Redis.

---

## Cached Data

```text
Coupon Rules

Usage Counters

Campaign Metadata

Eligibility Information
```

---

## Cache Keys

Examples:

```text
coupon:WELCOME10

coupon_usage:WELCOME10

campaign:summer_sale
```

---

## Benefits

* Faster validation
* Reduced database load
* Better scalability

---

# Coupon Usage Tracking

Every redemption must be recorded.

---

## Stored Data

```text
User

Coupon

Order

Discount Amount

Timestamp
```

---

## Benefits

* Reporting
* Fraud detection
* Auditing

---

# Cache Invalidation Strategy

Coupon data changes frequently.

---

## Invalidate On

```text
Coupon Updated

Coupon Disabled

Campaign Expired

Usage Limit Reached
```

---

## Refresh Flow

```text
Coupon Updated
       ↓
Invalidate Cache
       ↓
Reload Metadata
```

---

# Failure Handling

The coupon engine must handle failures gracefully.

---

## Scenario 1

Redis Unavailable

Resolution:

Fallback to database validation.

---

## Scenario 2

Usage Counter Mismatch

Resolution:

Reconcile with database.

---

## Scenario 3

Expired Coupon Cached

Resolution:

Strict cache expiration strategy.

---

# Monitoring Metrics

## Business Metrics

Track:

* Coupon usage rate
* Discount volume
* Revenue impact
* Campaign performance

---

## Security Metrics

Track:

* Redemption failures
* Abuse attempts
* Duplicate redemptions

---

## Technical Metrics

Track:

* Validation latency
* Cache hit ratio
* Redis performance
* Database queries

---

# Production Incidents and Lessons

## Incident 1

Coupon Usage Limit Exceeded

Cause:

Race condition during flash sale.

Resolution:

Atomic counter updates.

---

## Incident 2

Expired Coupon Accepted

Cause:

Stale cache entry.

Resolution:

Short TTL and proactive invalidation.

---

## Incident 3

Discount Applied Twice

Cause:

Promotion stacking bug.

Resolution:

Promotion priority enforcement.

---

## Incident 4

Heavy Campaign Overloaded Database

Cause:

Millions of validation requests.

Resolution:

Redis-first validation architecture.

---

# Scalability Strategy

The coupon engine scales through:

### Stateless Services

Easy horizontal expansion.

---

### Redis Caching

Fast validation lookups.

---

### Atomic Counters

Accurate usage tracking.

---

### Optimized Database Queries

Reduced validation latency.

---

### Event-Driven Campaign Updates

Efficient cache refresh workflows.

---

# Reliability Principles

The coupon engine follows:

* Validate before calculation
* Enforce limits consistently
* Record every redemption
* Prevent duplicate processing
* Protect against abuse
* Cache aggressively
* Reconcile continuously

---

# Engineering Outcomes

The Coupon Engine Architecture provides:

* Flexible promotion management
* Accurate discount calculation
* Fraud resistance
* High scalability
* Fast validation
* Revenue protection
* Campaign observability
* Operational reliability

This architecture enables ecommerce platforms to run large-scale promotional campaigns while maintaining pricing integrity, customer trust, and business profitability.
