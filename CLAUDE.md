# NorthWave Data Warehouse — Project Context

This is the practical project for a Udemy data warehousing course. We're
building an analytical warehouse for NorthWave, a fictional multi-country
outdoor retailer.

## Business Context

NorthWave is a multi-country outdoor retailer selling hiking gear, running
shoes, cycling equipment, and camping supplies. They sell through website,
mobile app, and physical stores. They operate in five countries: Germany,
the UK, the USA, Italy, and France.

Stakeholder: Sarah Mitchell, VP of Analytics. She has five business
questions to answer (see docs/brief.md) and a six-week deadline.

Business Questions:
1. Customer profitability by tier
2. Marketing ROI by channel
3. Inventory turns by region
4. Year-over-year sales growth
5. Customer retention by signup cohort

## Source Systems

NorthWave has 5 source systems feeding the warehouse, providing 7 source files
in source_data/:

| Source System | Purpose | Files Provided |
|---|---|---|
| CRM platform | USA, UK customers | customers_crm.csv |
| SAP | Germany, Italy, France customers + product master | customers_sap.csv, products_sap.csv |
| E-commerce platform | Catalog + denormalized orders | products_ecommerce.csv, orders_ecommerce.csv |
| Warehouse management system | Daily inventory snapshots | inventory_snapshots.csv |
| Marketing platform | Event delivery logs | marketing_events.csv |

## Tool Stack

- MotherDuck (cloud DuckDB) — data warehouse
- DuckDB CLI — for loading local CSVs into MotherDuck
- Claude Code — primary tool for files, SQL, Git, and MotherDuck via MCP
- Claude Desktop — secondary tool, used for MotherDuck Dives (dashboards)
- dbdiagram.io — schema visualization
- Git — version control (local-only for course; GitHub optional)

## Naming Conventions

- Table names: snake_case, prefixed by layer (bronze_, silver_, gold_)
- Surrogate keys: end in _key (customer_key, product_key)
- Natural keys: end in _id or named natural_*_id (preserved from source)
- Date columns: end in _date for actual dates, _key for date dimension surrogates

## Database Conventions

- Course testing happens in test_db
- Production recording happens in northwave_dwh
- Three schemas per database: bronze, silver, gold

## Locked Decisions (Lesson 1)

These business decisions came out of Sarah's brief and the requirements interview:

- Currency: All revenue values are stored in EUR (converted at transaction
  time). Currency is NOT a dimension — it's a fixed assumption.
- Customer Tiers: Platinum, Premium, Standard (these are NorthWave's
  customer tier names — NOT Gold/Silver/Bronze, which would collide with
  warehouse layer names)
- Marketing Attribution: 30 days after a marketing event delivery. If a
  customer was targeted by multiple campaigns within 30 days, attribute to
  the MOST RECENT event before the purchase.
- Year-over-Year Comparison: Same period current year vs same period prior
  calendar year. Products that didn't exist in the prior year are excluded.
- Retention Definition: A customer is "active" if they made at least one
  purchase in the measurement period.
- Cohort Periods: Quarterly signup cohorts, measured at 12-month and
  24-month retention checkpoints.
- Inventory Turn Definition: Monthly turn rate = monthly sales quantity /
  average monthly inventory level.
- Profitability Definition: Gross margin = net revenue minus cost of goods
  sold (COGS). Reported by current AND historical customer tier.

## Business Processes (Lesson 2)

Three operational activities at NorthWave generate the data the warehouse
analyzes. Each becomes one fact table. See
[docs/business_processes.md](docs/business_processes.md) for the full
write-up.

- **Sales / order placement**
  - Fact type: Transaction
  - Grain: One row per order line
  - Supports: Q1 (profitability), Q2 (marketing ROI), Q3 (inventory turns),
    Q4 (YoY growth), Q5 (retention) — the spine of the warehouse

- **Daily inventory snapshot**
  - Fact type: Periodic snapshot
  - Grain: One row per warehouse × product × day
  - Supports: Q3 (inventory turns)

- **Marketing event delivery**
  - Fact type: Factless
  - Grain: One row per customer × campaign × channel × delivery date
  - Supports: Q2 (marketing ROI)

### Drill-across patterns

Two of Sarah's questions span two facts and rely on conformed dimensions
to combine measures correctly:

- **Q2 — Marketing ROI:** drill-across between the **sales fact** and the
  **marketing event fact**, joined through conformed **customer** and
  **calendar** dimensions (calendar is role-played — `order_date` on
  sales, `delivery_date` on marketing). The 30-day, last-touch attribution
  is applied in ETL and stamped onto the sales fact, not recomputed at
  query time.

- **Q3 — Inventory turns:** drill-across between the **sales fact** and
  the **inventory snapshot fact**, joined through conformed **product**,
  **warehouse**, and **calendar** (month grain) dimensions. Sales quantity
  is summed across the month; on-hand inventory is averaged across daily
  snapshots — aggregation order matters.

## Bus Matrix and Conformed Dimensions (Lesson 3)

The dimensional architecture is documented in full in
[docs/bus_matrix.md](docs/bus_matrix.md). Summary:

### The six dimensions
- **Customer** — buyer / message recipient; carries embedded customer
  geography (city → country → region) and SCD2 tier history
- **Product** — what was bought / what's on the shelf; carries
  subcategory → category → department hierarchy with category history
- **Calendar** — when the event or state occurred; day grain, used by
  all three facts
- **Warehouse** — where inventory lives and from which warehouse each
  order is fulfilled; carries warehouse → country → region hierarchy
  (separate from customer geography)
- **Campaign** — which marketing initiative drove an event delivery
- **Channel** — unified dim covering web, mobile_app, in_store, email,
  sms, app_push

