# 08 — Replication

*Physical vs logical, synchronicity, lag, failover, split brain, and topologies.*

[← back to the handbook](../README.md)

---

## 1. Physical vs logical

```mermaid
flowchart TD
    subgraph P["Physical / streaming"]
        P1["Ships raw WAL records — byte-level<br/>page changes"]
        P2["+ Exact, low overhead, low latency<br/>+ Replica is byte-identical<br/>− Same major version required<br/>− Whole cluster only, no filtering<br/>− Replica is read-only, cannot have<br/>  its own indexes or extra tables"]
    end
    subgraph L["Logical"]
        L1["Decodes WAL into row-level<br/>INSERT/UPDATE/DELETE events"]
        L2["+ Cross-version, cross-platform<br/>+ Per-table / per-row filtering<br/>+ Target can be a DIFFERENT system<br/>+ Replica is writable, can add indexes<br/>− Higher overhead<br/>− Needs a REPLICA IDENTITY<br/>− DDL is NOT replicated"]
    end
    style P fill:#0b2545,color:#fff
    style L fill:#134e4a,color:#fff
```

| Use case | Choose |
|---|---|
| High-availability standby | Physical |
| Read replicas serving identical queries | Physical |
| Major-version upgrade with minimal downtime | **Logical** |
| Feeding a search index, warehouse, or cache invalidator | **Logical (CDC)** |
| Replicating a subset of tables to another team | **Logical** |
| Consolidating several databases into one | **Logical** |

### 1.1 Replica identity — the logical replication gotcha

For `UPDATE` and `DELETE`, logical replication must identify *which row* changed on
the subscriber. By default it uses the primary key.

```sql
-- a table with no primary key cannot be logically replicated for UPDATE/DELETE
ALTER TABLE events REPLICA IDENTITY FULL;   -- ships the ENTIRE old row as the key
-- correct, but expensive: doubles the WAL volume for that table
-- and makes the subscriber do a full-row comparison per change
```

**Every table you intend to replicate logically needs a primary key.** Discovering
this after a migration has started is a common and avoidable delay.

---

## 2. Synchronicity in depth

```mermaid
flowchart TD
    C["COMMIT on the primary"] --> M{"synchronous_commit"}
    M -->|"off"| O["Ack before local fsync.<br/>Loses recent commits on crash."]
    M -->|"local"| L["Ack after local fsync only.<br/>Loses committed data if the<br/>PRIMARY is lost permanently."]
    M -->|"remote_write"| RW["Ack after the standby has WRITTEN<br/>(OS cache, not fsynced).<br/>Survives primary loss; not both<br/>machines losing power together."]
    M -->|"on"| ON["Ack after the standby has FSYNCED.<br/>No loss if either machine survives."]
    M -->|"remote_apply"| RA["Ack after the standby has APPLIED.<br/>Reads on that standby are guaranteed<br/>to see the write. Slowest."]
    style ON fill:#14532d,color:#fff
    style RA fill:#0d3b66,color:#fff
```

### 2.1 Quorum-based synchronous replication

```sql
-- ✗ names one standby: if s1 is down or slow, ALL writes stall
synchronous_standby_names = 's1'

-- ✓ any one of three: a single sick standby cannot stop writes
synchronous_standby_names = 'ANY 1 (s1, s2, s3)'

-- stronger: any two of three
synchronous_standby_names = 'ANY 2 (s1, s2, s3)'
```

The `ANY k (...)` form is the configuration that gives durability without turning a
standby's problem into a primary outage. Naming a specific standby means that
standby's availability is now the primary's availability.

### 2.2 `remote_apply` and read-your-writes

`remote_apply` is the only setting that makes a read on the standby *guaranteed* to
see the write that just returned. It costs an extra round trip plus apply time on
every commit, so it is normally applied per-transaction rather than globally:

```sql
BEGIN;
SET LOCAL synchronous_commit = remote_apply;
UPDATE profiles SET bio = $1 WHERE user_id = $2;
COMMIT;   -- when this returns, the standby has applied it
```

---

## 3. Replication lag

### 3.1 Measuring it correctly

```sql
-- on the primary: lag per standby, in bytes, at each stage
SELECT application_name, state, sync_state,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn,   write_lsn)) AS write_lag,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn,   flush_lsn)) AS flush_lag,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn,   replay_lsn)) AS replay_lag,
       write_lag AS write_time, flush_lag AS flush_time, replay_lag AS replay_time
FROM pg_stat_replication;

-- on the standby: how stale is my data, in seconds?
SELECT CASE WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn()
            THEN 0
            ELSE extract(epoch FROM now() - pg_last_xact_replay_timestamp())
       END AS lag_seconds;
```

