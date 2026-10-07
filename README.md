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
| *Monitoring* | `group1_finance.monitoring` | Validation results, the mismatched-orders metric over time, alert views (written by `04_validation`) |

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
    ├── 03_gold.ipynb               # Finance business questions
    ├── 04_validation.ipynb         # Finance validation rules on every layer -> monitoring.*
    ├── 05_monitoring.ipynb         # Mismatched-orders trend, DQ scorecard, alert views
    └── 06_alert_demo.ipynb         # Demo: corrupt one order, the alert fires, Delta RESTORE
```

## How to Run

### Prerequisites

- A Databricks workspace with access to `samples.tpch`
- Permission to create Unity Catalog catalogs and schemas
- Serverless or interactive compute
- [uv](https://docs.astral.sh/uv/) installed (for local development)

### Steps

1. **Clone the repo** into your Databricks workspace (or import the notebooks manually).

2. **Run `00_setup`** — creates the `group1_finance` catalog with `bronze`, `silver`, and `gold` schemas.

3. **Run `01_bronze`** — copies all 8 TPC-H tables into `group1_finance.bronze` as-is and verifies row counts.

4. **Run `02_silver_profiling`** (optional, read-only) — profiles bronze and checks that the silver DQ bounds still match the source.

5. **Run `02_silver`** — transforms bronze into 3NF with enforced data quality; rows that fail a rule go to `silver.quarantine`.

6. **Run `03_gold`** — builds gold-layer tables answering the Finance business questions.

7. **Run `04_validation`** — runs every Finance validation rule on source, bronze, silver and gold and stores the results in `group1_finance.monitoring`. With `fail_on_error = true` (default) it raises when an error rule fails, so in a Job it stops the run.

8. **Run `05_monitoring`** — creates the monitoring views (dashboard sources) and charts, and prints the query for the Databricks SQL alert.

9. **Optional demo: run `06_alert_demo`** — corrupts one order in `silver.orders`, shows the alert firing, then restores the table with Delta time travel.

All notebooks take `catalog` and schema names as widgets (Job parameters), so nothing is hard-coded to this workspace. In a scheduled Job, run steps 3–7 and leave out `00_setup`: it drops the whole catalog, including the monitoring history the alert compares against.

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

| Rule | Bound | Justification |
|---|---|---|
| `l_discount` | `0.00 ≤ x ≤ 0.10` | Observed min/max are 0.00 and 0.10, every 0.01 step is populated with a near-uniform count, and this matches the TPC-H spec (§4.2.3, *random [0.00 .. 0.10]*). Outside this range is a load defect, not a real deal. |
| `l_tax` | `0.00 ≤ x ≤ 0.08` | Observed 0.00–0.08 in uniform 0.01 steps; matches the spec (*random [0.00 .. 0.08]*). |
| `l_quantity`, `l_extendedprice`, `p_retailprice` | `> 0` | Finance rule. A zero or negative quantity or price would distort revenue and AOV. |
| `ps_supplycost` / `ps_availqty` | `> 0` / `≥ 0` | Supply is never free; stock can be zero but never negative. |
| `o_totalprice` | `> 0` | Every order has at least one positive line. |
| `c_acctbal`, `s_acctbal` | none | Negative balances are valid (spec range −999.99 … 9 999.99). |
| `c_mktsegment`, `o_orderstatus`, `l_returnflag`, `l_linestatus` | profiled value lists | Closed, small domains. A new value is quarantined for review. |

Bounds are pinned to the observed domain (which equals the spec domain) rather than a loose business range such as `[0, 1]`. The data is generated with a known domain, so anything outside it means the pipeline broke. `02_silver_profiling` asserts that the source still fits these bounds.

### 3NF notes

TPC-H is already normalised: nation and region attributes live only in `nation`/`region`, and the non-key attributes of `partsupp`/`lineitem` depend on their whole composite key. Two derivable columns are kept on purpose:

- `o_totalprice` (≈ Σ line items) is the recorded order total. Reconciling it against its line items is the Finance validation rule.
- `l_extendedprice` (= quantity × retail price) is the price at the time of sale.

## Metric definitions

One written definition per headline number. Every query in `03_gold` and `04_validation` uses these.

| Metric | Definition | Used in |
|---|---|---|
| Gross revenue | `Σ l_extendedprice` (list price × quantity, before discount) | Q2 |
| **Net revenue** — *the* revenue | `Σ l_extendedprice × (1 − l_discount)` | Q1, Q2, Q3, Q4, AOV, reconciliation V4 |
| Net revenue incl. tax | `Σ l_extendedprice × (1 − l_discount) × (1 + l_tax)` | Q2 |
| Billed order total | Per line, in whole cents: truncate after the discount, then again after the tax; summed over the order's lines | Order-total rule V1 |
| AOV (average order value) | `Σ net revenue / number of distinct orders` | Q3 |
| Mismatched order | An order whose `o_totalprice` differs from its billed order total by more than **$0.01**, or that has no line items | V1, monitoring, alert |
| Year | Calendar year of `o_orderdate` | Q2, Q3 |

## Validation

`04_validation` checks the four Finance rule groups on **every layer**: the source (`samples.tpch`), bronze, silver and gold. A rule on silver or gold is an **error** (the business reads those layers, so a failure fails the run). The same rule on bronze is a **warning**: raw data may be dirty because silver quarantines bad rows, but we still want to see it. Results go to `monitoring.dq_results`, one row per rule × layer × run.

| ID | Rule | Layers |
|---|---|---|
| V1.1 | Every order total matches the sum of its line items, fixed tolerance **$0.01** | bronze (warn), silver |
| V1.2 | Every order has at least one line item | bronze (warn), silver |
| V1.3 | *Info:* the same check with the naive formula, to show the rounding effect | silver |
| V2.1 / V2.2 | `l_discount ∈ [0.00, 0.10]`, `l_tax ∈ [0.00, 0.08]` (`NULL` counts as a violation) | bronze (warn), silver, gold |
| V2.3 | The `CHECK` constraints registered on `silver.lineitem` use the same bounds | silver (warn) |
| V3.1–V3.3 | Quantity, extended price and retail price are strictly positive | bronze (warn), silver, gold |
| V4.1 | Source = bronze: row counts, gross and net revenue, `Σ o_totalprice` | bronze |
| V4.2 | Bronze = silver + quarantine (+ exact duplicates dropped) | silver |
| V4.3 | Silver = gold: line items, gross and net revenue | gold |
| V4.4 | Silver = gold net revenue **per order year** | gold |
| V4.5 | In every gold row: `net = gross − discount_amount`, `net_with_tax = net + tax_amount` | gold |

### Order totals: why the rule rounds per line

The obvious check, `o_totalprice = Σ l_extendedprice × (1 − l_discount) × (1 + l_tax)`, flags **6,651,236 of 7,500,000 orders (88.7%)** as mismatched by more than $0.01. Every difference is small (at most $0.12) and always in the same direction. That is not bad data. The TPC-H generator (`dbgen`, `build.c`) computes the order total in **integer cents and truncates every line twice**, first after the discount and again after the tax:

```
line_cents   = ((extendedprice_cents × (100 − discount%)) DIV 100) × (100 + tax%) DIV 100
o_totalprice = Σ line_cents / 100
```

This is ordinary invoicing: each line is billed in whole cents. Rule V1.1 reproduces that billing rule and keeps the required **$0.01** tolerance. With it, **0 orders** mismatch: the expected difference is exactly $0.00. We matched the rounding rule instead of widening the tolerance, so an order that is off by more than one cent is a real defect. V1.3 keeps reporting the naive figure so the size of the effect stays visible.

Orders are left-joined to their line items, so an order without lines is not lost in a join: it counts as a mismatch (V1.1) and on its own (V1.2). Because silver quarantines a bad line but keeps its order, an order that lost a line in silver fails V1.1. This is how Finance learns that an invoice is incomplete.

### Discount and tax bounds

The bounds come from profiling the source (see *DQ bounds* in the Silver section): observed min/max 0.00–0.10 and 0.00–0.08, every 0.01 step populated (11 and 9 distinct values), matching the TPC-H specification. `04_validation` stores the observed min, max and distinct count with every run, so a change in the source domain is visible in the history even while the rule still passes. V2.3 raises a warning if someone changes the bounds in `02_silver` without changing them here.

### Cross-layer reconciliation

The same measures are computed independently in each layer, and gold uses its stored `net_revenue` column rather than recomputing it. Row counts must match exactly and money within $0.01:

```
source = bronze = silver + quarantine (+ dropped exact duplicates) ;  silver = gold (in total and per year)
```

Quarantined amounts are read from the original bronze row kept as JSON in `silver.quarantine.record`, so every cent that left silver is accounted for. The figures per layer are stored in `monitoring.layer_reconciliation`.

## Monitoring and alerting

| Table / view (`monitoring.*`) | Content |
|---|---|
| `metric_mismatched_orders` | **The monitored metric**: mismatched orders per run, how many are new or resolved, alert flag |
| `mismatched_orders_latest` / `_log` | The offending orders (current run / every run) with the reason |
| `dq_results`, `vw_dq_current_status` | Every rule result / latest status of each rule × layer |
| `layer_reconciliation`, `vw_layer_reconciliation_latest` | Money and row counts per layer |
| `vw_mismatched_orders_trend`, `vw_mismatched_orders_by_month` | Metric over pipeline runs and over order months (dashboard sources) |
| `vw_alert_new_mismatches` | Latest run only: source of the SQL alert |

**Alert: a new mismatch appears.** A run raises the alert when it finds a mismatched order that was not mismatched in the previous run (`new_mismatched_orders > 0`). A mismatch that persists keeps rule V1.1 failing on every run but does not alert again. Two mechanisms:

- **Job gate:** `04_validation` raises when an error rule fails, so the Job fails, downstream tasks stop and the Job's failure e-mail goes out.
- **Databricks SQL alert** on `vw_alert_new_mismatches` with condition `new_mismatched_orders > 0`. Setup steps are in `05_monitoring`.

**Demo (`06_alert_demo`):** adds $100 to one order's `o_totalprice` in silver. That still passes every `CHECK` constraint, so only the reconciliation can catch it. The notebook runs the order-total rules (alert fires, the order is listed with a $100.00 difference), restores the table with `RESTORE TABLE … TO VERSION AS OF`, and runs them again (mismatch resolved). The runs are labelled `demo` in the history.

## Business Questions

1. Which 10 customers generate the most net revenue, and what share of total revenue do they account for?
2. Report 1997 revenue three ways — gross extended price, net of discount, and net of discount and tax. How far apart are the three figures, and which one should the business report? Defend the choice in one paragraph.
3. Which market segment has the highest average order value, and does that change year over year?
4. Which 5 suppliers generate the most revenue, and what is their average discount?

### Q2 — which revenue figure to report

| 1997 (by order date) | Amount | vs. previous |
|---|---:|---:|
| Gross extended price | $173,945,163,744.15 | |
| Net of discount | **$165,246,449,758.48** | −5.0% (−$8.70 B of discounts) |
| Net of discount and tax | $171,861,420,506.33 | +4.0% (+$6.61 B of tax) |

The business should report **net of discount, excluding tax: $165.25 B for 1997**. Gross extended price is list price × quantity. It contains $8.70 B (5.0%) of discounts the company never received, so it overstates revenue. Adding tax does not fix that. The $6.61 B of tax is collected on behalf of the tax authorities and passed on to them, so it is a liability, not income. Revenue standards (IFRS 15) measure revenue net of discounts and exclude amounts collected for third parties, such as sales taxes. The tax-inclusive figure still has a use, but for a different question: it is what customers were invoiced. It equals `Σ o_totalprice` for 1997 to within cent rounding ($38,924 on $171.86 B), so Finance uses it for receivables and for the order-total reconciliation, not as revenue. Net of discount is also the definition used everywhere else in the project (Q1, Q3, Q4, AOV and the cross-layer reconciliation), so every reported figure adds up to the same total. The three figures are close: net is 5.0% below gross, and tax-inclusive is 4.0% above net, ending 1.2% below gross. They are close enough to be confused, which is why the definition is fixed in writing above.

## Naming Convention

| Object | Pattern | Example |
|---|---|---|
| Catalog | `group1_finance` | — |
| Schema | `<layer>` | `bronze`, `silver`, `gold` |
| Table | `<catalog>.<schema>.<tpch_table>` | `group1_finance.bronze.customer` |
| Monitoring table | `<catalog>.monitoring.<name>` | `group1_finance.monitoring.dq_results` |
| Monitoring view | `<catalog>.monitoring.vw_<name>` | `group1_finance.monitoring.vw_alert_new_mismatches` |

## Tech Stack

- **Platform:** Databricks
- **Language:** Python / SQL
- **Source:** `samples.tpch`