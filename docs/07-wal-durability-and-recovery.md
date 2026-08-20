# 07 — WAL, Durability and Recovery

*Log records, checkpoints, ARIES in detail, the fsync stack, and what the WAL is really for.*

[← back to the handbook](../README.md)

---

## 1. The write-ahead principle

```
WAL rule:     a log record describing a change must reach durable storage
              BEFORE the changed data page does.

Commit rule:  the COMMIT log record must be durable BEFORE the client
              is told the transaction committed.
```

```mermaid
flowchart TD
    W["Why log instead of writing pages directly?"] --> R1["A commit touches scattered pages<br/>⇒ random I/O, one seek each"]
    W --> R2["The log is a SINGLE sequential append<br/>⇒ one write, maximum throughput"]
    R1 & R2 --> C["WAL converts random durability<br/>into sequential durability.<br/>Data pages are written later, lazily,<br/>in whatever order suits the I/O system."]
    style C fill:#14532d,color:#fff
```

---

## 2. Anatomy of a log record

```
LSN            monotonically increasing position in the log
xid            which transaction
prev_lsn       previous record of the SAME transaction (backward chain for undo)
type           INSERT / UPDATE / DELETE / COMMIT / ABORT / CHECKPOINT / CLR
page_id        which page
redo_info      how to REDO the change (physical, logical, or physiological)
undo_info      how to UNDO it  (in undo-logging engines)
```

Each page also stores the **LSN of the last log record applied to it**. This one field
makes recovery idempotent: during redo, if `page.lsn >= record.lsn`, the change is
already present and is skipped. Recovery can therefore be interrupted and restarted
any number of times.

```mermaid
flowchart LR
    L1["LSN 100<br/>xid 7, page 42"] --> L2["LSN 140<br/>xid 8, page 91"] --> L3["LSN 180<br/>xid 7, page 42<br/>prev_lsn=100"] --> L4["LSN 220<br/>xid 7, COMMIT"]
    N["prev_lsn chains a transaction's records backwards<br/>⇒ undo walks the chain without scanning the whole log"]
    style N fill:#0b2545,color:#fff
```

### 2.1 Physical, logical and physiological

| Style | Records | Trade-off |
|---|---|---|
| **Physical** | Exact byte images of pages | Large, but trivially idempotent |
| **Logical** | "Insert row (x,y) into table T" | Compact; must be replayed against an identical logical state |
| **Physiological** | "On page 42, insert this tuple at slot 3" | **What real engines use** — physical to a page, logical within it |

Physiological logging is the practical compromise: page-local so it is idempotent and
order-independent across pages, but far smaller than full page images.

**Full-page writes** are the exception: after a checkpoint, the first modification to
a page logs the *entire page*. This protects against torn pages — a partial 8 KB write
during a power failure leaves a page that is neither the old nor the new version, and
no incremental record can repair it. Full-page writes are a significant fraction of
WAL volume, which is why checkpoint frequency affects WAL size so strongly.

---

## 3. Group commit

```mermaid
sequenceDiagram
    participant T1 as Txn 1
    participant T2 as Txn 2
    participant T3 as Txn 3
    participant W as WAL writer
    participant D as Disk

    T1->>W: COMMIT record, request flush
    W->>W: begin a small delay window
    T2->>W: COMMIT record
    T3->>W: COMMIT record
    W->>D: ONE fsync covering all three
    D-->>W: durable
    W-->>T1: committed
    W-->>T2: committed
    W-->>T3: committed
    Note over W,D: 3 transactions, 1 fsync.<br/>At 1 ms per fsync this is the difference<br/>between 1,000 and 100,000 TPS.
```

This is why database throughput frequently **improves** as concurrency rises: larger
commit groups amortise the fsync. It is also why a low-concurrency benchmark
understates a database's capacity.

---

## 4. Checkpoints