```mermaid
flowchart LR
    S["sent"] --> W["written to the standby's OS"] --> F["flushed to the standby's disk"] --> R["REPLAYED — visible to queries"]
    N["Only REPLAY lag affects what readers see.<br/>Flush lag is what determines data loss on failover.<br/>They are different numbers and both matter."]
    style R fill:#14532d,color:#fff
    style N fill:#422006,color:#fff
```

### 3.2 Causes and fixes

| Cause | Fix |
|---|---|
| **Single-threaded apply** vs parallel writes on the primary | `max_parallel_apply_workers` (logical); for physical, reduce write burstiness |
| **A long query on the standby** blocking replay | `hot_standby_feedback = on` (costs bloat on the primary) or `max_standby_streaming_delay` |
| One enormous transaction | Batch large writes into chunks on the primary |
| Weaker standby hardware | Match the primary's I/O capability |
| Network saturation | Enable WAL compression; check bandwidth |
| Standby doing heavy analytics | Dedicate a standby to it, and accept its lag |

The **`hot_standby_feedback` trade-off** is worth understanding: with it on, the
standby tells the primary which old row versions it still needs, so the primary
delays vacuuming them and long standby queries are never cancelled. The cost is bloat
on the *primary* driven by query behaviour on the *standby* — a coupling that
surprises people during an incident.

### 3.3 Query conflicts on hot standbys

```mermaid
flowchart TD
    Q["A query runs on the standby"] --> R["WAL arrives that removes a row<br/>version the query still needs"]
    R --> C{"max_standby_streaming_delay"}
    C -->|"delay not exceeded"| W["Replay PAUSES — the standby<br/>falls further behind"]
    C -->|"delay exceeded"| K["Query CANCELLED:<br/>ERROR: canceling statement due to<br/>conflict with recovery"]
    K --> F["Fix: hot_standby_feedback = on,<br/>or a larger delay, or route long<br/>queries to a dedicated standby"]
    style K fill:#7d1128,color:#fff
```

---

## 4. Failover

```mermaid
sequenceDiagram
    participant M as Monitor / orchestrator
    participant P as Primary
    participant S1 as Standby 1
    participant S2 as Standby 2
    participant A as Applications

    M->>P: health check
    P--xM: no response
    M->>M: wait for N consecutive failures<br/>(avoid flapping on a transient blip)
    M->>S1: what is your replay LSN?
    M->>S2: what is your replay LSN?
    S1-->>M: 0/7A3F1200
    S2-->>M: 0/7A3F0100   (behind)
    M->>M: choose S1 — furthest ahead
    M->>P: FENCE — STONITH, revoke storage,<br/>or block the network
    Note over P: the old primary must NOT be able<br/>to accept writes if it returns
    M->>S1: promote
    S1->>S1: end recovery, become writable
    M->>S2: follow the new primary
    M->>A: repoint (DNS / proxy / VIP / service discovery)
    A->>S1: writes resume
```

### 4.1 The failure modes

```mermaid
flowchart TD
    F["Failover hazards"] --> F1["SPLIT BRAIN — both nodes accept writes.<br/>Requires fencing + a majority quorum."]
    F --> F2["Data loss — an async standby promoted<br/>without the primary's last commits."]
    F --> F3["ID COLLISION — the promoted node's<br/>sequence resumes where IT left off and<br/>reissues ids the old primary already gave out."]
    F --> F4["Flapping — a transient network blip<br/>triggers a needless failover."]
    F --> F5["The applications never repoint,<br/>because the connection string was hard-coded."]
    style F1 fill:#7d1128,color:#fff
    style F3 fill:#7d1128,color:#fff
```

**F3 is the quiet one.** After promotion, `nextval()` on the new primary continues
from the last value *it* replicated. Any ids the old primary issued but did not
replicate get handed out a second time — to different rows. Those duplicates then
propagate into caches, search indexes, analytics and third-party systems, and the
corruption is discovered weeks later. It is the strongest practical argument for
UUIDv7 or Snowflake identifiers in any system that will ever fail over.

### 4.2 Preventing split brain

