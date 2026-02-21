---
model: Customer
collection: customers
id_pattern: CT-N (e.g. CT-1)
status: active
last_updated: 2026-02-21
tags: [core, webhook, cross-project]
depends_on: [pii]
referenced_by: [cohort]
created: 2026-02-21
---

# Customer

Materialized entity created/updated by Inventory-Management-Backend when orders are delivered.

## Fields

| Field | Type | Notes |
|-------|------|-------|
| customer_id | String | Unique, CT-N |
| pii_id | String | Unique, ref to PII |
| first_name | String | |
| last_name | String | |
| pincode | String | |
| sector | String | e.g. Agriculture |
| subsector | String | |
| source | Array | `[{name, first_seen}]` — acquisition sources |
| last_order_date | Date | |
| total_order_value | Number | Sum of delivered orders |
| max_order_value | Number | Highest single order |
| order_count | Number | |
| customer_value_tier | Enum | VCA+, VCA, VCB, VCC |
| category | Enum | Key Account, Prospect, Repeat Buyer, Customer |
| status | Enum | Active, Inactive |
| customer_profile | Enum | Retailer, Farmer, Gardner, Dealer, Indian Company Client, Big Company, International Gardener, Probable Micro Dealer, Government Body, Government Order Middlemen, Foreign Client, Government Client |

## Indexes

- `pii_id` (unique), `status`, `customer_value_tier`, `category`, `customer_profile`, `pincode`, `total_order_value`

## API

- `POST /webhook/upsert` — @Public, x-webhook-secret (Inventory-Management-Backend)
- `PUT /webhook/update/:pii_id` — @Public, x-webhook-secret
- `GET /`, `GET /:customer_id`, `GET /pii/:pii_id`, `PUT /:customer_id` — JWT auth

## Key Rules

- Only delivered orders (order_status=15) count towards metrics
- Emits `entity.updated` event on every write (cohort evaluation)
- pii_id is the universal join key across lead/contact/customer
