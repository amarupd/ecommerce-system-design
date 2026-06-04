# Product Catalog Architecture

## Overview

The Product Catalog is the foundation of every ecommerce platform. It serves as the central source of truth for product information, category structures, variants, inventory relationships, search indexing, filtering capabilities, and customer-facing product discovery.

A poorly designed catalog creates operational bottlenecks, slow search experiences, inventory inconsistencies, and scalability limitations. For this reason, the catalog architecture was designed to support:

* Large product inventories
* Complex variant structures
* Fast product discovery
* Advanced filtering
* SEO optimization
* Inventory integration
* Horizontal scalability

This document explains the architectural decisions, data modeling strategies, performance optimizations, and production challenges involved in building a scalable ecommerce catalog.

---

# Business Requirements

The catalog must support:

### Customer Requirements

* Browse products quickly
* Search products efficiently
* Filter by attributes
* View variant options
* Access product specifications
* Mobile-friendly discovery

### Business Requirements

* Manage large catalogs
* Support seasonal collections
* Handle bulk imports
* Enable merchandising controls
* Maintain SEO visibility

### Engineering Requirements

* Fast response times
* Cache-friendly architecture
* Efficient indexing
* Inventory synchronization
* Scalable querying

---

# Catalog Domain Model

The catalog is built around several primary entities.

```text
Category
    ↓
Product
    ↓
Variant
    ↓
Inventory
```

Supporting entities:

```text
Category
Product
Variant
Attribute
Attribute Value
Image
Review
Inventory
Coupon Mapping
```

---

# Category Architecture

## Category Hierarchy

Categories are organized using a multi-level hierarchy.

Example:

```text
Sports
 ├── Cricket
 │     ├── Jerseys
 │     ├── Bats
 │     └── Accessories
 │
 ├── Football
 │     ├── Jerseys
 │     └── Footwear
 │
 └── Basketball
```

Benefits:

* Better navigation
* Improved SEO
* Easier merchandising
* Scalable taxonomy

---

## Category Fields

Typical category model:

```sql
id
parent_id
name
slug
description
image
is_active
sort_order
created_at
updated_at
```

---

## Slug Strategy

Each category receives a unique slug.

Examples:

```text
cricket
football
cricket-jerseys
sports-accessories
```

Benefits:

* SEO friendly URLs
* Better indexing
* Cleaner navigation

---

# Product Architecture

Products act as parent entities.

Example:

```text
SW Performance Jersey
```

Variants:

```text
Small
Medium
Large
XL
XXL
```

The product stores shared information while variants store purchasable details.

---

## Product Fields

```sql
id
category_id
name
slug
description
short_description
brand
status
is_featured
seo_title
seo_description
created_at
updated_at
```

---

# Variant Architecture

## Why Variants Exist

Customers purchase variants, not products.

Examples:

### Apparel

```text
Size: M
Size: L
Size: XL
```

### Color

```text
Black
Red
Blue
```

### Combined

```text
Black + M
Black + L
Red + M
Red + L
```

Each combination becomes a variant.

---

## Variant Model

```sql
id
product_id
sku
price
compare_price
stock_quantity
weight
status
created_at
updated_at
```

---

# Attribute System

The attribute system provides flexibility without changing database schemas.

---

## Attributes

Examples:

```text
Size
Color
Material
Gender
Sport
```

---

## Attribute Values

Examples:

```text
Size
 ├── S
 ├── M
 ├── L
 └── XL

Color
 ├── Black
 ├── Blue
 └── Red
```

---

# SKU Strategy

Each purchasable variant receives a unique SKU.

Example:

```text
SW-JERSEY-BLK-M
SW-JERSEY-BLK-L
SW-JERSEY-RED-M
```

Benefits:

* Inventory tracking
* Warehouse operations
* Order fulfillment
* ERP integrations

---

# Product Images

Image architecture supports multiple images per product.

Example:

```text
Front View
Back View
Side View
Lifestyle Image
Zoom Image
```

