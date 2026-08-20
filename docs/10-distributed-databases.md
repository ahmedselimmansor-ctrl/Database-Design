# 10 — Distributed Databases

*Distributed SQL architecture, Spanner and TrueTime, Calvin, quorums, and distributed deadlock.*

[← back to the handbook](../README.md)

---

## 1. The promise and the physics

Distributed SQL databases (Spanner, CockroachDB, TiDB, YugabyteDB) offer horizontal
scale **without** giving up SQL, secondary indexes, or transactions. The promise is
real. The physics is unchanged: a consensus round trip across regions costs 50–200 ms
and nothing can make it cheaper.

```mermaid
flowchart TD
    P["What distributed SQL removes"] --> P1["Manual shard routing in the app"]
    P --> P2["Cross-shard queries you must write yourself"]
    P --> P3["Loss of transactions across shards"]
    P --> P4["Resharding as a project"]
    P --> P5["Global uniqueness as an app concern"]

    N["What it does NOT remove"] --> N1["Network latency between regions"]
    N --> N2["The need to design for data LOCALITY"]
    N --> N3["Contention on hot rows — now worse,<br/>because the row is also replicated"]
    N --> N4["The operational complexity of a<br/>distributed system"]
    style P fill:#14532d,color:#fff
    style N fill:#7d1128,color:#fff
```

---

## 2. Architecture

```mermaid
flowchart TD
    subgraph L["Logical"]
        T["Table, sorted by primary key"]
        T --> R1["Range [a, f)"]
        T --> R2["Range [f, m)"]
        T --> R3["Range [m, z]"]
    end
    subgraph Ph["Physical — each range is its own Raft group"]
        R1 --> G1["leader: node 1<br/>followers: node 3, node 5"]
        R2 --> G2["leader: node 4<br/>followers: node 2, node 6"]
        R3 --> G3["leader: node 5<br/>followers: node 1, node 4"]
    end
    N["Ranges SPLIT when they grow past a size or load<br/>threshold, and MERGE when they shrink.<br/>Leaders are rebalanced across nodes for load.<br/>All of this happens online, automatically."]
    style N fill:#0b2545,color:#fff
```

### 2.1 A single-row write

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as Any node (gateway)
    participant L as Range leader
    participant F1 as Follower 1
    participant F2 as Follower 2

    C->>GW: UPDATE t SET x = 1 WHERE id = 'g7'
    GW->>GW: consult the range map: 'g7' → range [f,m), leader = node 4
    GW->>L: forward
    L->>L: append to the Raft log
    par replicate
        L->>F1: AppendEntries
        L->>F2: AppendEntries
    end
    F1-->>L: ack
    Note over L: majority (leader + F1) reached → COMMITTED
    L->>L: apply
    L-->>GW: ok
    GW-->>C: ok
    Note over C,F2: cost = 1 consensus RTT.<br/>Same AZ: ~1-3 ms. Cross-region: 50-200 ms.
```

### 2.2 A multi-range transaction

```mermaid
flowchart TD
    T["BEGIN; UPDATE a; UPDATE z; COMMIT"] --> R["Touches ranges A and Z<br/>— different Raft groups"]
    R --> P["2PC across the two groups"]
    P --> P1["A transaction record is written<br/>(itself consensus-replicated)"]
    P --> P2["Write intents placed on each key<br/>— provisional values plus a pointer<br/>to the transaction record"]
    P --> P3["COMMIT = one atomic flip of the<br/>transaction record's status"]
    P3 --> P4["Intents are resolved to real values<br/>asynchronously afterwards"]
    N["Because the transaction record is itself<br/>replicated by consensus, no single node<br/>failure can leave the transaction blocked —<br/>which is exactly 2PC's classic weakness,<br/>solved."]
    style N fill:#14532d,color:#fff
```

A reader encountering an unresolved write intent must find out the transaction's
status: it reads the transaction record, and if the writer has gone silent, it may
**abort the writer** and proceed. This is why an abandoned transaction in a
distributed SQL system does not block readers indefinitely the way an orphaned 2PC
participant does.

---

## 3. Time

### 3.1 The problem

To order transactions globally without coordinating every pair of them, you need
timestamps that respect causality. Machine clocks do not.

```mermaid
flowchart TD
    W["Naive wall-clock timestamps"] --> S["NTP skew of 10-100 ms is routine"]
    S --> B["Txn A commits at real time T on a fast clock;<br/>Txn B commits later on a slow clock and<br/>gets a SMALLER timestamp.<br/>A reader at that timestamp sees B but not A —<br/>a causality violation."]
    style B fill:#7d1128,color:#fff
