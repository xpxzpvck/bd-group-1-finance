# Silver Layer — ER Diagram

Schema: `group1_finance.silver`. Built by [`notebooks/02_silver.ipynb`](../notebooks/02_silver.ipynb).

```mermaid
erDiagram
    REGION   ||--|{ NATION   : "n_regionkey"
    NATION   ||--o{ SUPPLIER : "s_nationkey"
    NATION   ||--o{ CUSTOMER : "c_nationkey"
    PART     ||--|{ PARTSUPP : "ps_partkey"
    SUPPLIER ||--o{ PARTSUPP : "ps_suppkey"
    CUSTOMER ||--o{ ORDERS   : "o_custkey"
    ORDERS   ||--|{ LINEITEM : "l_orderkey"
    PARTSUPP ||--o{ LINEITEM : "(l_partkey, l_suppkey)"

    REGION {
        bigint r_regionkey PK
        string r_name
        string r_comment
    }
    NATION {
        bigint n_nationkey PK
        string n_name
        bigint n_regionkey FK
        string n_comment
    }
    SUPPLIER {
        bigint  s_suppkey PK
        string  s_name
        string  s_address
        bigint  s_nationkey FK
        string  s_phone
        decimal s_acctbal "may be negative"
        string  s_comment
    }
    CUSTOMER {
        bigint  c_custkey PK
        string  c_name
        string  c_address
        bigint  c_nationkey FK
        string  c_phone
        decimal c_acctbal "may be negative"
        string  c_mktsegment "CHECK in 5 segments"
        string  c_comment
    }
    PART {
        bigint  p_partkey PK
        string  p_name
        string  p_mfgr
        string  p_brand
        string  p_type
        int     p_size
        string  p_container
        decimal p_retailprice "CHECK > 0"
        string  p_comment
    }
    PARTSUPP {
        bigint  ps_partkey PK, FK
        bigint  ps_suppkey PK, FK
        int     ps_availqty "CHECK >= 0"
        decimal ps_supplycost "CHECK > 0"
        string  ps_comment
    }
    ORDERS {
        bigint  o_orderkey PK
        bigint  o_custkey FK
        string  o_orderstatus "CHECK in F, O, P"
        decimal o_totalprice "CHECK > 0"
        date    o_orderdate
        string  o_orderpriority
        string  o_clerk
        int     o_shippriority
        string  o_comment
    }
    LINEITEM {
        bigint  l_orderkey PK, FK
        bigint  l_partkey FK "composite FK to partsupp"
        bigint  l_suppkey FK "composite FK to partsupp"
        int     l_linenumber PK "CHECK > 0"
        decimal l_quantity "CHECK > 0"
        decimal l_extendedprice "CHECK > 0"
        decimal l_discount "CHECK 0.00 to 0.10"
        decimal l_tax "CHECK 0.00 to 0.08"
        string  l_returnflag "CHECK in A, N, R"
        string  l_linestatus "CHECK in F, O"
        date    l_shipdate
        date    l_commitdate
        date    l_receiptdate
        string  l_shipinstruct
        string  l_shipmode
        string  l_comment
    }
```

All `decimal` columns are `DECIMAL(18,2)`. Keys, foreign keys, monetary columns, dates and finance-relevant flags are `NOT NULL`; descriptive text columns are nullable.

`lineitem` does **not** reference `part` and `supplier` separately: it references the `(ps_partkey, ps_suppkey)` pair in `partsupp`, so a line item can only use a supplier that actually offers that part. Valid part and supplier keys on their own do not satisfy it.

Rows that fail any rule are written to `silver.quarantine`, which is outside the model and has no relationships.

## Screenshot for the presentation

GitHub renders the diagram above. To get an image of it:

- **Databricks:** *Catalog Explorer → group1_finance → silver → lineitem → View relationships*. This draws the ER diagram from the PK/FK constraints that `02_silver` registers in Unity Catalog.
- **Mermaid:** paste the block above into <https://mermaid.live> and export it as PNG or SVG.