---

## Image Storage

Images stored in:

```text
Amazon S3
```

Delivered through:

```text
CloudFront CDN
```

Benefits:

* Global delivery
* Lower latency
* Reduced application load

---

# Search Architecture

Search directly impacts conversion rates.

Goals:

* Fast results
* Relevant ranking
* Typo tolerance
* Attribute awareness

---

## Search Flow

```text
Customer Search
      ↓
Search API
      ↓
Cache Check
      ↓
Search Query
      ↓
Results Returned
```

---

## Searchable Fields

```text
Product Name
Category Name
Brand
SKU
Description
Tags
Attributes
```

---

# Filtering Architecture

Filters help customers narrow large catalogs.

Common filters:

```text
Category
Price
Brand
Size
Color
Rating
Availability
```

---

## Filter Query Example

```text
Category = Jerseys

Color = Black

Size = XL

Price < ₹1500
```

---

## Dynamic Filter Generation

Filters generated from:

```text
Category Attributes
Product Attributes
Inventory Status
```

Benefits:

* No hardcoded filters
* Flexible catalog growth

---

# SEO Architecture

Organic search traffic is critical.

Every product includes:

```text
SEO Title
SEO Description
Canonical URL
Slug
Structured Metadata
```

---

## Example URL

```text
/shop/product/sw-performance-jersey-black
```

Benefits:

* Search engine visibility
* Better rankings
* Higher CTR

---

# Catalog Caching Strategy

Product catalog traffic is highly read-heavy.

Redis stores:

```text
Product Details
Category Lists
Product Variants
Filter Metadata
Featured Products
```

---

## Cache Key Examples

```text
product:65

category:7

featured_products

product_variants:65
```

---

# Database Indexing Strategy

Critical indexes include:

```sql
INDEX(category_id)

INDEX(slug)

INDEX(status)

INDEX(is_featured)
```

Variants:

```sql
INDEX(product_id)

UNIQUE(sku)
```

---

# Catalog Scalability Challenges

## Challenge 1

Large Product Catalogs

Problem:

```text
100,000+ products
Millions of variants
```

Solution:

* Pagination
* Caching
* Efficient indexes

---

## Challenge 2

Slow Filtering

Problem:

Complex attribute combinations.

Solution:

* Composite indexes
* Cached filter metadata
* Optimized query plans

---

## Challenge 3

Search Performance

Problem:

Large search result sets.

Solution:

* Search indexing
* Cached results
* Query optimization

---

# Product Synchronization

Catalog updates impact:

* Inventory
* Search indexes
* Cache entries
* Promotions

Update flow:

```text
Product Update
      ↓
Database Update
      ↓
Cache Invalidation
      ↓
Search Refresh
      ↓
Customer Visibility
```

---

# Catalog Monitoring

Important metrics:

### Business Metrics

* Product views
* Search volume
* Conversion rate
* Search abandonment

---

### Engineering Metrics

* Search latency
* Cache hit rate
* Query execution time
* API response times

---

# Production Lessons Learned

### Lesson 1

Never couple inventory directly to product records.

Use dedicated inventory tables.

---

### Lesson 2

Always model purchasable entities as variants.

Avoid storing prices directly on products.

---

### Lesson 3

Design filtering architecture before scaling catalogs.

Retrofitting filters becomes expensive.

---

### Lesson 4

SEO architecture must be considered from the beginning.

Changing URL structures later creates migration complexity.

---

### Lesson 5

Cache invalidation is one of the hardest catalog problems.

Every product update must trigger predictable cache refresh workflows.

---

# Engineering Outcomes

The catalog architecture provides:

* Flexible product modeling
* Scalable variant management
* Fast search experiences
* Efficient filtering
* SEO optimization
* Inventory integration
* High read performance
* Operational simplicity

This design supports both growing D2C brands and enterprise ecommerce platforms with large product inventories while maintaining excellent customer experience and operational efficiency.
