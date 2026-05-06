# cinderhaven-data

The shared dataset behind the [Cinderhaven Provisions](https://github.com/MsShawnP) portfolio. Cinderhaven is a fictional ~$25M specialty food brand with 90 SKUs across three product lines, selling through Walmart, Costco, Whole Foods, regional chains, UNFI/KeHE distribution, and DTC.

This repo is the single source of truth for the database and the scripts that generate it. Other repos consume the built database — they don't maintain their own copies.

## The dataset

**`cinderhaven_product_master.db`** — a SQLite database with the following tables:

| Table | Rows | What it contains |
|---|---|---|
| `product_master` | 90 | SKU-level attributes, GTINs, UPCs, case dimensions, nutritional info, active retailers, 1WorldSync status. Includes deliberate data-quality defects (missing GTINs, truncated descriptions, wrong units, etc.) |
| `stores` | ~902 | Store/door list with retailer, chain name, region, state, volume tier |
| `distribution_log` | ~12,595 | SKU × store authorization history. Reflects data-quality-driven delays and deauthorizations |
| `sku_costs` | 90 | COGS, landed cost, retailer-specific wholesale prices, trade spend rates by channel |
| `price_history` | varies | Time-based wholesale price and MSRP changes over the 104-week window |
| `promotions` | ~75 | Promotional events with retailer-specific patterns (Walmart = frequent/shallow, Costco = infrequent/deep) |
| `chargebacks` | varies | Defect-driven chargebacks traceable to actual `product_master` data-quality issues |
| `scan_data` | ~1.19M | Weekly unit and dollar sales by SKU × store. 18–24 months of history with seasonal patterns, promo lifts, post-promo dips, stockout events, launch ramp curves, cannibalization effects, and retailer-specific pricing |

## The thesis

Every downstream problem in the dataset — chargebacks, slow launches, delisted SKUs, missed velocity thresholds — is causally traceable to data-quality defects in `product_master`. Clean SKUs perform well. Dirty SKUs get chargebacks, delayed distribution, and eventually deauthorized. The dataset is designed to make that story discoverable.

## Revenue validation

Annual wholesale revenue should land at **$23M–$27M** (~$25M target). Channel split: Walmart ~50%, UNFI ~18%, Whole Foods ~10%, Regional ~9%, Costco ~9%, DTC ~3%.

## Generation scripts

Run in order. Scripts 01–04b can run in any order relative to each other. Script 05 depends on all prior tables. Script 06 validates everything.

| Script | Purpose |
|---|---|
| `defect_profile.py` | Defines the data-quality defect types baked into `product_master` |
| `01_generate_stores.py` | Builds the store/door list |
| `02_generate_distribution.py` | Creates SKU × store authorization history |
| `03_generate_costs.py` | Generates COGS, wholesale prices, trade spend rates |
| `04_generate_promos.py` | Creates promotional events by retailer |
| `04b_generate_price_history.py` | Adds time-based pricing changes |
| `05_generate_scan_data.py` | Generates ~1.19M rows of weekly scan data (depends on all prior tables) |
| `06_validate_dataset.py` | Validates row counts, revenue targets, referential integrity |

After regeneration, always verify revenue lands at $23–27M.

## Repos that use this dataset

- **[retail-velocity-decision-tool](https://github.com/MsShawnP/retail-velocity-decision-tool)** — Velocity decision tool for specialty food CEOs. [Try it live →](https://velocity-tool.streamlit.app/)
- **[product-data-audit-demo](https://github.com/MsShawnP/product-data-audit-demo)** — SQL diagnostic queries for auditing product master data quality

## Setup for consuming repos

1. Clone this repo (or download `cinderhaven_product_master.db` from the latest release)
2. Place the database file in the consuming repo's `data/` directory
3. That's it

Each consuming repo's README documents the expected path.

## Tools

Python, SQLite, pandas
