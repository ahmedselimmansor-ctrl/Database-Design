# 12 — OLAP, Warehousing and CDC

*Columnar storage, dimensional modelling, ELT pipelines, change data capture, and the lakehouse.*

[← back to the handbook](../README.md)

---

## 1. Why analytics needs a different system

```mermaid
flowchart TD
    Q["SELECT category, sum(revenue)<br/>FROM sales WHERE year = 2026<br/>GROUP BY category"] --> R["On the OLTP primary"]
    R --> R1["Reads ~800 GB of pages to touch<br/>3 columns of 400 M rows"]
    R --> R2["EVICTS the hot working set from<br/>the buffer pool — every user-facing<br/>query slows down for minutes afterwards"]
    R --> R3["Holds an MVCC snapshot for the<br/>duration ⇒ vacuum cannot clean up<br/>⇒ bloat"]
    R --> R4["On a replica: replay conflicts<br/>⇒ lag, or the query gets cancelled"]
    style R2 fill:#7d1128,color:#fff
    style R3 fill:#7d1128,color:#fff
```

The buffer-pool eviction is the effect people underestimate. One large analytical scan
does not merely take a long time — it degrades **every other query** for as long as it
takes the cache to refill.

---

## 2. Columnar storage

```mermaid
flowchart TD
    subgraph R["Row-oriented"]
        R1["[id=1|name=A|cat=X|price=10|qty=5]"]
        R2["[id=2|name=B|cat=Y|price=20|qty=3]"]
        RN["Reading price requires touching<br/>every byte of every row"]
    end
    subgraph C["Column-oriented"]
        C1["id:    1,2,3,4,5,..."]
        C2["name:  A,B,C,D,E,..."]
        C3["cat:   X,Y,X,X,Y,...  ← RLE: X x3, Y x2"]
        C4["price: 10,20,30,40,...  ← delta encoding"]
        CN["Read ONLY the columns referenced.<br/>Similar values are adjacent ⇒<br/>5-20x compression.<br/>Vectorised SIMD over packed arrays."]
    end
    style C fill:#4a044e,color:#fff
```

### 2.1 The encodings

| Encoding | Applies to | Effect |
|---|---|---|
| **Run-length (RLE)** | Sorted or low-cardinality columns | `X,X,X,X,Y,Y` → `(X,4),(Y,2)` |
| **Dictionary** | Repeated strings | Store integers, plus one dictionary |
| **Delta** | Sorted numerics, timestamps | Store differences, which are small |
| **Delta-of-delta** | Regular time intervals | Near-zero bytes for evenly-spaced series |
| **Bit-packing** | Small integer ranges | 3 bits instead of 32 |
| **Frame of reference** | Clustered numerics | Store an offset from a block minimum |

**Sort order determines compression.** A column store sorted by `(date, category)`
compresses those columns dramatically better than an unsorted one. Choosing the sort
key is the columnar equivalent of choosing a clustered index, and it is the highest-
impact schema decision in a warehouse.

### 2.2 Predicate pushdown and zone maps

```mermaid
flowchart LR
    Q["WHERE date = '2026-08-20'"] --> Z["Zone map / min-max per block:<br/>each 8,192-row block records<br/>min and max per column"]
    Z --> S["Blocks whose range cannot match<br/>are SKIPPED without decompression"]
    S --> R["A well-sorted table may read 0.1%<br/>of its blocks for a narrow predicate"]
    style R fill:#14532d,color:#fff
```

This is the same idea as BRIN in PostgreSQL, applied natively.

---

## 3. Dimensional modelling

```mermaid
erDiagram
    FACT_ORDER_LINE }o--|| DIM_DATE : ""
    FACT_ORDER_LINE }o--|| DIM_PRODUCT : ""
    FACT_ORDER_LINE }o--|| DIM_CUSTOMER : ""
    FACT_ORDER_LINE }o--|| DIM_CHANNEL : ""

    FACT_ORDER_LINE {
        bigint order_line_key PK
        int date_key FK
        int product_key FK
        int customer_key FK
        int channel_key FK
        int quantity
        numeric gross_amount
        numeric discount_amount
        numeric net_amount
        numeric cost_amount
    }
    DIM_PRODUCT {
        int product_key PK
        text sku
        text name
        text category
        text brand
        numeric list_price
        date valid_from
        date valid_to
        boolean is_current
    }
```

### 3.1 Grain — decide it first

The **grain** is what one row of the fact table represents. Getting it wrong is the
most expensive warehouse mistake, because everything downstream assumes it.

