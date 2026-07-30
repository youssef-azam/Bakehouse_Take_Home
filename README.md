# Bakehouse Customer Analytics

End-to-end data engineering pipeline on Databricks, built with a medallion architecture (Bronze → Silver → Gold), to answer two business questions for Bakehouse, a franchise bakery chain:

- **Customer value** — which customers drive revenue, how they group by behavior, and which are slipping away.
- **Franchise expansion** — which locations perform, what geography says about demand, and where the case for a new branch is strongest.

## Architecture

| Layer | Notebook | Purpose |
|---|---|---|
| Setup | `Setup.ipynb` | Catalog/schema creation, source data exploration |
| Bronze | `Bronze layer.ipynb` | Raw ingestion of source tables with lineage metadata |
| Silver | `Silver layer.ipynb` | Deduplication, validation, referential integrity checks, PII masking |
| Gold | `Gold layer.ipynb` | Business-ready outputs: customer segments and franchise performance |

All tables are written as **Delta tables** using `workspace.bakehouse_{bronze|silver|gold}` schemas. Source data is read from `samples.bakehouse` (Databricks sample dataset, read-only).

## Bronze layer

Lands the three source tables — `sales_transactions`, `sales_customers`, `sales_franchises` — unmodified except for two ingestion metadata columns:

- `_ingested_at` — timestamp of ingestion
- `_source_table` — fully-qualified source table path, for lineage

No business logic applied at this layer by design — it's a raw, replayable copy of the source.

## Silver layer

Cleans and validates each table before anything moves downstream:

- **Deduplication** on business keys (`transactionID`, `customerID`, `franchiseID`)
- **Null-key filtering** — rows missing their primary identifier are dropped
- **Value validation** — non-positive `quantity` or `unitPrice` filtered out
- **Referential integrity** — transactions referencing a non-existent customer or franchise are dropped, with counts logged before removal
- **Consistency check** — flags where `totalPrice != quantity × unitPrice`
- **PII handling** — raw `cardNumber` is masked to the last 4 digits and the original column dropped before writing to silver

**Result:** across all 3,333 transactions, every check returned zero issues — no duplicates, no invalid values, no orphaned references, no price mismatches. The dataset was clean; the pipeline validates rather than assumes that.

## Gold layer

Two business-facing outputs, both queryable directly:

### `gold_customer_segments`
RFM (Recency, Frequency, Monetary) analysis, scored via quartiles per dimension and grouped into four segments: **Champions**, **At Risk**, **Lost**, **Regular**.

### `gold_franchise_performance`
Per-franchise revenue, transaction count, unique customers, and average transaction value, joined with franchise location data (city, country, size). A geography rollup surfaces revenue-per-franchise by country.

## Key findings

- **Champions** are 21.7% of customers but drive 29% of total revenue — the clearest retention priority.
- **At Risk** customers ($17,421 in revenue across 61 customers) represent the strongest "win-back" opportunity — previously high spenders who have gone quiet.
- Franchise revenue ranges from **$6,642** (Baked Bliss, Rome) to **$177** (Sapporo Sweets, Sapporo) — a ~37x spread.
- **Italy and Sweden** each run a single franchise at the highest revenue-per-location in the network — the strongest data-backed case for expansion.
- **Japan** carries the most franchises (20) but the lowest average revenue-per-franchise, and 4 of the 5 lowest-performing locations overall — worth auditing before further expansion there.

Full findings and recommendations are in [`Bakehouse_Findings.pptx`](./Bakehouse_Findings.pptx).

## Tech stack

- Databricks Free Edition
- PySpark
- Delta Lake
- Unity Catalog (`workspace` catalog, `bakehouse_bronze` / `bakehouse_silver` / `bakehouse_gold` schemas)