```mermaid
flowchart TD
    C["Checkpoint"] --> S1["Record the current LSN as the checkpoint start"]
    C --> S2["Flush all dirty buffer pages"]
    C --> S3["fsync the data files"]
    C --> S4["Write the checkpoint record"]
    C --> S5["Recycle WAL segments older than the<br/>REDO point"]
    S5 --> T["Recovery need only replay from<br/>the checkpoint's redo point"]

    F["Frequency trade-off"] --> F1["Frequent: short recovery, small WAL,<br/>but constant I/O and more full-page writes"]
    F --> F2["Infrequent: smooth I/O, but long recovery<br/>and a large WAL to retain"]
    style T fill:#14532d,color:#fff
```

### 4.1 Spreading the I/O

```
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9    # spread writes over 90% of the interval
max_wal_size = 8GB                    # raise so checkpoints are TIME-triggered,
                                      # not volume-triggered
```

A checkpoint triggered by WAL volume rather than by time is a warning sign: it means
the system is generating WAL faster than the checkpoint interval anticipated, and
those checkpoints arrive as unpredictable I/O bursts. Watch
`pg_stat_bgwriter.checkpoints_req` (requested) against `checkpoints_timed`; a healthy
system is overwhelmingly timed.

---

## 5. ARIES recovery

```mermaid
flowchart TD
    C["CRASH"] --> A["1. ANALYSIS<br/>Scan forward from the last checkpoint.<br/>Rebuild the Dirty Page Table and the<br/>Transaction Table. Determine the redo<br/>start point and the set of losers."]
    A --> R["2. REDO — 'repeat history'<br/>Replay EVERY logged change from the earliest<br/>dirty page LSN, including changes made by<br/>transactions that later aborted.<br/>Skip any record whose page.lsn is already ≥ it."]
    R --> U["3. UNDO<br/>Roll back all loser transactions, following<br/>prev_lsn chains backwards. Each undo writes<br/>a Compensation Log Record (CLR)."]
    U --> D["Consistent database, open for business"]
    style R fill:#0d3b66,color:#fff
    style D fill:#14532d,color:#fff
```

### 5.1 Why redo the work of aborted transactions

It seems wasteful, and it is the design's central insight. Repeating history exactly
restores the **precise pre-crash state**, which means undo then operates on a state it
fully understands. The alternative — trying to selectively redo only committed work —
requires reasoning about interleaved changes to the same page and is far more complex
and error-prone.

### 5.2 Compensation log records

```mermaid
flowchart LR
    N["Normal record<br/>LSN 500: set x = 9 (was 4)"] --> C["CLR<br/>LSN 700: set x = 4,<br/>undoNextLSN = 480"]
    C --> P["If the system crashes DURING recovery,<br/>the restarted recovery sees the CLR,<br/>knows LSN 500 is already undone,<br/>and continues from LSN 480.<br/>Undo is never repeated."]
    style P fill:#14532d,color:#fff
```

CLRs are **redo-only** — they are never themselves undone. This is what makes ARIES
recovery restartable, which matters because recovery of a large database can take long
enough for a second failure to occur.

---

## 6. The durability stack, honestly

```mermaid
flowchart TD
    A["COMMIT"] --> B["WAL buffer — process memory"]
    B --> C["write() → OS page cache"]
    C --> D["fsync() / fdatasync()"]
    D --> E["Disk controller cache"]
    E --> F["Persistent media"]

    Q1["Process crash: B is lost.<br/>Everything from C onward survives."] -.-> B
    Q2["OS/machine crash: C is lost<br/>unless fsync completed."] -.-> C
    Q3["Power loss: E is lost UNLESS the device<br/>has power-loss protection (capacitors).<br/>Enterprise SSDs do; many consumer<br/>SSDs do not and will lie about fsync."] -.-> E
    style D fill:#14532d,color:#fff
    style Q3 fill:#7d1128,color:#fff
```

Test your hardware's honesty rather than assuming it — tools such as `diskchecker.pl`
and `fio` with `--fsync=1` exist for exactly this. A storage device that acknowledges
fsync before data is durable will produce "impossible" corruption after a power event,
and no amount of database configuration will prevent it.

### 6.1 Configuration and what each setting really costs