```
"One row per order line item"   ✓ atomic, flexible, aggregates to anything
"One row per order"             ✗ loses per-product detail permanently
"One row per customer per day"  ✗ already an aggregate; cannot drill down
```

**Model at the most atomic grain available.** Aggregates can always be derived;
detail that was never stored cannot be recovered.

### 3.2 Fact table types

| Type | Contains | Example |
|---|---|---|
| **Transaction** | One row per event | Each sale, each click |
| **Periodic snapshot** | State at regular intervals | Daily account balance |
| **Accumulating snapshot** | One row per process, updated as it progresses | An order with columns for placed/paid/shipped/delivered dates |
| **Factless** | Only keys — the event itself is the fact | Attendance, promotion eligibility |

### 3.3 Slowly changing dimensions

```mermaid
flowchart TD
    C["A product moves from<br/>'Electronics' to 'Home'"] --> T1["TYPE 1 — overwrite<br/>Historical sales now appear under 'Home'.<br/>History is rewritten."]
    C --> T2["TYPE 2 — new row + validity dates<br/>Old sales keep the old row's key ⇒<br/>they stay under 'Electronics'.<br/>Correct history. THE STANDARD."]
    C --> T3["TYPE 3 — a previous_category column<br/>Keeps exactly one prior state."]
    style T2 fill:#14532d,color:#fff
    style T1 fill:#3b0d0d,color:#fff
```

Type 2 requires a **surrogate dimension key** distinct from the natural key: the fact
row references `product_key = 8891` (a specific version), not `sku = 'W-1'`. That
indirection is precisely what preserves history.

---

## 4. ELT pipelines

```mermaid
flowchart LR
    S1["OLTP databases"] & S2["SaaS APIs"] & S3["Event streams"] & S4["Files"] --> E["EXTRACT<br/>Fivetran, Airbyte, custom, CDC"]
    E --> L["LOAD raw, unchanged<br/>→ the RAW / bronze layer"]
    L --> T1["TRANSFORM → staging / silver<br/>typed, deduplicated, cleaned"]
    T1 --> T2["TRANSFORM → marts / gold<br/>dimensional models, business logic"]
    T2 --> C["BI, dashboards, ML features,<br/>reverse ETL"]
    style L fill:#0b2545,color:#fff
    style T2 fill:#14532d,color:#fff
```

### 4.1 Why the raw layer is non-negotiable

Keeping the untransformed extract means a transformation bug is fixed by re-running
SQL, not by re-extracting from a source system that may have changed, rate-limited
you, or discarded the data. It is cheap (object storage) and it is the difference
between a two-hour fix and a two-week one.

### 4.2 Incremental models

```sql
-- dbt-style incremental model
{{ config(materialized='incremental', unique_key='order_line_key') }}

SELECT ...
FROM {{ ref('stg_order_lines') }}
{% if is_incremental() %}
  -- overlap the window: late-arriving data is real
  WHERE updated_at > (SELECT max(updated_at) - interval '3 days' FROM {{ this }})
{% endif %}
```

Two details that separate a working incremental model from a subtly wrong one: a
**lookback window** (source systems backdate records, and a strict `>` watermark
misses them permanently) and a **unique key** so the reprocessed overlap merges rather
than duplicates.

### 4.3 Testing the pipeline

| Test | Catches |
|---|---|
| Uniqueness on the primary key | Fan-out from a bad join — the most common transformation bug |
| Not-null on required columns | Upstream schema drift |
| Referential integrity to dimensions | Orphaned facts, missing dimension rows |
| Accepted values on enumerations | New source values nobody told you about |
| **Row count within an expected band** | A silently-failed extract |
| **Freshness** | A pipeline that stopped running |
| Reconciliation totals against the source | Systematic transformation errors |

The last three catch the failures that matter most, because they are the ones that
produce a dashboard that is *wrong* rather than *broken* — and a wrong dashboard is
trusted.

---

## 5. Change data capture

### 5.1 Log-based vs query-based

```mermaid
flowchart TD
    subgraph Q["Query-based"]
        Q1["SELECT * WHERE updated_at > :watermark"]
        Q2["✗ DELETEs are invisible<br/>✗ Multiple updates within one poll<br/>  interval collapse to the last one<br/>✗ Load on the source database<br/>✗ Requires a trustworthy updated_at<br/>✓ Works against any database"]
    end
    subgraph L["Log-based"]
        L1["Read the WAL / binlog / redo log"]
        L2["✓ Every INSERT, UPDATE and DELETE<br/>✓ Exact commit order<br/>✓ Before AND after images<br/>✓ Negligible source load<br/>✓ Sub-second latency<br/>− A replication slot to operate<br/>− Needs the right privileges"]
    end
    style L fill:#14532d,color:#fff
    style Q fill:#3b0d0d,color:#fff
```