```

### 3.2 TrueTime (Spanner)

```mermaid
flowchart TD
    TT["TT.now() returns an INTERVAL<br/>[earliest, latest], not a point"] --> E["ε = interval width,<br/>typically ~1-7 ms<br/>(GPS + atomic clocks per datacentre)"]
    E --> CW["COMMIT WAIT:<br/>after choosing commit timestamp s,<br/>wait until TT.now().earliest > s<br/>before releasing locks"]
    CW --> G["GUARANTEE: if txn A commits before txn B<br/>starts in real time, then<br/>timestamp(A) < timestamp(B).<br/>= EXTERNAL CONSISTENCY, globally."]
    G --> C["Cost: ~2ε of added commit latency.<br/>Benefit: consistent global snapshots<br/>with NO coordination between readers."]
    style G fill:#14532d,color:#fff
```

The insight worth carrying beyond Spanner: **expose uncertainty rather than hiding
it.** A clock that admits an error bound lets you wait it out; a clock that pretends
to be exact leaves you with silent anomalies.

### 3.3 Hybrid logical clocks

```
HLC = (physical_time, logical_counter)

on local event:      pt = max(local_clock, pt);  if unchanged, lc += 1 else lc = 0
on receiving msg m:  pt = max(pt, m.pt, local_clock)
                     lc = appropriate increment to stay strictly greater
```

```mermaid
flowchart LR
    H["HLC properties"] --> H1["Stays close to physical time<br/>⇒ timestamps are humanly meaningful<br/>and usable for TTL and retention"]
    H --> H2["Strictly respects causality<br/>⇒ if A happened-before B, HLC(A) < HLC(B)"]
    H --> H3["Needs no special hardware"]
    H --> H4["Requires bounded clock skew as an<br/>assumption; a node whose clock drifts<br/>too far must remove ITSELF from the cluster"]
    style H fill:#14532d,color:#fff
```

CockroachDB uses HLC with a `max_offset` setting (default 500 ms). Reads that fall
within the uncertainty interval trigger an **uncertainty restart** — the transaction
retries at a higher timestamp. This is invisible when clocks are well-synchronised and
becomes a visible performance problem when they are not, which is why NTP health is a
first-class metric on such clusters.

---

## 4. Consistency levels available

```mermaid
flowchart TD
    R["Read types"] --> S["Strongly consistent<br/>served by the leaseholder;<br/>sees all committed writes"]
    R --> F["Follower / stale read<br/>served by any replica at a<br/>timestamp slightly in the past"]
    R --> B["Bounded staleness<br/>'no more than 10 s old'"]
    R --> E["Exact staleness<br/>'as of exactly timestamp T'"]

    S --> SN["Correct always. May require a<br/>cross-region hop to the leader."]
    F --> FN["Local, fast, cheap.<br/>May miss very recent writes.<br/>The right default for dashboards,<br/>catalogues and analytics."]
    style S fill:#0b2545,color:#fff
    style F fill:#14532d,color:#fff
```

**Follower reads are the single largest performance lever** in a geo-distributed
deployment: they turn a 150 ms cross-region read into a 2 ms local one. The
application must be able to tolerate a few seconds of staleness for that query, which
most read paths can.

---

## 5. Locality is the design problem

```mermaid
flowchart TD
    L["Data locality controls"] --> L1["REGIONAL BY ROW<br/>each row lives in the region<br/>named in one of its columns"]
    L --> L2["REGIONAL BY TABLE<br/>the whole table is homed in one region"]
    L --> L3["GLOBAL tables<br/>fast reads everywhere,<br/>slow writes — for reference data"]
    L --> L4["Range co-location / interleaving<br/>put a parent and its children<br/>in the SAME range"]
    L4 --> N["A transaction touching one order and<br/>its items then hits ONE Raft group —<br/>one consensus round, not four."]
    style N fill:#14532d,color:#fff
