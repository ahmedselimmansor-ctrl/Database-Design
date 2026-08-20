# 11 — NoSQL and Polyglot Persistence

*Each family's data model and access patterns, and when it genuinely beats relational.*

[← back to the handbook](../README.md)

---

## 1. The one idea that transfers

```mermaid
flowchart LR
    R["Relational modelling<br/>model entities → normalise →<br/>write any query"] --> Q["The PLANNER reorganises<br/>data at query time"]
    N["NoSQL modelling<br/>list access patterns → design keys →<br/>duplicate as needed"] --> P["YOU organise data at<br/>WRITE time, per access pattern"]
    style N fill:#134e4a,color:#fff
```

There is no planner to rescue a query the schema was not designed for. In a
wide-column or key-value store, "can I run this query?" is answered by the key
structure, not by an optimiser — so the access-pattern list *is* the schema design.

---

## 2. Key-value

**Model:** opaque value addressed by key. **Operations:** `GET`, `SET`, `DEL`,
sometimes atomic numeric and structural operations.

| Use | Why |
|---|---|
| Session storage | Pure key lookup, TTL, tolerable loss |
| Rate limiting counters | Atomic `INCR`, expiry |
| Feature flags and configuration | Small, hot, read-constantly |
| Distributed locks | Atomic `SET NX PX` — **with fencing tokens** |
| Caching | The canonical use |
| Job queues | Redis lists/streams, or `SKIP LOCKED` in Postgres |
| Leaderboards | Sorted sets |

```mermaid
flowchart TD
    R["Redis specifics worth knowing"] --> R1["SINGLE-THREADED command execution<br/>⇒ one slow command (KEYS, a huge SORT,<br/>a 500 MB value) blocks EVERYTHING"]
    R --> R2["Persistence: RDB snapshots (point-in-time,<br/>fork-based) and AOF (append-only log).<br/>Neither makes it a system of record."]
    R --> R3["Cluster mode: 16,384 hash slots.<br/>Multi-key operations require all keys in<br/>ONE slot — use hash tags: user:{42}:name"]
    R --> R4["Eviction: set maxmemory AND a policy.<br/>The default (noeviction) turns a full<br/>cache into write errors."]
    style R1 fill:#7d1128,color:#fff
```

---

## 3. Document

**Model:** nested JSON-like documents; each may have a different shape.

```json
{
  "_id": "ord_8821",
  "customer": { "id": "c_42", "name": "Ahmed", "tier": "gold" },
  "items": [
    { "sku": "W-1", "name": "Widget", "qty": 2, "price": 10.00 },
    { "sku": "G-9", "name": "Gadget", "qty": 1, "price": 25.00 }
  ],
  "total": 45.00,
  "placed_at": "2026-08-20T10:14:00Z"
}
```

```mermaid
flowchart TD
    E["Embed vs reference"] --> EM["EMBED when:<br/>the child is always read with the parent,<br/>the child has no independent identity,<br/>the array is BOUNDED"]
    E --> RE["REFERENCE when:<br/>the child is queried independently,<br/>the child is shared by many parents,<br/>the array is UNBOUNDED,<br/>the child updates far more often<br/>than the parent"]
    EM --> W["Unbounded embedded arrays are the classic<br/>document-store failure: a document that grows<br/>forever hits the size limit, and every<br/>update rewrites the whole thing."]
    style W fill:#7d1128,color:#fff
    style EM fill:#14532d,color:#fff
```

> **Postgres `JSONB` covers most document use cases**, with GIN indexing, real
> constraints on the structured columns, and full SQL alongside. The genuine reasons
> to prefer a dedicated document store are its horizontal scaling model and its
> ecosystem — not the data model itself.

---

## 4. Wide-column

**Model:** `partition key → sorted rows within the partition`. The most
misunderstood family, and the most instructive.

```sql
CREATE TABLE messages_by_conversation (
    conversation_id uuid,
    bucket          text,        -- e.g. '2026-08' — bounds the partition size
    sent_at         timestamp,
    message_id      timeuuid,
    sender_id       uuid,
    body            text,
    PRIMARY KEY ((conversation_id, bucket), sent_at, message_id)
) WITH CLUSTERING ORDER BY (sent_at DESC);
```