```mermaid
flowchart TD
    S["Two-node cluster, network partition"] --> P["Each node sees the other as dead"]
    P --> B["Both promote. Split brain."]
    S2["Three-node cluster with a witness"] --> Q["Only the side holding a MAJORITY<br/>may promote"]
    Q --> OK["The isolated side cannot form a majority<br/>and refuses to promote. ✓"]
    style B fill:#7d1128,color:#fff
    style OK fill:#14532d,color:#fff
```

Fencing mechanisms, strongest first: **STONITH** (power off the old node), storage
fencing (revoke its access to the volume), network fencing (drop its traffic at the
switch or security group), and — weakest — a cooperative shutdown, which does nothing
if the node is unreachable rather than misbehaving.

---

## 5. Topologies

```mermaid
flowchart TD
    subgraph A["Single primary + cascading standbys"]
        A1["Primary"] --> A2["Standby 1"] --> A3["Standby 2"] & A4["Standby 3"]
        AN["Reduces load on the primary<br/>when there are many replicas.<br/>Standby 1 becomes a dependency."]
    end
    subgraph B["Multi-primary / multi-leader"]
        B1["Primary EU"] <--> B2["Primary US"]
        BN["Write locality per region.<br/>CONFLICTS ARE INEVITABLE —<br/>needs LWW, CRDTs, or app resolution.<br/>Auto-increment collisions must be<br/>avoided by offset or UUID."]
    end
    subgraph C["Chain replication"]
        C1["Head — accepts writes"] --> C2["Middle"] --> C3["Tail — serves reads"]
        CN["Strong consistency with high throughput:<br/>the tail has, by construction, everything<br/>that was acknowledged.<br/>Higher write latency; a node failure<br/>requires chain reconfiguration."]
    end
    style B fill:#3b0d0d,color:#fff
```

### 5.1 Multi-primary conflict resolution

| Strategy | Behaviour | Risk |
|---|---|---|
| **Last-write-wins** | Higher timestamp survives | **Silent data loss**; clock skew decides business outcomes |
| First-write-wins | Earlier timestamp survives | Same, inverted |
| Site priority | A designated region always wins | Predictable, arbitrary |
| Merge function | Application-defined | Correct, most work |
| **Avoid conflicts entirely** | Partition writes so each row has exactly one writing region | **Best answer where the domain allows** |

The last row is the one to reach for first. If EU users' rows are only ever written in
EU and US users' rows only in US, there is no conflict to resolve — you have
multi-primary write locality with single-primary semantics per row.

---

## 6. Using replicas correctly

```mermaid
flowchart TD
    R["Read routing"] --> S1["Route to a replica ONLY when<br/>staleness is acceptable"]
    R --> S2["Route to the primary for:<br/>reads within a write transaction,<br/>read-after-write in the same session,<br/>anything driving a business decision"]
    R --> S3["EJECT replicas whose lag exceeds<br/>a threshold from the read pool"]
    R --> S4["Track WHY each read went where —<br/>otherwise the split becomes untraceable"]
    style S3 fill:#14532d,color:#fff
```

### 6.1 The LSN token pattern

```sql
-- after writing on the primary
SELECT pg_current_wal_lsn();     -- e.g. 0/7A3F1200 → return to the client/session

-- before reading on a replica
SELECT pg_last_wal_replay_lsn() >= '0/7A3F1200'::pg_lsn;
-- true  → this replica is current enough, serve the read
-- false → wait briefly, try another replica, or fall back to the primary
```

This gives precise read-your-writes with full replica parallelism — far better than
"send everything to the primary for 5 seconds", which sacrifices most of the benefit
of having replicas at all.

---

## 7. Checklist

```
□ Replication mode chosen deliberately: physical for HA, logical for CDC/upgrades
□ Every logically-replicated table has a primary key
□ synchronous_standby_names uses ANY k (...) — never a single named standby
□ Replay lag AND flush lag both monitored, in bytes and seconds
□ Replicas beyond a lag threshold are ejected from the read pool
□ hot_standby_feedback decision made consciously (standby cancellations vs primary bloat)
□ Failover is automated, fenced, and requires a majority
□ Failover TESTED on a schedule, not just documented
□ External ids are UUIDv7/Snowflake, not bare sequences, if failover is possible
□ Applications repoint automatically (proxy, service discovery, not a hard-coded host)
□ Read routing rules are explicit and observable
□ Replication slots have max_slot_wal_keep_size set
```

---

[← previous: WAL, durability and recovery](07-wal-durability-and-recovery.md) · [back to the handbook](../README.md) · [next: Partitioning and sharding →](09-partitioning-and-sharding.md)
