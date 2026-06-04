# Redis Strategy

## Overview

Redis is a critical component of the ecommerce architecture and serves as the primary in-memory data store for caching, session management, cart acceleration, coupon validation, and performance optimization.

Without an effective caching strategy, large ecommerce platforms quickly encounter:

* Database bottlenecks
* Increased API latency
* Poor customer experience
* Scalability limitations
* Higher infrastructure costs

The Redis architecture was designed to:

* Reduce database load
* Improve response times
* Support high traffic volumes
* Enable horizontal scaling
* Maintain operational reliability

This document explains how Redis is utilized across the platform, including cache design, invalidation patterns, consistency considerations, and production lessons learned.

---

# Why Redis

Ecommerce workloads are heavily read-oriented.

Example:

```text id="a1m8v2"
Product Views

Category Browsing

Search Queries

Cart Retrieval

Session Validation
```

Reads significantly outnumber writes.

Redis provides:

* In-memory speed
* Low latency
* High throughput
* Flexible data structures
* Horizontal scalability

---

# Redis Architecture

High-level architecture:

```text id="h5xq8v"
Client
   ↓
Application Layer
   ↓
Redis
   ↓
MySQL
```

Read flow:

```text id="m4u9pw"
Request
   ↓
Redis Lookup
   ↓
Cache Hit
   ↓
Response
```

Cache miss:

```text id="k2v7ra"
Request
   ↓
Redis Miss
   ↓
Database Query
   ↓
Redis Update
   ↓
Response
```

---

# Redis Use Cases

Redis supports multiple domains.

---

## Product Caching

Stores:

```text id="s7n2fj"
Product Details

Product Variants

Category Data

Featured Products
```

---

## Cart Caching

Stores:

```text id="p8y3td"
Cart State

Cart Items

Cart Totals

Checkout Metadata
```

---

## Session Caching

Stores:

```text id="g6m4ze"
User Sessions

Refresh Tokens

Authentication Metadata
```

---

## Coupon Caching

Stores:

```text id="r3f9bk"
Coupon Rules

Campaign Metadata

Usage Counters
```

---

## Inventory Metadata

Stores:

```text id="v8n1lh"
Availability Information

Inventory Summaries
```

Important:

Final inventory validation always occurs against MySQL.

---

# Product Cache Design

Products receive high read traffic.

---

## Cache Key Structure

Examples:

```text id="d7q5ya"
product:101

product:205

product:999
```

---

## Category Cache

Examples:

```text id="c4w2kn"
category:7

category:15
```

---

## Featured Products

Example:

```text id="j9r8eh"
featured_products
```

---

# Cart Cache Design

Cart interactions require extremely low latency.

---

## Key Structure

Example:

```text id="u3v6pg"
cart:user:123
```

---

## Stored Data

```json id="f2x8me"
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

* Faster retrieval
* Reduced database pressure
* Improved customer experience

---

# Session Cache Design

Authentication performance depends heavily on Redis.

---

## Session Keys

Examples:

```text id="l7m2cw"
session:user:123

session:user:456
```

---

## Refresh Token Keys

Examples:

```text id="b5n9zr"
refresh:user:123
```

---

# Coupon Cache Design

Coupon validation occurs frequently during promotions.

---

## Cached Data

```text id="t6y4ka"
Coupon Rules

Usage Limits

Campaign Status
```

---

## Coupon Keys

Examples:

```text id="n8w1de"
coupon:WELCOME10

coupon:SUMMER25
```

---

# Cache-Aside Pattern

The platform primarily uses the Cache-Aside strategy.

---

## Flow

```text id="q1e5tm"
Application
      ↓
Redis Lookup
      ↓
Cache Miss
      ↓
Database Query
      ↓
Store In Redis
      ↓
Return Response
```

---

## Benefits

* Simplicity
* Reliability
* Database fallback support

---

# Write Strategy

Updates occur in the database first.

Flow:

```text id="z4p7gu"
Database Update
      ↓
Cache Invalidation
      ↓
Cache Rebuild
```

This ensures database remains the source of truth.

---

# Cache Invalidation Strategy

One of the most important engineering concerns.

---

## Product Invalidation

Triggered by:

```text id="e2r6ys"
Product Update

Variant Update

Category Update
```

---

## Cart Invalidation

Triggered by:

```text id="h9f3lm"
Add Item

