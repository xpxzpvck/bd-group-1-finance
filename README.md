# TPC-H Lakehouse Migration - Group 1 Finance

A Lakehouse migration project for the TPC-H decision-support benchmark, built on Databricks. The goal is to design bronze, silver, and gold layers, build the data processing pipeline, and answer a set of finance business questions.

## Team

**Finance** - responsible for financial accuracy: reconciling order totals with line items, verifying business rules (discount ranges, positive values), and comparing monetary columns across layers.

## Architecture

The project follows a **medallion architecture** with three layers:

| Layer | Schema | Purpose |
|---|---|---|
| **Bronze** | `group1_finance.bronze` | Raw copy of `samples.tpch` — as-is, no transformations, metadata only |
| **Silver** | `group1_finance.silver` | 3NF with enforced data quality (not-null constraints, foreign keys, validation rules) |
| **Gold** | `group1_finance.gold` | Aggregated tables designed to answer the Finance business questions |

### Source Data

TPC-H ships with every Databricks workspace as `samples.tpch`:

```
customer  lineitem  nation  orders  part  partsupp  region  supplier
```

## Project Structure

```
bd-group-1-finance/
├── README.md
├── pyproject.toml
├── uv.lock
├── docs/
│   └── silver_er.md                # ER diagram of the silver layer (Mermaid)
└── notebooks/
    ├── 00_setup.ipynb              # Create catalog & schemas (run once)
    ├── 01_bronze.ipynb             # Copy TPC-H tables as-is into bronze
    ├── 02_silver_profiling.ipynb   # Profile bronze; derive the silver DQ rules (read-only)
    ├── 02_silver.ipynb             # 3NF + enforced data quality + quarantine
    └── 03_gold.ipynb               # Finance business questions
```

## How to Run

### Prerequisites

- A Databricks workspace with access to `samples.tpch`

### Steps

Run the notebooks in order. Each notebook is idempotent — safe to re-run from the top.

1. **Clone the repo** into your Databricks workspace (or import the notebooks manually).

2. **Run `00_setup`** — creates the `group1_finance` catalog with `bronze`, `silver`, and `gold` schemas. Drops and recreates the catalog if it already exists (cascade). No source data is touched.

3. **Run `01_bronze`** — copies all 8 TPC-H tables from `samples.tpch` into `group1_finance.bronze` as-is using `CREATE OR REPLACE TABLE`. Verifies that source and bronze row counts match.

   Expected row counts:

   | Table | Rows |
   |---|---|
   | `customer` | 750,000 |
   | `lineitem` | 29,999,795 |
   | `nation` | 25 |
   | `orders` | 7,500,000 |
   | `part` | 1,000,000 |
   | `partsupp` | 4,000,000 |
   | `region` | 5 |
   | `supplier` | 50,000 |

4. **Run `02_silver_profiling`** (optional, read-only) — profiles bronze and checks that the silver DQ bounds still match the source. Asserts at the end: if the source has drifted, the bounds in `02_silver` must be revisited before proceeding.

5. **Run `02_silver`** — transforms bronze into 3NF with enforced data quality. Tables load parent-first (region → nation → supplier/customer → … → lineitem). Rows that fail a rule go to `silver.quarantine` with the list of broken rules and the original bronze row as JSON. After loading, the notebook asserts primary-key uniqueness, foreign-key integrity, and prints a quarantine summary. An append-only `silver.load_log` records `bronze = silver + quarantined + exact duplicates` per table per run.

6. **Run `03_gold`** — builds the Gold star schema (`dim_customer`, `dim_supplier`, `dim_part`, `dim_date`, `fact_lineitem`) and the views answering the four Finance business questions. Pre-flight checks assert that all required silver tables exist. After building, the notebook verifies fact-row count, grain uniqueness, dimension key uniqueness, and that Gold net revenue reconciles with Silver (tolerance $0.01).

### Portability

All notebooks accept widget parameters for the catalog and schema names, so the pipeline can run in another workspace without code changes:

| Notebook | Widgets |
|---|---|
| `02_silver_profiling` | `catalog`, `bronze_schema` |
| `02_silver` | `catalog`, `bronze_schema`, `silver_schema` |
| `03_gold` | `catalog`, `silver_schema`, `gold_schema` |

Override them from the notebook widget bar or pass them from a Lakeflow Job. Defaults are `group1_finance` / `bronze` / `silver` / `gold`.

## Silver Layer

The 8 TPC-H entities in 3NF with explicit types, primary keys, foreign keys and CHECK constraints. ER diagram: [`docs/silver_er.md`](docs/silver_er.md).

### How data quality is enforced

Unity Catalog does not enforce every constraint type, so each rule is applied in up to three places:

| Constraint | Declared on table | Enforced by Delta | Enforced by pipeline |
|---|---|---|---|
| Types | ✓ | ✓ | `try_cast`; a value that does not cast is quarantined |
| `NOT NULL` | ✓ | ✓ | quarantined before write |
| `CHECK` | ✓ | ✓ | the same expression is quarantined before write |
| `PRIMARY KEY` | ✓ (informational) | ✗ | duplicate keys quarantined; uniqueness asserted after load |
| `FOREIGN KEY` | ✓ (informational) | ✗ | orphans quarantined; integrity asserted after load |

- **One spec per table** in `02_silver` generates both the DDL and the quarantine rules, so the two cannot drift apart.
- **Composite FK `lineitem (l_partkey, l_suppkey) → partsupp (ps_partkey, ps_suppkey)`.** It is declared as a two-column `FOREIGN KEY` in Unity Catalog, which is informational and shows up in the ER diagram. The pipeline enforces it with a left join on both columns against the already-cleaned `silver.partsupp`. A line item whose pair is missing gets `fk_lineitem_partsupp_orphan`, even if its part and supplier each exist on their own.
- **Quarantine, not drop.** Failing rows go to `silver.quarantine` with `source_table`, `record_key`, `failed_rules` (all rules the row broke) and the original bronze row as JSON.
- **Cascading.** Tables load parent-first and check FKs against the clean parent. A quarantined order therefore also quarantines its line items, and no orphan can reach silver.
- **Strict only where it matters.** `NOT NULL` covers keys, FKs, money, dates and finance-relevant flags. Names, addresses and comments stay nullable, because quarantining a customer over a missing phone number would cascade to all of its orders and distort revenue.
- **Customers with no orders survive.** Rules filter bad rows and never inner-join parents to children.
- **Audit.** `silver.load_log` (append-only) records `bronze = silver + quarantined + exact duplicates` per table per run.

### DQ bounds (derived in `02_silver_profiling`)

Every bound below was derived by profiling the bronze copy of `samples.tpch` in `02_silver_profiling` (sections 5–8). The profiling notebook computes observed min/max, full distinct-value histograms for rate columns, and categorical value lists, then asserts that the source still fits the bounds hard-coded in `02_silver`.

| Rule | Bound | How derived | Justification |
|---|---|---|---|
| `l_discount` | `0.00 ≤ x ≤ 0.10` | Full distinct-value histogram (§5). Observed min/max = 0.00/0.10; every 0.01 step in between is populated with a near-uniform count. | Matches TPC-H spec clause 4.2.3 (*random [0.00 .. 0.10]*). Discount is a rate; anything outside this range is a load defect, not a real deal. |
| `l_tax` | `0.00 ≤ x ≤ 0.08` | Same histogram method. Observed 0.00–0.08 in uniform 0.01 steps. | Matches spec (*random [0.00 .. 0.08]*). |
| `l_quantity` | `> 0` | Numeric range scan (§5). Zero non-positive values observed. | Finance rule. TPC-H generates quantity in [1 .. 50] (spec §4.2.3). A zero or negative quantity would distort revenue and AOV. |
| `l_extendedprice` | `> 0` | Numeric range scan. Zero non-positive values observed. | Finance rule. Extended price = quantity × retail price; both factors are positive, so the product must be positive. |
| `p_retailprice` | `> 0` | Numeric range scan. Zero non-positive values observed. | Finance rule. A zero or negative price would distort revenue and AOV. |
| `ps_supplycost` | `> 0` | Numeric range scan. Zero non-positive values observed. | Supply is never free; a zero cost would distort margin calculations. |
| `ps_availqty` | `≥ 0` | Numeric range scan. No negative values; zero is present and valid. | Stock can be zero (out of stock) but never negative. |
| `o_totalprice` | `> 0` | Numeric range scan. Zero non-positive values observed. | Every order has at least one positive line item, so the order total must be positive. |
| `c_acctbal`, `s_acctbal` | *no rule* | Numeric range scan. Negative values observed and valid. | TPC-H allows negative balances (spec range −999.99 … 9 999.99); they represent credit or debt and are legitimate. |
| `c_mktsegment` | `{AUTOMOBILE, BUILDING, FURNITURE, HOUSEHOLD, MACHINERY}` | Categorical domain scan (§6). Exactly 5 values, all populated. | Closed domain. A new value is quarantined for review. |
| `o_orderstatus` | `{F, O, P}` | Categorical domain scan. Exactly 3 values. | F = fulfilled, O = open, P = pending. Closed domain. |
| `l_returnflag` | `{A, N, R}` | Categorical domain scan. Exactly 3 values. | A = accepted, N = not returned, R = returned. Closed domain. |
| `l_linestatus` | `{F, O}` | Categorical domain scan. Exactly 2 values. | F = fulfilled, O = open. Closed domain. |