```mermaid
flowchart TD
    PK["((conversation_id, bucket))<br/>PARTITION KEY"] --> N["hashed → determines the NODE.<br/>All rows with this key are stored<br/>together, contiguously, sorted."]
    CK["sent_at DESC, message_id<br/>CLUSTERING KEYS"] --> O["Determines the ORDER on disk.<br/>Range scans within the partition<br/>are a single sequential read."]
    N & O --> G["'Last 50 messages in conversation X<br/>for August' = ONE node, ONE seek."]
    B["The bucket column exists SOLELY to bound<br/>partition size. Without it, a busy<br/>conversation becomes a multi-gigabyte<br/>partition on one node — the single most<br/>common Cassandra design failure."]
    style G fill:#14532d,color:#fff
    style B fill:#7d1128,color:#fff
```

### 4.1 Query-driven duplication

```
Access patterns:
  AP1  messages in a conversation, newest first  → messages_by_conversation
  AP2  messages sent by a user                   → messages_by_sender
  AP3  messages mentioning a user                → messages_by_mention

Three tables. The same message written three times.
Storage is cheap; a scatter-gather across every node is not.
```

Writing the same fact three times means **three places to update or delete**, which is
why wide-column stores suit append-mostly, immutable-ish data (events, messages,
metrics, logs) far better than mutable entity state.

### 4.2 Tunable consistency

```
W + R > RF  ⇒  a read overlaps a write ⇒ strong consistency

RF=3, W=QUORUM(2), R=QUORUM(2)  → 2+2 > 3 ✓  strong, tolerates one node down
RF=3, W=ONE,       R=ONE        → 1+1 < 3 ✗  eventual, fastest
RF=3, W=ALL,       R=ONE        → strong, but any node down blocks writes
```

Lightweight transactions (`IF NOT EXISTS`) use Paxos and are roughly an order of
magnitude slower than a normal write. They exist for the rare cases that need
linearisability — and reaching for them frequently is a sign the workload wanted a
different database.

---

## 5. Graph

**Model:** nodes and edges, both with properties. **Operation:** traversal.

```cypher
// friends-of-friends who work at the same company, excluding direct friends
MATCH (me:Person {id: $id})-[:FRIEND]->(f)-[:FRIEND]->(fof)
WHERE NOT (me)-[:FRIEND]->(fof) AND me <> fof
MATCH (fof)-[:WORKS_AT]->(c)<-[:WORKS_AT]-(me)
RETURN fof, count(*) AS mutual ORDER BY mutual DESC LIMIT 10
```

```mermaid
flowchart TD
    D["Depth of traversal"] --> D1["1-2 hops<br/>SQL joins are fine — often faster"]
    D --> D2["3-5 hops<br/>SQL recursive CTEs get awkward;<br/>graph databases start to win"]
    D --> D3["Variable / unbounded depth<br/>shortest path, connected components,<br/>pattern matching<br/>⇒ GRAPH, clearly"]
    N["Graph databases use INDEX-FREE ADJACENCY:<br/>each node holds direct pointers to its edges,<br/>so a hop is a pointer dereference rather than<br/>an index lookup. Traversal cost is proportional<br/>to the subgraph touched, NOT to total data size."]
    style D3 fill:#14532d,color:#fff
    style N fill:#0b2545,color:#fff
```

---

## 6. Time-series

**Model:** `(metric, tags, timestamp) → value`. Append-heavy, rarely updated,
queried by time range with aggregation, usually with a retention policy.

```mermaid
flowchart TD
    O["Why a specialised store"] --> O1["Delta-of-delta timestamp encoding<br/>and XOR float compression<br/>⇒ 10-20x smaller than row storage"]
    O --> O2["Automatic partitioning by time<br/>⇒ retention is a partition DROP"]
    O --> O3["Continuous aggregates / downsampling<br/>keep 1 s for a day, 1 m for a month,<br/>1 h for a year"]
    O --> O4["Time-bucketing and gap-filling<br/>as first-class query operations"]
    C["CARDINALITY is the killer:<br/>each distinct tag combination is a series.<br/>Adding user_id as a tag on a 1 M-user<br/>system creates 1 M series and will<br/>destroy any TSDB."]
    style C fill:#7d1128,color:#fff
```