```

A schema designed without locality in mind will be correct and slow. The specific
anti-pattern: a globally-replicated table on the write path of every transaction,
turning every write into a multi-region consensus round.

---

## 6. Calvin — the other approach

```mermaid
flowchart LR
    C["Calvin / deterministic databases<br/>(FaunaDB and relatives)"] --> S["A sequencing layer establishes a<br/>GLOBAL ORDER of transactions FIRST"]
    S --> D["Every replica then executes that<br/>order DETERMINISTICALLY"]
    D --> R["Same order + deterministic execution<br/>⇒ identical results everywhere<br/>⇒ NO commit protocol needed at all"]
    R --> T["+ No 2PC, no distributed deadlock<br/>+ Very high throughput<br/>− The full read/write set must be known<br/>  IN ADVANCE — interactive transactions<br/>  are hard<br/>− The sequencer is a global bottleneck"]
    style R fill:#14532d,color:#fff
```

The trade is stark and clarifying: Calvin removes the entire commit-protocol problem
by requiring transactions to be declared up front. Systems that need interactive,
multi-round-trip transactions (which is most application code) cannot use it directly.

---

## 7. Distributed deadlock

```mermaid
flowchart TD
    D["Detection across nodes is expensive:<br/>the wait-for graph is distributed,<br/>and any snapshot of it may be stale<br/>(a PHANTOM deadlock)"] --> A["Approaches"]
    A --> A1["Timeout — simple, may abort<br/>transactions that were merely slow"]
    A --> A2["Wound-wait — an OLDER transaction<br/>ABORTS a younger lock holder;<br/>a younger one waits. Used by Spanner."]
    A --> A3["Wait-die — an older transaction WAITS;<br/>a younger one aborts immediately"]
    A --> A4["Centralised detector — accurate,<br/>a bottleneck and a single point of failure"]
    A2 --> N["Both wound-wait and wait-die are<br/>DEADLOCK-FREE BY CONSTRUCTION,<br/>because they impose a total order on<br/>transactions by age. No cycle can form."]
    style N fill:#14532d,color:#fff
```

Both schemes guarantee that older transactions eventually complete — a transaction
that gets aborted keeps its original timestamp on retry, so it grows older and
eventually wins. That property is what prevents starvation.

---

## 8. Choosing

```mermaid
flowchart TD
    Q["Do I need a distributed database?"] --> A{"Does the data fit one node?<br/>(2 TB RAM, 100 TB NVMe is one node)"}
    A -->|"yes"| B{"Does the write volume fit one node?<br/>(tens of thousands of TPS)"}
    B -->|"yes"| C{"Is single-region availability enough?"}
    C -->|"yes"| PG["PostgreSQL + replicas + partitioning.<br/>Simpler, cheaper, faster per query."]
    C -->|"no"| DSQL
    B -->|"no"| DSQL
    A -->|"no"| DSQL["Distributed SQL, or manual sharding"]
    DSQL --> D{"Team size and operational maturity?"}
    D -->|"small"| M["Managed service — Spanner,<br/>CockroachDB Cloud, TiDB Cloud"]
    D -->|"large, with a platform team"| S["Self-hosted is viable"]
    style PG fill:#14532d,color:#fff
```

| Choose distributed SQL when | Stay single-node when |
|---|---|
| Data genuinely exceeds one machine | It fits — and it probably fits |
| You need multi-region writes with transactions | Single region is acceptable |
| You need zero-downtime elastic scaling | Scheduled maintenance windows are acceptable |
| You would otherwise build sharding yourself | You have not yet exhausted indexes, pooling, caching, replicas, partitioning |
| Survival of a whole region is a requirement | An RTO measured in minutes is acceptable |

> **The honest default in 2026:** a well-tuned single PostgreSQL primary with replicas
> and partitioning serves the overwhelming majority of applications, at lower latency
> per query and a fraction of the operational cost. Reach for distributed SQL when you
> have a measured reason, not an anticipated one.

---

[← previous: Partitioning and sharding](09-partitioning-and-sharding.md) · [back to the handbook](../README.md) · [next: NoSQL and polyglot persistence →](11-nosql-and-polyglot-persistence.md)