**Why these bounds and not looser ones.** Bounds are pinned to the observed domain (which equals the spec domain) rather than a loose business range such as `[0, 1]` for discount. The data is generated with a known domain, so anything outside it means the pipeline broke — not that a rare but legitimate value appeared. `02_silver_profiling` asserts that the source still fits these bounds on every run; if the assertion fails, the source has drifted and the bounds must be revisited before re-running silver.

### 3NF notes

TPC-H is already normalised: nation and region attributes live only in `nation`/`region`, and the non-key attributes of `partsupp`/`lineitem` depend on their whole composite key. Two derivable columns are kept on purpose:

- `o_totalprice` (≈ Σ line items) is the recorded order total. Reconciling it against its line items is the Finance validation rule.
- `l_extendedprice` (= quantity × retail price) is the price at the time of sale.

## Business Questions

1. Which 10 customers generate the most net revenue, and what share of total revenue do they account for?
2. Report 1997 revenue three ways — gross extended price, net of discount, and net of discount and tax. How far apart are the three figures, and which one should the business report? Defend the choice in one paragraph.
3. Which market segment has the highest average order value, and does that change year over year?
4. Which 5 suppliers generate the most revenue, and what is their average discount?

## Gold Layer

The Gold layer is a star schema built from the cleaned Silver tables in `03_gold`.

### Schema

| Table | Grain | Purpose |
|---|---|---|
| `fact_lineitem` | One row per `(order_key, line_number)` | Line-item fact with pre-computed revenue metrics |
| `dim_customer` | One row per customer | Customer attributes (name, segment, nation, balance) |
| `dim_supplier` | One row per supplier | Supplier attributes (name, nation, balance) |
| `dim_part` | One row per part | Part attributes (name, manufacturer, brand, type, size, container, retail price) |
| `dim_date` | One row per order date | Date dimension (year, quarter, month, day) |

The fact table joins `silver.lineitem` to `silver.orders` to attach the customer key and order date to each line item. It deliberately does **not** repeat `o_totalprice` on every line, because summing a repeated order-header total would double-count orders.

### Metric definitions

All financial metrics are pre-computed on `fact_lineitem` and used consistently across every business question:

| Metric | Formula | Description |
|---|---|---|
| `gross_revenue` | `extended_price` | Line-item extended price (= quantity × retail price at time of sale). No adjustments. |
| `discount_amount` | `extended_price × discount` | Monetary value of the discount applied to the line. |
| `net_revenue` | `extended_price × (1 − discount)` | Revenue after discount, excluding tax. **Project-wide reporting metric.** |
| `tax_amount` | `extended_price × (1 − discount) × tax` | Sales tax on the discounted amount. |
| `net_revenue_with_tax` | `extended_price × (1 − discount) × (1 + tax)` | Revenue after discount plus tax collected. |
| `AOV` | `SUM(net_revenue) / COUNT(DISTINCT order_key)` | Average Order Value — net revenue per distinct order. |
| `average_discount_pct` | `AVG(discount) × 100` | Arithmetic mean of line-item discount rates, as a percentage. |
| `customer_share_pct` | `customer_net_revenue / total_net_revenue × 100` | A customer's share of total net revenue. |

**Why net revenue (excluding tax) is the reporting metric.** Gross extended price ignores discounts, overstating revenue. Net of discount and tax mixes the company's sales with tax collected on behalf of the government — tax is not revenue. Net revenue (`extended_price × (1 − discount)`) reflects the actual amount the customer pays for goods after negotiated discounts, which is the figure Finance should report.

### Gold views

| View | Question |
|---|---|
| `vw_q1_top_customers` | Q1 — Top 10 customers by net revenue and their revenue share |
| `vw_q2_1997_revenue` | Q2 — 1997 revenue three ways (gross, net of discount, net of discount + tax) |
| `vw_q3_aov_by_segment_year` | Q3 — AOV by market segment and year |
| `vw_q3_yearly_winner` | Q3 — Highest-AOV segment per year |
| `vw_q4_top_suppliers` | Q4 — Top 5 suppliers by net revenue and their average discount |

### Cross-layer reconciliation

`03_gold` verifies that Gold net revenue reconciles with Silver: `SUM(net_revenue)` from `fact_lineitem` must equal `SUM(l_extendedprice * (1 - l_discount))` from `silver.lineitem` within $0.01. This ensures no rows were lost or doubled when building the fact table.

## Naming Convention

| Object | Pattern | Example |
|---|---|---|
| Catalog | `group1_finance` | — |
| Schema | `<layer>` | `bronze`, `silver`, `gold` |
| Table | `<catalog>.<schema>.<tpch_table>` | `group1_finance.bronze.customer` |

## Tech Stack

- **Platform:** Databricks
- **Language:** Python / SQL
- **Source:** `samples.tpch`