### Conformed dimensions (5)
- **Customer** — Sales + Marketing Event
- **Product** — Sales + Inventory Snapshot
- **Calendar** — Sales + Inventory Snapshot + Marketing Event
  (role-played as `order_date`, `ship_date`, `snapshot_date`,
  `delivery_date`)
- **Warehouse** — Sales (via `fulfilling_warehouse_code`) + Inventory
  Snapshot
- **Channel** — Sales + Marketing Event

### Non-conformed dimensions (1)
- **Campaign** — Marketing Event only. Single-fact use is appropriate
  because attribution is a query-layer policy, not a schema decision —
  no campaign FK on Sales.

### Drill-across patterns
- **Marketing ROI (Q2):** Sales × Marketing Event, joined via conformed
  **Customer + Calendar**. The 30-day, last-touch attribution policy
  lives in the query layer, not stamped onto the Sales fact.
- **Inventory turns (Q3):** Sales × Inventory Snapshot, joined via
  conformed **Product + Warehouse + Calendar** (month grain). Sales
  quantity summed, on-hand inventory averaged across daily snapshots.

### Channel: documented dual meaning of "web"
The value `web` appears in both the sales-channel value set (website
purchase) and the marketing-channel value set (web ad / push delivery)
with different operational meanings. Documented as attributes on the
Channel dimension rather than split into `web_sales` and `web_marketing`
rows. Revisitable if it causes reporting confusion later.

## Star Schema (Lessons 4 and 5)

The gold layer comprises **6 dimensions** and **3 facts**, fully
specified in [docs/star_schema.md](docs/star_schema.md). Visual diagram
in [docs/diagrams/star_schema.dbml](docs/diagrams/star_schema.dbml)
(paste into dbdiagram.io to render).

### Dimension highlights

- **`dim_customer`** — SCD2 on `tier`, `country`, `region`, `city`.
  Neither CRM nor SAP exposes `effective_from`/`effective_to`; the
  silver/gold load detects changes by comparing incoming rows against
  the current dim row.
- **`dim_product`** — SCD2 on the 3-level hierarchy
  (`subcategory` / `category` / `department`). The two product sources
  (ECOM and SAP) join on `product_name`, not on ID — silver handles
  the merge.
- **`dim_calendar`** — All Type 0. Smart key in `YYYYMMDD` form.
  `prior_year_date_key` denormalised onto each row enables Q4 YoY
  with a single join.
- **`dim_warehouse`** — Direct map from `warehouses.csv`. All Type 1.
- **`dim_campaign`** — Only `natural_campaign_id` and `campaign_name`
  come from source; `start_date` and `end_date` are derived from
  MIN/MAX of `event_date` per campaign at ETL time.
- **`dim_channel`** — Reference dim. `natural_channel_code` from
  source; `channel_type` and `description` manually maintained.
  `channel_type='both'` on the `web` row solves the dual-meaning
  problem cleanly.

### Fact highlights

- **`fct_sales`** — Transaction fact, one row per order line. Carries
  both `order_date_key` and `ship_date_key` (calendar role-playing).
  `unit_cost_at_sale_eur` is stamped at silver from `products_sap` via
  the `product_name` join, then `cogs_eur` and `gross_margin_eur` are
  derived. Operational flags (`payment_method`, `is_gift`, `is_promo`,
  `is_guest_checkout`) kept as degenerate dimensions — they exist in
  source at no extra cost.
- **`fct_inventory_snapshot`** — Periodic snapshot, one row per
  product × warehouse × day. `units_on_hand` and `units_reserved` are
  **semi-additive on date** — sum across products/warehouses, average
  across days. `units_sold_that_day` is **reconciliation only** —
  `fct_sales` is the authoritative source for sales quantity; the WMS
  figure exists to detect drift between inventory and order systems.
- **`fct_marketing_event`** — Factless fact, one row per
  customer × campaign × channel × delivery date. `event_count = 1`
  measure for BI semantic layers. `customer_country` from source was
  **dropped** — the SCD2 customer dim resolves country at `event_date`
  via the join, so carrying it on the fact would duplicate dim work
  and risk drift.

### Cross-cutting

- **FK resolution at silver.** Natural keys are resolved to surrogate
  `*_key` values via SCD2 lookup at the fact's event date (`order_date`
  for sales, `snapshot_date` for inventory, `event_date` for
  marketing).
- **Unknown handling.** Every dim carries a `-1` sentinel "Unknown"
  row; unresolvable natural keys default to `-1` and are logged for
  ETL alerting. Rows are never dropped.
- **Currency.** All `*_eur` measures are stored in EUR. Currency is
  not a dimension.

## Source Data Simplification

For this course, NorthWave's source systems are represented as pre-generated
synthetic CSV files in source_data/. In a real production project, these
would be live databases (CRM, SAP, e-commerce platform, etc.) connected to
Claude via MCP servers. The engineering patterns are identical — only the
data access method differs.

## Methodology

We follow Kimball's 4-step dimensional design process, extended into 6
explicit steps for clarity:

1. Understand the business (Lesson 1)
2. Identify business processes with grain (Lesson 2)
3. Build the bus matrix — identify dimensions, declare conformity (Lesson 3)
4. Design the star schema in detail (Lesson 4)
5. Source-to-target mapping (Lesson 5)
6. Review and refine (ongoing across lessons)

Reference: Ralph Kimball, The Data Warehouse Toolkit (1996).

## How to Update This File

CLAUDE.md is a living document. After each lesson, update this file with the
decisions made in that lesson. Add a new section under "Locked Decisions"
named after the lesson topic. This preserves design context across sessions.
