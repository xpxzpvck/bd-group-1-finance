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
└── notebooks/
    ├── 00_setup.ipynb    # Create catalog & schemas (run once)
    ├── 01_bronze.ipynb   # Copy TPC-H tables as-is into bronze
    ├── 02_silver.ipynb   # 3NF + data quality
    └── 03_gold.ipynb     # Finance business questions
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

4. **Run `02_silver`** — transforms bronze into 3NF with enforced data quality and validation.

5. **Run `03_gold`** — builds gold-layer tables answering the Finance business questions.

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