Update Quantity

Remove Item
```

---

## Coupon Invalidation

Triggered by:

```text id="w6x1tb"
Coupon Update

Campaign Expiration

Usage Limit Reached
```

---

## Session Invalidation

Triggered by:

```text id="p7k8cv"
Logout

Password Change

Token Revocation
```

---

# TTL Strategy

Not all cached data should live forever.

---

## Product Cache TTL

Example:

```text id="r5n2yf"
30 Minutes
```

---

## Category Cache TTL

Example:

```text id="u1m7eg"
60 Minutes
```

---

## Session TTL

Example:

```text id="v4x9jq"
Session Lifetime
```

Aligned with authentication policy.

---

## Coupon TTL

Example:

```text id="c8r5zw"
Campaign Duration
```

---

# Cache Consistency Model

The platform balances performance and consistency.

---

## Strong Consistency

Required for:

```text id="f3k1yb"
Payments

Orders

Inventory
```

---

## Eventual Consistency

Acceptable for:

```text id="s6p4nc"
Product Pages

Category Listings

Featured Products
```

---

# Cache Stampede Protection

Popular products can cause cache stampedes.

---

## Problem

```text id="x8w2gh"
Cache Expires
       ↓
Thousands Of Requests
       ↓
Database Overload
```

---

## Solution

Techniques:

* Staggered TTLs
* Background refresh
* Request coalescing

---

# Hot Key Management

Certain products receive extremely high traffic.

Example:

```text id="m1e9zk"
Flash Sale Product
```

---

## Risk

Single Redis key becomes overloaded.

---

## Mitigation

Use:

* Key partitioning
* Controlled refresh
* Traffic distribution

---

# Redis High Availability

Redis must not become a single point of failure.

---

## Architecture

```text id="y7n3vf"
Primary Redis
      ↓
Replica Redis
```

---

## Benefits

* Fault tolerance
* Faster recovery
* Improved resilience

---

# Scaling Redis

As traffic grows:

### Vertical Scaling

Increase:

* Memory
* CPU

---

### Horizontal Scaling

Introduce:

```text id="g2m8ar"
Redis Cluster
```

Benefits:

* More capacity
* Better throughput

---

# Monitoring Strategy

Redis requires continuous monitoring.

---

## Performance Metrics

Track:

```text id="n4x7pq"
Memory Usage

CPU Usage

Latency

Throughput
```

---

## Cache Metrics

Track:

```text id="b7w5jc"
Hit Rate

Miss Rate

Evictions

Key Count
```

---

## Operational Metrics

Track:

```text id="d9k2rh"
Replication Status

Connection Count

Persistence Health
```

---

# Failure Handling

Redis failures must not break ecommerce operations.

---

## Scenario 1

Redis Unavailable

Resolution:

Fallback to database.

---

## Scenario 2

Cache Corruption

Resolution:

Invalidate and rebuild.

---

## Scenario 3

Memory Exhaustion

Resolution:

Eviction policies and monitoring.

---

## Scenario 4

Replication Failure

Resolution:

Automatic failover procedures.

---

# Production Incidents and Lessons

## Incident 1

Stale Product Data

Cause:

Missed cache invalidation.

Resolution:

Event-driven invalidation.

---

## Incident 2

Expired Coupon Remained Cached

Cause:

Incorrect TTL.

Resolution:

Campaign-aware expiration logic.

---

## Incident 3

Cart Data Lost

Cause:

Aggressive eviction policy.

Resolution:

Persistent cart storage in MySQL.

---

## Incident 4

Redis Saturation During Flash Sale

Cause:

Unexpected traffic spike.

Resolution:

Cluster scaling and hot key mitigation.

---

# Reliability Principles

The Redis strategy follows:

* Database remains source of truth
* Cache accelerates reads
* Invalidate aggressively
* Monitor continuously
* Design for failure
* Protect against stampedes
* Scale proactively

---

# Engineering Outcomes

The Redis architecture provides:

* Fast product delivery
* Low-latency cart operations
* Efficient authentication
* Scalable coupon validation
* Reduced database load
* Improved customer experience
* Horizontal scalability
* Operational resilience

This Redis strategy enables the ecommerce platform to support high traffic volumes, large catalogs, and complex business workflows while maintaining excellent performance and reliability.