### 5.2 A CDC pipeline

```mermaid
flowchart LR
    DB[("PostgreSQL<br/>wal_level = logical")] --> SLOT["Replication slot"]
    SLOT --> DBZ["Debezium"]
    DBZ --> K["Kafka — one topic per table"]
    K --> C1["Search indexer"]
    K --> C2["Cache invalidator"]
    K --> C3["Warehouse loader"]
    K --> C4["Downstream services"]
    K --> C5["Audit / compliance store"]
    style DBZ fill:#0b2545,color:#fff
```

### 5.3 The operational realities

| Concern | Detail |
|---|---|
| **Slot lag → disk full** | A stopped consumer accumulates WAL until the primary's disk fills and the **database goes down**. Set `max_slot_wal_keep_size` |
| **Snapshot then stream** | Initial load must be consistent with the stream's start LSN; Debezium handles this, hand-rolled connectors often do not |
| **Schema changes** | DDL is not replicated logically. Consumers must tolerate new and dropped columns — use a schema registry with compatibility rules |
| **Replica identity** | `UPDATE`/`DELETE` need a primary key, or `REPLICA IDENTITY FULL` (which doubles WAL volume) |
| **At-least-once** | Duplicates will occur. Downstream must be idempotent, keyed by primary key plus LSN |
| **Ordering** | Guaranteed per key if the key is the Kafka partition key. Not guaranteed across keys |
| **Backfill vs stream** | Reprocessing history means replaying from the topic's start, or re-snapshotting — design which one you rely on |

### 5.4 CDC beyond analytics

```mermaid
flowchart TD
    C["CDC unlocks"] --> C1["Cache invalidation ORDERED AFTER the commit<br/>— the only version that is actually correct"]
    C --> C2["The outbox pattern without polling"]
    C --> C3["Search index freshness in milliseconds"]
    C --> C4["Zero-downtime major-version upgrades"]
    C --> C5["Strangler-fig migrations —<br/>keep the old and new systems in sync"]
    C --> C6["Audit trails that cannot be bypassed,<br/>even by direct database access"]
    style C1 fill:#14532d,color:#fff
    style C6 fill:#14532d,color:#fff
```

C6 is worth emphasising: an audit derived from the WAL captures **every** change,
including ones made by a console session, a migration script, or an attacker with
database credentials. An application-level audit log captures only what the
application did.

---

## 6. Lakehouse

```mermaid
flowchart TD
    subgraph W["Data warehouse"]
        W1["Structured, schema-on-write<br/>Proprietary storage format<br/>Excellent SQL performance<br/>Expensive per TB"]
    end
    subgraph L["Data lake"]
        L1["Any format, schema-on-read<br/>Object storage — very cheap<br/>No transactions, no schema enforcement<br/>Degenerates into a 'data swamp'"]
    end
    subgraph LH["Lakehouse"]
        LH1["Open table formats on object storage:<br/>Delta Lake, Apache Iceberg, Hudi"]
        LH2["+ ACID transactions on object storage<br/>+ Schema evolution and enforcement<br/>+ TIME TRAVEL — query as of a version<br/>+ Multiple engines read the same tables<br/>+ Object-storage economics"]
    end
    W & L --> LH
    style LH fill:#14532d,color:#fff
```

The mechanism behind the ACID claim is a **metadata log**: the table format keeps a
transaction log of which data files constitute the table at each version. A write adds
new files and atomically appends a new log entry; readers see a consistent file list.
This gives snapshot isolation and time travel over immutable object-storage files
without any coordination server.

---

## 7. Checklist

```
□ Analytics does NOT run on the OLTP primary
□ Fact table grain decided explicitly, at the most atomic level available
□ Dimensions use surrogate keys; SCD type chosen per dimension
□ Sort/cluster key chosen for the actual query patterns (drives compression too)
□ Raw layer retained unchanged; transformations are re-runnable
□ Incremental models have a lookback window and a unique key
□ Pipeline tests: uniqueness, not-null, referential integrity, row-count band, freshness
□ CDC uses log-based capture, not polling
□ Every logically-replicated table has a primary key
□ max_slot_wal_keep_size set; slot lag alerted on
□ Consumers are idempotent and tolerate schema evolution
□ The rebuild path for every derived store is documented and TESTED
```

---

[← previous: NoSQL and polyglot persistence](11-nosql-and-polyglot-persistence.md) · [back to the handbook](../README.md) · [next: Migrations and operations →](13-migrations-and-operations.md)
