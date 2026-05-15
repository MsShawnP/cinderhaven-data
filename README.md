# cinderhaven-data

The generation layer for the [Cinderhaven Provisions](https://github.com/MsShawnP) portfolio dataset. Cinderhaven is a fictional ~$25M specialty food brand with 90 SKUs across three product lines, selling through Walmart, Costco, Whole Foods, regional chains, UNFI/KeHE distribution, and DTC.

This repo generates and validates the SQLite database. The
[Cinderhaven Data Platform](https://github.com/MsShawnP/cinderhaven-data-platform)
ingests it into Postgres (dbt + Dagster) and serves as the analytical
source of truth for most portfolio projects. This repo remains the
canonical place to regenerate the dataset or modify the generation
scripts.

## The dataset

**`cinderhaven_product_master.db`** — a SQLite database with the following tables:

### Base tables

| Table | Rows | What it contains |
|---|---|---|
| `product_master` | 90 | SKU-level attributes, GTINs, UPCs, case dimensions, nutritional info, active retailers, 1WorldSync status. Includes deliberate data-quality defects |
| `stores` | ~902 | Store/door list with retailer, chain name, region, state, volume tier |
| `distribution_log` | ~12,507 | SKU × store authorization history. Reflects data-quality-driven delays and deauthorizations |
| `sku_costs` | 90 | COGS, landed cost, retailer-specific wholesale prices, trade spend rates by channel |
| `price_history` | ~398 | Time-based wholesale price and MSRP changes over the 104-week window |
| `promotions` | ~195 | Promotional events with retailer-specific patterns, promo_cost, and funding_mechanism |
| `chargebacks` | ~391 | Defect-driven chargebacks traceable to actual `product_master` data-quality issues |
| `scan_data` | ~1.19M | Weekly unit and dollar sales by SKU × store. 104 weeks (2024-05-11 to 2026-05-02) |

### Deduction tables

| Table | Rows | What it contains |
|---|---|---|
| `retailers` | 11 | Canonical retailer reference (retailer_id, name, channel_type, dispute portal details) |
| `retailer_rules` | 90 | Per-retailer × deduction-type dispute rules (window_days, evidence_required, recovery_rate) |
| `deduction_codes` | 97 | Retailer-specific deduction codes (Walmart Code 22, KeHE UDR, etc.) |
| `edi_requirements` | 42 | Per-retailer compliance specs (label, pallet, ASN, OTIF) |
| `orders` | ~5,800 | Purchase orders from retailers to Cinderhaven |
| `order_lines` | ~30,000 | Line items per order (SKU, units, price) |
| `shipments` | ~5,800 | What was shipped (carrier, BOL, delivery, ASN, short/damage flags) |
| `pack_records` | ~5,800 | What was packed/labeled (label compliance, evidence format/location) |
| `deductions` | ~3,000 | Deduction records: short_ship, label_fine, pallet_fine, damaged, late_delivery, promo_billback, vague, spoilage, slotting |
| `remittances` | ~500 | Payment events bundling deductions per retailer × week |
| `disputes` | ~1,400 | Dispute filings with evidence quality, outcome, recovery amount, labor hours |
| `dispute_evidence` | ~3,000 | Evidence items per dispute (submitted/required/format) |
| `post_audit_claims` | ~45 | Retroactive audit clawbacks (Walmart APL, KeHE pass-throughs) |

## The thesis

Every downstream problem in the dataset — chargebacks, slow launches, delisted SKUs, missed velocity thresholds — is causally traceable to data-quality defects in `product_master`. The deduction tables extend this by modeling five compounding failures at scale: no visibility into deduction patterns, process gaps generating avoidable deductions, weak evidence, inaccessible records, and missed dispute windows.

## Revenue and deduction targets

- Annual wholesale revenue: **$23M–$27M** (~$25M target)
- Channel split: Walmart ~50%, UNFI ~18%, Whole Foods ~10%, Regional ~9%, Costco ~9%, DTC ~3%
- Annualized deductions: **$750K–$1.2M** (3–5% of wholesale revenue)
- Recovery rate: 5–10% (lean team, mostly handwritten evidence)
- Double-dip deductions: 3 flagged cases totaling ~$15K–$20K

## Generation scripts

Run via `build_db.py`. The full pipeline produces both the base dataset and the deduction extension in a single build.

### Base pipeline

| Script | Purpose |
|---|---|
| `build_db.py` | Orchestrates the full generation pipeline |
| `seed_product_master.sql` | Seeds 90 SKUs with deliberate data-quality defects |
| `01_generate_stores.py` | Store/door list |
| `02_generate_distribution.py` | SKU × store authorization history |
| `02b_generate_chargebacks.py` | Defect-driven chargebacks |
| `03_generate_costs.py` | COGS, wholesale prices, trade spend |
| `04_generate_promos.py` | Promotional events with promo_cost and funding_mechanism |
| `04b_generate_price_history.py` | Time-based pricing changes |
| `05_generate_scan_data.py` | ~1.19M rows of weekly scan data |
| `06_validate_dataset.py` | Base table validation |

### Deduction pipeline

| Script | Purpose |
|---|---|
| `seed_deduction_schema.sql` | DDL for 13 new tables |
| `seed_deduction_static.sql` | Static seed data (retailers, rules, codes, EDI requirements) |
| `07_seed_deduction_tables.py` | Applies schema + static seeds |
| `08_generate_orders.py` | Purchase orders + order_lines |
| `09_generate_pack_records.py` | Pack/label compliance + evidence records |
| `10_generate_shipments.py` | Shipments + BOL/POD/ASN flags |
| `11_generate_deductions.py` | Deduction records + double-dip seeding |
| `12_generate_post_audit_claims.py` | Retroactive audit clawbacks |
| `13_generate_remittances.py` | Payment events bundling deductions |
| `14_generate_disputes.py` | Dispute filings + evidence |
| `15_validate_deductions.py` | Deduction-specific validation |

The built database is not committed — it's ~164 MB and regenerable. Run:

```bash
python scripts/build_db.py          # build if missing
python scripts/build_db.py --force  # rebuild from scratch
```

## Downstream consumers

### Via Postgres (Cinderhaven Data Platform)

These projects read from the data platform's Postgres mart tables, not
from the SQLite database directly. To update their data, regenerate the
SQLite here, then re-run the platform's ingest pipeline.

- **[cinderhaven-data-platform](https://github.com/MsShawnP/cinderhaven-data-platform)** — Ingests this SQLite into Postgres via COPY loader. dbt transforms raw → staging → marts. Dagster orchestrates.
- **[retail-velocity-decision-tool](https://github.com/MsShawnP/retail-velocity-decision-tool)** — Dash decision tool for specialty food CEOs. [Try it live →](https://retail-velocity-decision-tool.fly.dev)
- **[retailer-deduction-recovery](https://github.com/MsShawnP/retailer-deduction-recovery)** — Interactive React demo tracing retailer deduction losses through five compounding failures. Exports Postgres data to static JSON.
- **[product-data-audit-demo](https://github.com/MsShawnP/product-data-audit-demo)** — 53-query product data audit with HTML/PDF report generator.
- **[trade-spend-data-diagnostic](https://github.com/MsShawnP/trade-spend-data-diagnostic)** — Trade spend diagnostic workbook quantifying $1M margin leak.
- **[product-data-health-audit](https://github.com/MsShawnP/product-data-health-audit)** — R/Quarto product data health audit with Shiny calculator.

### Via SQLite (directly)

These projects read from the built `.db` file without going through the
data platform.

- **[short-ship-cost](https://github.com/MsShawnP/short-ship-cost)** — Extracts product, cost, and store data into a local SQLite for order generation and cost modelling.
- **[cinderhaven-data-dirty](https://github.com/MsShawnP/cinderhaven-data-dirty)** — Degrades the clean database with realistic data-quality defects for data-hygiene portfolio work.

## Setup

1. Clone this repo
2. Run `python scripts/build_db.py`
3. To update Postgres consumers: run the data platform's ingest pipeline
4. To update SQLite consumers: place or symlink the `.db` file in the consuming repo's `data/` directory

## Tools

Python, SQLite

## License

MIT — see [LICENSE](LICENSE).
