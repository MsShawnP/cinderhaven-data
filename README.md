# cinderhaven-data

Seed dataset for the [Cinderhaven Provisions](https://github.com/MsShawnP) portfolio. Cinderhaven is a fictional ~$27.5M specialty food brand with 50 SKUs across three product lines, selling through Walmart, Costco, Whole Foods, KeHE, regional chains, UNFI distribution, and DTC (Shopify).

The SQLite database in `data/` is exported directly from **[cinderhaven-data-platform](https://github.com/MsShawnP/cinderhaven-data-platform)** (Fly.io Postgres, dbt pipeline). The generation scripts in `scripts/` are stale — they produce an older 90-SKU dataset and should not be run with `--force` unless you intend to overwrite the platform data.

## The dataset

**`data/cinderhaven_product_master.db`** — a 118 MB SQLite database with 30 tables.

### Base tables

| Table | Rows | What it contains |
|---|---|---|
| `product_master` | 50 | SKU-level attributes, GTINs, UPCs, case dimensions, nutritional info, active retailers |
| `stores` | ~902 | Store/door list with retailer, chain name, region, state, volume tier |
| `distribution_log` | ~12,500 | SKU x store authorization history |
| `sku_costs` | 50 | COGS, landed cost, retailer-specific wholesale prices, trade spend rates by channel (incl. KeHE) |
| `price_history` | ~398 | Time-based wholesale price and MSRP changes |
| `promotions` | ~195 | Promotional events with promo_cost and funding_mechanism |
| `chargebacks` | ~391 | Defect-driven chargebacks |
| `scan_data` | ~977K | Weekly unit and dollar sales by SKU x store. 157 weeks (2024-01-06 to 2027-01-02) |

### Deduction tables

| Table | Rows | What it contains |
|---|---|---|
| `retailers` | 11 | Canonical retailer reference (retailer_id, name, channel_type, dispute portal details) |
| `retailer_rules` | 90 | Per-retailer x deduction-type dispute rules |
| `deduction_codes` | 97 | Retailer-specific deduction codes |
| `edi_requirements` | 42 | Per-retailer compliance specs |
| `orders` | ~5,800 | Purchase orders from retailers |
| `order_lines` | ~30,000 | Line items per order |
| `shipments` | ~5,800 | Shipment records (carrier, BOL, delivery, ASN) |
| `pack_records` | ~5,800 | Pack/label compliance + evidence records |
| `deductions` | ~13,500 | Deduction records (trailing window through 2027-01-02) |
| `remittances` | ~2,700 | Payment events bundling deductions |
| `disputes` | ~6,100 | Dispute filings with evidence quality, outcome, recovery |
| `dispute_evidence` | ~12,800 | Evidence items per dispute |
| `post_audit_claims` | ~45 | Retroactive audit clawbacks |

### Shopify / DTC tables (new)

| Table | Rows | What it contains |
|---|---|---|
| `shopify_orders` | ~4,700 | DTC orders |
| `shopify_order_lines` | ~9,800 | Line items per DTC order |
| `shopify_chargebacks` | ~140 | DTC chargebacks |
| `shopify_refunds` | ~470 | DTC refunds |
| `shopify_transactions` | ~5,200 | Payment transactions |
| `shopify_payouts` | ~150 | Payout batches |
| `distributors` | 2 | Distributor reference (UNFI, KeHE) |
| `sku_distributors` | ~100 | SKU x distributor mapping |
| `retailer_requirements` | ~20 | Per-retailer compliance requirements |

## Revenue and deduction targets

- Annual wholesale revenue: **$27.5M** (trailing 52 weeks)
- Channel split: Walmart ~50%, UNFI ~18%, KeHE ~10%, Whole Foods ~8%, Regional ~7%, Costco ~5%, DTC ~3%
- Structural trade spend: **$5.2M (18.9%)** — negotiated rate-card
- Operational deductions (trailing 365 days): **$2.0M (7.2%)**
- All-in trade cost: **$7.2M (26.1%)**
- Disputes filed: ~6,100, recovered $988K (19.8% rate)
- Double-dip deductions: 3 flagged cases, ~$19.5K

## Data source

The database is exported from the cinderhaven-data-platform Postgres instance. To re-sync:

```bash
# Requires fly proxy running (localhost:5432)
python export_pg_to_sqlite.py
```

The generation scripts (`scripts/01-15`) are stale — they produce a 90-SKU dataset from 2024. They remain in the repo for reference but should not be used to rebuild the current database.

## Downstream

- **[trade-spend-data-diagnostic](https://github.com/MsShawnP/trade-spend-data-diagnostic)** — Excel workbook diagnostic (consumes via git submodule)
- **[cinderhaven-data-platform](https://github.com/MsShawnP/cinderhaven-data-platform)** — dbt pipeline on Fly.io Postgres (upstream source)
- **[retailer-deduction-recovery](https://github.com/MsShawnP/retailer-deduction-recovery)** — Interactive React demo
- **[retail-velocity-decision-tool](https://github.com/MsShawnP/retail-velocity-decision-tool)** — Velocity decision tool

## Setup

1. Clone this repo
2. The database is already at `data/cinderhaven_product_master.db` (118 MB)

No build step required — the DB is committed directly.

## Tools

Python 3.10+ (stdlib only), SQLite

## License

MIT — see [LICENSE](LICENSE).