TimescaleDB is worth singling out because it is a PostgreSQL extension: you get
hypertables, compression and continuous aggregates while keeping SQL, joins,
transactions and your existing tooling. For most teams that is a better trade than
adding a wholly separate system.

---

## 7. Vector

**Model:** high-dimensional embeddings with approximate nearest-neighbour search.

```sql
-- pgvector
CREATE TABLE documents (
  id        bigserial PRIMARY KEY,
  content   text,
  tenant_id bigint NOT NULL,
  embedding vector(1536)
);
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

SELECT id, content, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE tenant_id = $2                     -- filter first, then search
ORDER BY embedding <=> $1
LIMIT 10;
```

```mermaid
flowchart TD
    K["Things to get right"] --> K1["Recall is TUNABLE, not fixed.<br/>Measure it against exact search<br/>on a sample before choosing parameters."]
    K --> K2["FILTERED search is the hard part —<br/>an ANN index searches the whole space,<br/>so a restrictive filter can return<br/>very few results. Check your engine's<br/>pre-filter vs post-filter behaviour."]
    K --> K3["Normalise vectors if using cosine;<br/>use the matching distance operator"]
    K --> K4["Re-embedding on a model change means<br/>reprocessing EVERYTHING. Plan for it:<br/>version the embedding column."]
    style K2 fill:#7d1128,color:#fff
```

---

## 8. Polyglot persistence — the honest accounting

```mermaid
flowchart TD
    A["Adding a second store"] --> C1["A second backup and restore regime"]
    A --> C2["A second failure mode, monitoring<br/>setup, and on-call runbook"]
    A --> C3["A synchronisation path — and therefore<br/>a consistency boundary"]
    A --> C4["A second set of operational expertise<br/>the team must maintain"]
    A --> C5["A second security, access-control<br/>and compliance surface"]
    A --> C6["Queries that used to be one JOIN<br/>are now application-level joins"]
    B["The benefit must exceed ALL of that,<br/>measurably. It sometimes does."]
    style A fill:#7d1128,color:#fff
    style B fill:#0b2545,color:#fff
```

### 8.1 When it genuinely pays

| Store added | Justified when |
|---|---|
| **Redis** | Cache hit ratio or counter contention is a measured bottleneck |
| **Elasticsearch** | Relevance ranking, faceting and typo tolerance are product requirements — not just `LIKE` |
| **ClickHouse / warehouse** | Analytical queries are degrading OLTP latency |
| **Cassandra / Scylla** | Sustained write volume exceeds what a partitioned relational cluster can absorb |
| **Neo4j** | Traversals are routinely deeper than 3 hops and central to the product |
| **A vector store** | pgvector's scaling limits are actually reached, with measurements |

### 8.2 The synchronisation rule

```mermaid
flowchart LR
    S["ONE system of record"] -->|"CDC / outbox"| D1["Derived store 1"]
    S --> D2["Derived store 2"]
    S --> D3["Derived store 3"]
    N["Every derived store must be REBUILDABLE<br/>from the system of record.<br/>Design and TEST the rebuild path on day one —<br/>you will need it after the first bug."]
    style S fill:#14532d,color:#fff
    style N fill:#422006,color:#fff
```

Dual-writing from the application to two stores is the anti-pattern: there is no
ordering of two independent writes that survives a crash between them. Use the outbox
pattern or log-based CDC so there is exactly one commit and everything else is derived
from it.

---

## 9. Decision summary

| Requirement | Store |
|---|---|
| Transactions, joins, ad-hoc queries | Relational |
| Point lookups at extreme throughput | Key-value |
| Varying document shapes queried by field | Document (or Postgres JSONB) |
| Huge sustained writes, ordered reads per entity | Wide-column |
| Traversals deeper than 3 hops | Graph |
| Timestamped measurements with aggregation | Time-series (or TimescaleDB) |
| Semantic similarity search | Vector (or pgvector) |
| Full-text relevance ranking | Search engine (or Postgres FTS) |
| Analytical scans over billions of rows | Columnar warehouse |
| Large immutable files | Object storage |

**Default to PostgreSQL**, which covers a surprising number of these adequately, and
add a specialist store when a measurement — not an intuition — says you must.

---

[← previous: Distributed databases](10-distributed-databases.md) · [back to the handbook](../README.md) · [next: OLAP, warehousing and CDC →](12-olap-warehousing-and-cdc.md)
