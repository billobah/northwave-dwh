# NorthWave Data Warehouse

Analytical data warehouse for **NorthWave**, a fictional multi-country outdoor
retailer (hiking, running, cycling, camping) operating in Germany, the UK, the
USA, Italy, and France. Built as the practical project of a Udemy data
warehousing course, following Kimball's dimensional modeling methodology.

## Business Questions

The warehouse answers five questions from the VP of Analytics
(see [docs/brief.md](docs/brief.md)):

1. Customer profitability by tier
2. Marketing ROI by channel
3. Inventory turns by region
4. Year-over-year sales growth
5. Customer retention by signup cohort

## Architecture

Medallion layers in MotherDuck, one schema per layer:

| Layer | Purpose |
|---|---|
| `bronze` | Raw source data, loaded as-is from CSV |
| `silver` | Cleaned, merged, conformed data; natural keys resolved to surrogate keys |
| `gold` | Star schema: 6 dimensions and 3 facts |

**Gold layer**

- Dimensions: `dim_customer` (SCD2), `dim_product` (SCD2), `dim_calendar`,
  `dim_warehouse`, `dim_campaign`, `dim_channel`
- Facts: `fct_sales` (transaction), `fct_inventory_snapshot` (periodic
  snapshot), `fct_marketing_event` (factless)

Databases: `test_db` for experimentation, `northwave_dwh` for production.

## Source Data

Synthetic CSV extracts from five source systems, in `source_data/`:

| Source system | Files |
|---|---|
| CRM (USA, UK customers) | `customers_crm.csv` |
| SAP (DE, IT, FR customers + product master) | `customers_sap.csv`, `products_sap.csv` |
| E-commerce platform | `products_ecommerce.csv`, `orders_ecommerce.csv` |
| Warehouse management system | `inventory_snapshots.csv` |
| Marketing platform | `marketing_events.csv` |

## Tool Stack

- [MotherDuck](https://motherduck.com/) (cloud DuckDB): the data warehouse
- DuckDB CLI / Python: loading local CSVs into MotherDuck
- Claude Code: SQL, files, Git, and MotherDuck via MCP
- dbdiagram.io: schema visualization

## Getting Started

```bash
# Create and activate a virtual environment
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # macOS / Linux

pip install duckdb

# Authenticate to MotherDuck (keep the token out of Git, e.g. in .env)
set motherduck_token=<your_token>
```

## Conventions

- Tables: `snake_case`, prefixed by layer (`bronze_`, `silver_`, `gold_`)
- Surrogate keys end in `_key`; natural keys end in `_id` / `natural_*_id`
- Date columns end in `_date`; date-dimension foreign keys end in `_key`
- All monetary values are stored in EUR

## Project Structure

```
northwave-dwh/
├── CLAUDE.md        # Design decisions and project context (living document)
├── docs/            # Business brief and design documentation
└── source_data/     # Source system CSV extracts
```

For full design decisions (grain, bus matrix, SCD strategy, attribution rules),
see [CLAUDE.md](CLAUDE.md).