| Setting | Effect | Loses |
|---|---|---|
| `synchronous_commit = on` | fsync WAL before ack | Nothing |
| `= remote_write` | Replica received it (not yet fsynced) | Both nodes failing simultaneously |
| `= remote_apply` | Replica has applied it — read-your-writes on replicas | Slowest |
| `= local` | Local fsync only, no replica wait | Committed data if the primary is lost |
| `= off` | No fsync wait at all | Up to ~3×`wal_writer_delay` of recent commits. **Still crash-consistent** |
| `fsync = off` | Never fsync | **Everything; possible corruption.** Throwaway instances only |
| `full_page_writes = off` | Skip torn-page protection | Corruption on power loss unless the filesystem/storage provides atomic 8 KB writes |

The distinction between `synchronous_commit = off` and `fsync = off` is the one people
conflate. The first loses *recent transactions* and leaves a consistent database; the
second can leave a *corrupt* one. The first is a legitimate tuning choice; the second
essentially never is.

---

## 7. What else the WAL powers

```mermaid
flowchart TD
    W["Write-Ahead Log"] --> R["Crash recovery"]
    W --> PR["Physical replication<br/>ship and replay WAL"]
    W --> LR["Logical replication / CDC<br/>decode WAL into row events"]
    W --> PITR["Point-in-time recovery<br/>base backup + WAL replay to an instant"]
    W --> IB["Incremental backup<br/>archive WAL between full backups"]
    W --> ST["Standby servers, read replicas,<br/>delayed replicas"]
    style W fill:#0b2545,color:#fff
```

### 7.1 WAL retention hazards

Three things hold WAL and can fill a disk:

| Holder | Symptom | Guard |
|---|---|---|
| **Replication slot** with a dead consumer | WAL grows without bound → disk full → database down | `max_slot_wal_keep_size` |
| `wal_keep_size` set too high | Constant large WAL directory | Size it deliberately |
| Failing `archive_command` | WAL cannot be recycled | Alert on `pg_stat_archiver.last_failed_time` |

**A dead logical replication slot is one of the few configuration mistakes that takes
a healthy primary database completely offline.** Setting `max_slot_wal_keep_size`
converts that outage into a broken slot — a much better failure.

```sql
-- monitor slot lag
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;
```

---

## 8. Point-in-time recovery

```mermaid
flowchart LR
    B["Base backup<br/>taken Sunday 02:00"] --> A["Archived WAL<br/>Sunday 02:00 → now"]
    A --> R["Restore the base backup"]
    R --> P["Replay WAL up to a target:<br/>recovery_target_time,<br/>recovery_target_lsn,<br/>recovery_target_xid, or<br/>recovery_target_name (a named restore point)"]
    P --> D["Database as it was at that exact instant —<br/>e.g. one second before the bad DELETE"]
    style D fill:#14532d,color:#fff
```

```sql
-- create a named restore point before something risky
SELECT pg_create_restore_point('before_v42_migration');
```

Practical notes: restore into a **new** instance rather than over the damaged one, so
you retain the ability to try a different target; use `recovery_target_action = pause`
so you can inspect the state before promoting; and remember that PITR restores the
*whole cluster*, so recovering one accidentally-dropped table means restoring
elsewhere and copying the table across.

---

## 9. Checklist

```
□ synchronous_commit setting is a deliberate, documented decision
□ fsync = on (unless the instance is genuinely disposable)
□ full_page_writes = on unless the storage guarantees atomic page writes
□ Storage hardware verified to honour fsync
□ checkpoint_completion_target = 0.9; checkpoints are timed, not WAL-triggered
□ max_wal_size sized so requested checkpoints are rare
□ WAL archiving configured, and archive failures alerted on
□ max_slot_wal_keep_size set so a dead slot cannot fill the disk
□ Replication slot lag monitored
□ PITR tested end to end — restore, replay to a target, verify with real queries
□ Recovery time measured against the stated RTO
```

---

[← previous: Locking and concurrency control](06-locking-and-concurrency-control.md) · [back to the handbook](../README.md) · [next: Replication →](08-replication.md)
