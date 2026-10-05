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

## Business Questions

1. Which 10 customers generate the most net revenue, and what share of total revenue do they account for?
2. Report 1997 revenue three ways — gross extended price, net of discount, and net of discount and tax. How far apart are the three figures, and which one should the business report? Defend the choice in one paragraph.
3. Which market segment has the highest average order value, and does that change year over year?
4. Which 5 suppliers generate the most revenue, and what is their average discount?

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