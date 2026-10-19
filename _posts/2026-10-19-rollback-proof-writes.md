---
layout: post
title: "Three Ways to Rollback-Proof Writes in PostgreSQL"
date: 2026-10-19
author: Hannu Krosing
render_with_liquid: false
---

* TOC
{:toc}

PostgreSQL's transaction model is built on a simple, uncompromising contract: **all or nothing**. Every `INSERT`, `UPDATE`, and `DELETE` executed inside a `BEGIN ... COMMIT` block is stamped with the transaction's ID (`xmin` / `xmax`) and remains invisible to the outside world until the transaction commits. If the transaction executes `ROLLBACK`, raises an unhandled `ERROR`, or if the server crashes before `COMMIT` completes, PostgreSQL acts as if none of those writes ever happened.

Ninety-nine percent of the time, this strict atomicity is exactly what you want. Look closer at the remaining one percent, however, and you will find a class of problems where **you explicitly need a write inside a transaction to survive even if the surrounding transaction rolls back or crashes**.

In this post, we explore the problem of rollback-proof writes in PostgreSQL and examine three architectural solutions proposed for PostgreSQL—arranged along a spectrum of **growing complexity and engine heaviness**:

1. **Immediate Global Insert Visibility (`SET LOCAL immediate_global_insert_visibility`)** — Tuple-level freeze-on-insert with synchronous WAL flush (**immediately visible and immediately crash-proof per `INSERT`**).
2. **Committing Savepoints (`COMMIT SAVEPOINT`)** — Subtransaction-level early commit with 15 safety blocker checks (**rollback-proof and crash-proof once `COMMIT SAVEPOINT` is reached**).
3. **Autonomous Transactions (`BEGIN AUTONOMOUS ... COMMIT AUTONOMOUS`)** — Isolated transaction frame with independent snapshot, GUC, trigger, and portal stacks (**rollback-proof and crash-proof once `COMMIT AUTONOMOUS` is reached**).

---

## 1. The Problem to Solve: Why Would You Want a Write to Survive `ROLLBACK`?

At first glance, wanting data to persist when a transaction aborts sounds like an anti-pattern. Why not just commit? Because in real-world database systems, the *reason* you are writing the record is often inseparable from the fact that the outer transaction is still in flight—or is about to fail.

### Use Case A: Crash-Safe Two-Phase Commit (2PC) Coordinator Ledgers
Suppose your PostgreSQL backend acts as a distributed transaction coordinator (for example, via `pl/proxy`, `postgres_fdw`, or `dblink`) executing a Two-Phase Commit across remote shards:

1. Before sending `PREPARE TRANSACTION '<gid>'` over the network to a remote node, the coordinator **must** record `<gid>` in a local `prepared_xact_log` table.
2. Why? Because if the remote node processes `PREPARE TRANSACTION '<gid>'` and then the local PostgreSQL coordinator crashes, loses network connectivity, or aborts its local transaction before Phase 2 (`COMMIT PREPARED` / `ROLLBACK PREPARED`) finishes, an external recovery manager needs to read `prepared_xact_log` to find and resolve the orphaned remote prepared transaction.
3. If the `INSERT INTO prepared_xact_log` uses normal PostgreSQL MVCC, then a local transaction abort or crash rolls back the very log entry that was supposed to tell the recovery daemon about the orphaned `<gid>`!
4. Furthermore, that log entry must be **durable on disk before the network call leaves the box**—not merely at the end of some multi-step block.

### Use Case B: Security, Compliance, and Audit Logging
Regulated applications (finance, healthcare, authentication) often require that every access attempt or policy violation be recorded in an audit table inside the database—especially the ones that fail:

```sql
BEGIN;
-- 1. Record that user_id=42 attempted a high-value transfer
INSERT INTO security_audit_log (user_id, action, attempted_at)
VALUES (42, 'WIRE_TRANSFER $500,000', clock_timestamp());

-- 2. Verify daily limit and compliance rules
IF NOT check_compliance_limits(42, 500000) THEN
    RAISE EXCEPTION 'Compliance limit exceeded for user 42';
END IF;
COMMIT;
```

In standard PostgreSQL, raising that exception aborts the transaction and **erases the `security_audit_log` row**. The attacker or malfunctioning client leaves zero trace in the audit table.

### Use Case C: Error Logging Inside Stored Functions and Triggers
When a complex PL/pgSQL payment pipeline or trigger fails deep inside nested function calls, engineers want to write structured diagnostic payloads (stack traces, input JSON, intermediate state) into an `error_log` table and then re-raise the exception so all business table changes roll back cleanly.

### Use Case D: Real-Time Progress & Queue Producers in Long Transactions
A batch job or streaming producer running inside a long transaction may want to emit progress heartbeats or enqueue notifications into an append-only table that monitoring dashboards and worker processes can see immediately—without prematurely committing the main business transaction.

### Why Existing Workarounds (`dblink` / Background Workers) Hurt at Scale
Historically, PostgreSQL users have worked around this by opening a loopback TCP/Unix-socket connection back into the same database using `dblink` or `postgres_fdw`. While functional for low-rate tasks, loopback connections are painful in production:

- **Connection amplification**: Every backend executing a logged transaction opens a second `postgres` backend process (or exhausts `max_connections` / deadlocks `pgBouncer` pools when all pool slots are held by outer transactions waiting for inner loopback connections).
- **Latency overhead**: Forking a backend or round-tripping through a socket and parser adds milliseconds of overhead to what should be a microsecond local page write.
- **Credential & security management**: Loopback connections require managing connection strings, `.pgpass` / user mappings, and role switching logic.

---

## 2. The Spectrum of Solutions: Growing Complexity and Heaviness

When we ask *"How can PostgreSQL make a write survive an outer transaction rollback?"*, there isn't just one answer. Instead, there are three distinct architectural points along a spectrum of increasing generality—and rapidly increasing engine complexity:

![three_ways_to_rollback_proof_writes_blog diagram 1](/assets/images/postsql_postgres/three_ways_to_rollback_proof_writes_blog_diagram_1.svg)

### The Critical Distinction: *When* Does a Write Become Crash-Proof?

Before diving into the internals of each patch, there is one foundational difference that sets **Immediate Insert Visibility** apart from both **`COMMIT SAVEPOINT`** and **`BEGIN AUTONOMOUS`**:

> [!IMPORTANT]
> **Immediate Insert Visibility is crash-proof *immediately* after each `INSERT` statement completes.**
> Both **`COMMIT SAVEPOINT`** and **`BEGIN AUTONOMOUS`** are transactional containers: writes inside them remain uncommitted, invisible, and volatile until execution actually reaches their **`COMMIT SAVEPOINT`** or **`COMMIT`** statement.

Why does this matter? Consider what happens if your backend executes an `INSERT` and then—before reaching the end of the sub-block or procedure—the server loses power, the OOM killer sends `SIGKILL`, or a C extension segfaults:

![three_ways_to_rollback_proof_writes_blog diagram 2](/assets/images/postsql_postgres/three_ways_to_rollback_proof_writes_blog_diagram_2.svg)

With **Immediate Insert Visibility**, durability and visibility happen **at the tuple level inside the `INSERT` itself**. With **`COMMIT SAVEPOINT`** and **`BEGIN AUTONOMOUS`**, durability and visibility are deferred until the transaction or subtransaction state machine transitions to committed.

---

## 3. Deep Dive into Each Approach

### Approach 1: Immediate Global Insert Visibility (`SET LOCAL immediate_global_insert_visibility`)

#### The Concept
What if, for specific append-only or tracking tables, we don't need a new transaction state machine, a new `TransactionId`, a new snapshot, or a separate lock table at all? What if we simply stamp newly inserted tuples as **already frozen (`HEAP_XMIN_FROZEN`)** and flush the WAL record before the `INSERT` returns?

That is exactly what the `immediate_global_insert_visibility` patch does. It introduces a transaction-local GUC listing the tables whose inserts should bypass normal MVCC commit wait:

```sql
-- Default 'sync' mode (immediate crash safety with XLogFlush)
SET LOCAL immediate_global_insert_visibility = 'public.twophase_gid_ledger';

-- Optional 'async' or per-table 'sync'/'async' modifiers (relaxed crash safety without XLogFlush)
SET LOCAL immediate_global_insert_visibility = 'public.audit_log:async, public.twophase_gid_ledger:sync';
-- or prefix / list-level syntax:
SET LOCAL immediate_global_insert_visibility = 'async:public.audit_log, public.telemetry_events';
```

#### How It Works Under the Hood
A naive implementation might try to stamp `FrozenTransactionId` onto the tuple header *before* calling `heap_insert()`. However, doing so breaks PostgreSQL's B-tree uniqueness checks and speculative insertions (`INSERT ... ON CONFLICT`). When `btinsert()` checks for duplicate keys during `ExecInsertIndexTuples()`, it inspects the inserting transaction's active XID to coordinate concurrent conflict resolution.

To preserve 100% compatibility with unique indexes, constraints, TOAST, and `ON CONFLICT`, the patch uses a **two-step insert-then-freeze pipeline** inside `ExecInsert()` and `CopyMultiInsertBufferFlush()`:

1. **Standard Heap & Index Insertion**:
   - `table_tuple_insert()` / `table_multi_insert()` writes the heap tuple with the current transaction's normal `xmin` (`GetCurrentTransactionId()`).
   - If the tuple has out-of-line TOAST attributes, `heap_prepare_insert()` passes `HEAP_INSERT_IMMEDIATE_FROZEN` so newly inserted TOAST chunk tuples are immediately frozen on the TOAST table.
   - `ExecInsertIndexTuples()` inserts into all indexes and resolves any unique constraint or `ON CONFLICT` checks using standard MVCC semantics.
2. **Immediate Tuple Freeze & Configurable `sync` / `async` WAL Flush**:
   - Right after `ExecInsertIndexTuples()` succeeds, `heap_freeze_inserted_tuple()` re-locks the target heap buffer in `BUFFER_LOCK_EXCLUSIVE`.
   - Inside a critical section (`START_CRIT_SECTION()`), it sets `HEAP_XMIN_FROZEN` (`HEAP_XMIN_COMMITTED | HEAP_XMIN_INVALID`) on the tuple's `t_infomask`, marks the buffer dirty, and writes an `XLOG_HEAP2_PRUNE_ON_ACCESS` freeze WAL record.
   - **In `sync` mode (the default)**: it calls **`XLogFlush(recptr)`** (once per single-row `INSERT`, or once at the end of a multi-insert batch in `COPY`), guaranteeing **immediate crash safety** before the `INSERT` returns.
   - **In `async` mode (`table:async` or `async:table`)**: it **skips `XLogFlush(recptr)`**. Because the tuple is already marked `HEAP_XMIN_FROZEN` in shared buffers and its freeze WAL record is already in shared WAL buffers, the row is **immediately visible to all concurrent sessions and 100% immune to `ROLLBACK` or transaction errors**, while avoiding synchronous `fdatasync()` latency and `WALWriteLock` contention on every single-row `INSERT`.

#### Why the 2PC Insert-Then-Delete Pattern Works So Elegantly
Consider how `immediate_global_insert_visibility` solves the 2PC coordinator problem in under 15 lines of SQL:

```sql
BEGIN;
-- Enable immediate frozen inserts for our local 2PC tracking table
SET LOCAL immediate_global_insert_visibility = 'public.twophase_gid_ledger';

-- 1. Do local and remote transactional work
INSERT INTO accounts_local VALUES (...);
SELECT dblink_exec('shard_1', 'INSERT INTO accounts_remote VALUES (...)');

-- 2. Log the 2PC GID locally BEFORE sending PREPARE TRANSACTION
INSERT INTO public.twophase_gid_ledger (gid, remote_node, created_at)
VALUES ('tx_99182', 'shard_1', clock_timestamp());
-- ^^^ At this exact line, the row is FROZEN on the heap page and XLogFlush()'d to disk!
-- Even if power is pulled 1ms later while PREPARE is in flight, this row survives.

-- 3. Prepare and commit on the remote shard
SELECT dblink_exec('shard_1', 'PREPARE TRANSACTION ''tx_99182''');
SELECT dblink_exec('shard_1', 'COMMIT PREPARED ''tx_99182''');

-- 4. Remote 2PC succeeded! Delete the tracking entry using normal MVCC:
DELETE FROM public.twophase_gid_ledger WHERE gid = 'tx_99182';

COMMIT;
```

Look closely at step 4 (`DELETE FROM public.twophase_gid_ledger`):

- Because `immediate_global_insert_visibility` only freezes `INSERT`s, the `DELETE` stamps a **normal MVCC `xmax`** on the frozen tuple.
- If the local transaction reaches `COMMIT`, the `DELETE` commits and the tracking row is gone—zero leftover clutter.
- If the local transaction **crashes or rolls back** before `COMMIT`, the `DELETE` rolls back, while the frozen `INSERT` from step 2 **remains alive and visible** so the recovery daemon can verify and clean up `'tx_99182'`!

#### Guardrails and Limitations
Because freezing a tuple in place is a sharp, low-level primitive, the patch enforces strict guardrails:

- **`SET LOCAL` only**: Rejects `SET` at session level, `postgresql.conf`, `ALTER SYSTEM`, or `ALTER ROLE/DATABASE` (`GUC_NO_SHOW_ALL | GUC_NOT_IN_SAMPLE`), preventing accidental session-wide activation via connection poolers.
- **Eager ACL & Relkind Validation**: Immediately verifies `ACL_INSERT` permissions and rejects system catalogs, views, foreign tables, and relations with `BEFORE` or `INSTEAD OF` insert triggers (whose side effects on other tables would still use normal MVCC and roll back on error).
- **Halloween Problem Protection**: Forces `update_tuple_routing_needed` when modifying partitioned tables in the list so a cross-partition `UPDATE` (which deletes from partition A and inserts into partition B) never sees its own newly frozen tuple twice in the same scan.
- **`INSERT`-only**: You cannot make an `UPDATE` or `DELETE` rollback-proof with this mechanism.

---

### Approach 2: Committing Savepoints (`COMMIT SAVEPOINT`)

#### The Concept
What if an `INSERT`-only primitive isn't enough? Suppose your rollback-proof operation needs to:

1. `UPDATE` an existing counter row (e.g., `SET failed_login_count = failed_login_count + 1 WHERE user_id = 42`), **and**
2. `INSERT` a detail record into `login_audit_log`, **and**
3. Have those two statements succeed or fail *together as an atomic unit*—so if the `INSERT` violates a constraint, the `UPDATE` can be rolled back via `ROLLBACK TO SAVEPOINT`, but if both succeed, they can be permanently committed via **`COMMIT SAVEPOINT`** while the outer transaction continues!

This is the motivation behind **`COMMIT SAVEPOINT`**, the second point on our complexity spectrum:

```sql
BEGIN;

-- Outer transaction work
UPDATE bank_accounts SET balance = balance - 500 WHERE acct_id = 10;

SAVEPOINT audit_step;
    -- Multiple DML statements (INSERT, UPDATE, DELETE) inside the savepoint
    UPDATE user_security_stats
       SET transfer_attempts = transfer_attempts + 1,
           last_attempt_at = clock_timestamp()
     WHERE user_id = 42;

    INSERT INTO security_audit_log (user_id, acct_id, amount)
    VALUES (42, 10, 500);

-- Permanently commit the work done since SAVEPOINT audit_step!
COMMIT SAVEPOINT audit_step;

-- Later in the outer transaction, something fails:
ROLLBACK;
-- The bank_accounts UPDATE is rolled back,
-- but the user_security_stats UPDATE and security_audit_log INSERT remain committed!
```

#### How It Works (and How It Differs from `RELEASE SAVEPOINT`)
In standard PostgreSQL, `SAVEPOINT sp` starts a **subtransaction** (`SubTransactionId`) with its own child `TransactionState` on the backend's transaction stack and, once it modifies data, its own `TransactionId` (sub-XID).

- Standard **`RELEASE SAVEPOINT sp`** does *not* commit the sub-XID to disk. Instead, it merges the child subtransaction's XID, locks, and resources into the parent transaction (`pg_subtrans` links `child_xid -> parent_xid`). If the parent later aborts, `child_xid` aborts too.
- **`COMMIT SAVEPOINT sp`** (`CommitSavepointSubTransaction()`) changes the subtransaction termination path: instead of merging `child_xid` into the parent's uncommitted XID tree, it calls `RecordSubTransactionCommit()` to **write a commit WAL record (`XLOG_XACT_COMMIT`), flush WAL (`XLogFlush`), mark `child_xid` (and any nested child XIDs) as `TRANSACTION_STATUS_COMMITTED` in `pg_xact` (CLOG), clear its parent link in `pg_subtrans` (`SubTransClearParent`), and purge the XIDs from `MyProc->subxids` in shared memory (`XidCacheRemoveRunningXids`)**, while releasing the savepoint's locks immediately via `ResourceOwnerRelease(s->curTransactionOwner, RESOURCE_RELEASE_LOCKS, false, false)`.

#### Durability Timing: Must Reach `COMMIT SAVEPOINT`
Unlike `immediate_global_insert_visibility`, statements inside a `SAVEPOINT ... COMMIT SAVEPOINT` block use ordinary MVCC tuple headers stamped with `child_xid`:

- While execution is between `SAVEPOINT audit_step` and `COMMIT SAVEPOINT audit_step`, the rows are **not visible** to other sessions and **not crash-proof**.
- If the backend crashes or an error occurs before `COMMIT SAVEPOINT audit_step` is reached, `child_xid` is never marked committed in CLOG and all writes inside the savepoint roll back.
- Only when execution reaches `COMMIT SAVEPOINT audit_step` and `XLogFlush()` completes do those writes become crash-durable and globally visible.

#### Why `COMMIT SAVEPOINT` Requires 15 Runtime Blocker Checks (`CheckCommitSavepointBlockers`)
While `COMMIT SAVEPOINT` looks like a small syntactic addition next to `RELEASE SAVEPOINT`, a savepoint shares the parent transaction's snapshot, GUC state, trigger queue, portal table, and lock space right up until the moment you ask to commit it independently. To prevent corrupting the enclosing transaction, `CheckCommitSavepointBlockers()` must inspect **15 subsystem invariants** at commit time and reject `COMMIT SAVEPOINT` if any are violated:

1. **Breaking the `pg_subtrans` & Snapshot Hierarchy**:
   PostgreSQL's MVCC visibility routines (`TransactionIdIsInProgress()` and `XidInMVCCSnapshot()`) rely on the invariant that **a subtransaction cannot be committed unless its top-level parent transaction is also committed**. When `COMMIT SAVEPOINT` commits `subxid` while `topxid` is still running:

   - `SubTransClearParent()` must zero the parent link in `pg_subtrans`,
   - `XidCacheRemoveRunningXids()` must splice `subxid` out of `MyProc->subxids` under `ProcArrayLock`, and
   - `RecordCommittedSavepointXids()` must record `subxid` on the enclosing transaction root (and serialize it to parallel workers) so an enclosing `REPEATABLE READ` snapshot taken *before* the savepoint still sees its own committed savepoint's tuples in `XidInMVCCSnapshot()`!
2. **Uncommitted Parent Tuple Dependencies & Lock Conflicts**:
   - **Tuple chains (`minModifiedAncestorLevel`)**: If the outer transaction updated or locked row $$R_1$$ (`xmin = top_xid`) and `SAVEPOINT sp` updated or deleted the *same* row $$R_1$$ (`xmax = sub_xid`), committing `sub_xid` while later aborting `top_xid` would leave a committed update/delete pointing to an aborted parent tuple version! Therefore, `heap_update()`, `heap_delete()`, and `heap_lock_tuple()` must call `RecordSavepointTupleDependency()` on every tuple modification, and `COMMIT SAVEPOINT` errors out if the savepoint touched any tuple modified by an uncommitted ancestor.
   - **Conflicting Locks (`LockHasSavepointBlockers`)**: Because `COMMIT SAVEPOINT` releases the savepoint's locks (`ResourceOwnerRelease(..., isCommit=false)`), `LockHasSavepointBlockers()` must scan `LockMethodLocalHash` and forbid committing if the savepoint and an ancestor transaction hold self-conflicting locks on the same object, or if the savepoint executed an explicit `LOCK TABLE` or `pg_advisory_xact_lock()`.
3. **11 Additional Subsystem Blockers**:
   Because the savepoint was not isolated on entry, `CheckCommitSavepointBlockers()` must also forbid `COMMIT SAVEPOINT` if:

   - The transaction is in `SERIALIZABLE` isolation (`IsolationIsSerializable()`) or has exported a snapshot,
   - The savepoint or an uncommitted ancestor has catalog/DDL invalidation messages (`InvalHasSubXactChanges()` / `InvalHasAncestorChanges()`) or pending relation file deletes (`smgrHasSubXactPendingDeletes()`),
   - The savepoint queued pending `AFTER` triggers or `SET CONSTRAINTS` changes (`AfterTriggerHasSubXactBlockers()`),
   - The savepoint created or modified open cursors/portals (`PortalHasSubXactBlockers()`), queued `LISTEN`/`UNLISTEN`/`NOTIFY` actions (`AsyncHasSubXactBlockers()`), or has `SET LOCAL` / conflicting `SET` GUC changes (`GUCSubXactHasBlockers()`),
   - The savepoint registered `ON COMMIT` temp-table actions, initialized a temp namespace, allocated virtual file or large-object descriptors, or changed `GetUserId()` (`SET ROLE` / `SECURITY DEFINER`).

---

### Approach 3: Autonomous Transactions (`BEGIN AUTONOMOUS ... COMMIT AUTONOMOUS`)

#### The Concept
At the far end of the spectrum is the full **Autonomous Transaction** (`BEGIN AUTONOMOUS [TRANSACTION] ... COMMIT AUTONOMOUS` / `ROLLBACK AUTONOMOUS`), familiar to anyone who has migrated applications from Oracle or DB2.

Instead of freezing individual tuples (Approach 1) or retroactively checking whether a shared-state savepoint is safe to detach (Approach 2), an autonomous transaction isolates its state **upfront on entry**: **"Pause my current transaction's snapshot, GUC state, trigger queue, invalidation list, and lock ownership, start an isolated autonomous transaction frame, let me run arbitrary SQL (including DDL, triggers, `NOTIFY`, cursors, and nested `SAVEPOINT`s) and `COMMIT AUTONOMOUS` or `ROLLBACK AUTONOMOUS`, and then resume my paused transaction."**

```sql
BEGIN;

-- Main transaction updates an order
UPDATE orders SET status = 'PROCESSING' WHERE order_id = 9001;

-- Start a completely independent autonomous transaction
BEGIN AUTONOMOUS;
    -- Runs with its own XID, its own Snapshot, and its own Locks!
    -- Cannot see uncommitted changes from the paused main transaction.
    INSERT INTO payment_gateway_attempts (order_id, gateway, attempted_at)
    VALUES (9001, 'stripe', clock_timestamp());

    UPDATE gateway_metrics
       SET total_calls = total_calls + 1
     WHERE gateway = 'stripe';
COMMIT AUTONOMOUS; -- Commits the autonomous transaction (XLogFlush) and resumes the main transaction

-- Main transaction encounters an error and aborts
ROLLBACK;
-- orders update is rolled back; payment_gateway_attempts & gateway_metrics stay committed!
```

#### How It Works Under the Hood
When a backend executes `BEGIN AUTONOMOUS` (`StartAutonomousSubTransaction()`):

1. **Upfront Isolation on Entry**:
   The backend pushes an autonomous transaction frame (`s->isAutonomous = true`, incrementing `s->savepointLevel`), saves the current GUC state (`BeginAutonomousGUC()`), isolates the snapshot stack (`AtAutonomousStart_Snapshot()` so it cannot see the parent's `ActiveSnapshot`), starts a separate after-trigger context (`AfterTriggerBeginAutonomousSubXact()`), and creates a new `CurTransactionContext` and `ResourceOwner`.
2. **Isolated Visibility (`TransactionIdIsCurrentTransactionId`)**:
   In `TransactionIdIsCurrentTransactionId()`, traversal up the `CurrentTransactionState` parent chain stops at `s->isAutonomous`. That means inside the autonomous block, the parent transaction's uncommitted XIDs are **not** treated as "current transaction"—the autonomous transaction sees only globally committed rows (plus its own writes), and operates under its own `CommandId` visibility domain.
3. **Commit or Rollback at the Autonomous Boundary (`CommitAutonomousSubTransaction`)**:
   When execution reaches `COMMIT AUTONOMOUS`, it runs a full mini-`CommitTransaction` sequence for the autonomous frame: firing `AfterTriggerEndAutonomousSubXact(true)`, `PreCommit_Autonomous_on_commit_actions()`, `PreCommit_Autonomous_Notify()`, calling `RecordSubTransactionCommit()` (`XLOG_XACT_COMMIT` + `XLogFlush()` + `TransactionIdCommitTree()` + `SubTransClearParent()` + `XidCacheRemoveRunningXids()`), closing autonomous portals (`AtAutonomousCommit_Portals()`), broadcasting catalog invalidations (`AtAutonomousCommit_Inval()`), flushing pgstat (`AtAutonomousCommit_PgStat()`), and releasing the autonomous block's locks (`ResourceOwnerRelease(s->curTransactionOwner, RESOURCE_RELEASE_LOCKS, false, false)`).

#### Durability Timing: Must Reach the Autonomous `COMMIT`
Just like `COMMIT SAVEPOINT`, an autonomous transaction is a multi-statement transactional container:

- Nothing written inside `BEGIN AUTONOMOUS ...` is visible or crash-durable while the autonomous block is still executing.
- If a statement inside the autonomous block raises an error, or if the database server crashes *before* execution reaches the matching `COMMIT;`, the entire autonomous transaction rolls back during crash recovery.
- Only when execution reaches `COMMIT;` and flushes the commit WAL record are its writes safe from a crash.

#### Why `BEGIN AUTONOMOUS` Is the Heaviest Mechanism in PostgreSQL
Autonomous transactions have been proposed and debated on `pgsql-hackers` multiple times over the past decade (including designs using in-backend transaction context switching as well as designs delegating to background worker pools). They are by far the heaviest of the three approaches for three fundamental architectural reasons:

1. **Self-Deadlocks Within a Single Backend**:
   Because an autonomous transaction is completely independent of the suspended parent transaction, **it cannot share the parent's locks** (otherwise, if the autonomous transaction modified a table or row locked by the parent and then committed, it would corrupt the parent's uncommitted state or violate lock isolation).
   Consequently, if the parent transaction holds a lock that conflicts with a lock requested by the autonomous transaction, **the single-threaded backend blocks waiting for itself**:

   ```sql
   BEGIN;
   UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- Parent holds row lock on id=1

   BEGIN AUTONOMOUS TRANSACTION;
       -- Tries to update the same row id=1 -> WAITS FOR PARENT TRANSACTION -> SELF-DEADLOCK!
       UPDATE accounts SET audit_flag = true WHERE id = 1;
   COMMIT;
   ```

   The lock manager (`LockAcquire()` / `DeadLockCheck()`) must partition `LOCALLOCK` and `PROCLOCK` ownership by autonomous nesting level and immediately detect self-deadlocks across suspended transactions in the same PID.
2. **Invisibility of the Parent Transaction's Uncommitted Data**:
   Because the autonomous transaction takes its own fresh MVCC snapshot, **it cannot see uncommitted rows inserted or updated by the parent transaction**. For example, if your parent transaction inserts a new `users` row (`user_id = 500`) and then enters `BEGIN AUTONOMOUS` to insert an audit record into `audit_log(user_id)` which has a foreign key `REFERENCES users(user_id)`, **the foreign key check inside the autonomous transaction will fail** because `user_id = 500` does not exist yet in the committed snapshot!
3. **Multiplexing `PGPROC` in Shared Memory**:
   PostgreSQL's shared memory architecture assigns exactly one `PGPROC` / `PGXACT` slot per backend process, advertising a single `xid` and `xmin` to all other backends in `ProcArray`. When a backend suspends a parent transaction (with `xid = 1001, xmin = 950`) to run an autonomous transaction (which gets `xid = 1005`), the backend must still advertise `xid = 1001` as in-progress and keep `xmin = 950` pinned against `VACUUM` while simultaneously advertising and committing `xid = 1005`. Supporting this cleanly requires either secondary dummy `PGPROC` slots (similar to `PreparedTransaction` gxact entries) or deep surgery across `ProcArray`, `sinval`, `SerializableXact`, and `ResourceOwner`.

---

## 4. In-Depth Resource Analysis: Disk, CPU, Memory, and Locks

> [!NOTE]
> **A Quick Note on `BEGIN ATOMIC` vs. `BEGIN AUTONOMOUS`**:
> Standard SQL:2003 / PostgreSQL 14+ supports `BEGIN ATOMIC ... END` for SQL-language function bodies (`CREATE FUNCTION ... BEGIN ATOMIC ... END`), and PL/pgSQL uses `BEGIN ... EXCEPTION ... END` blocks (backed by internal subtransactions via `BeginInternalSubTransaction()`). Neither of those makes writes survive an outer `ROLLBACK`—a standard `BEGIN ATOMIC` body runs directly inside the caller's transaction with zero subtransaction boundary, and a PL/pgSQL `BEGIN ... END` block merely merges its subtransaction into the parent on exit (just like `RELEASE SAVEPOINT`). To make a block rollback-proof, you either need **`COMMIT SAVEPOINT`** (subtransaction-level independent commit) or **`BEGIN AUTONOMOUS`** (independent top-level transaction).

To understand why these three mechanisms sit at very different points on the heaviness spectrum, let's trace their exact footprint inside a PostgreSQL backend across **Disk**, **CPU**, **Memory**, and **Locks**.

![three_ways_to_rollback_proof_writes_blog diagram 3](/assets/images/postsql_postgres/three_ways_to_rollback_proof_writes_blog_diagram_3.svg)

---

### 4.1 Disk I/O Footprint (WAL, `fdatasync`, SLRUs, and Future `VACUUM`)

Disk cost in PostgreSQL has four distinct components: **WAL bytes written**, **synchronous WAL flushes (`XLogFlush` / `fdatasync`)**, **SLRU metadata writes (`pg_xact` and `pg_subtrans`)**, and **deferred read/vacuum write amplification**.

| Disk Metric | 1a. Immediate Writes (`async` mode) | 1b. Immediate Writes (`sync` mode, default) | 2. `COMMIT SAVEPOINT` | 3. `BEGIN AUTONOMOUS` |
| :--- | :--- | :--- | :--- | :--- |
| **Extra WAL Bytes (Single Insert)** | **`66 bytes`** (`XLOG_HEAP2_PRUNE_ON_ACCESS`) | **`66 bytes`** (`XLOG_HEAP2_PRUNE_ON_ACCESS`) | **`~36–48 bytes`** (`XLOG_XACT_COMMIT` + `~32 B` `XLOG_XACT_ASSIGNMENT` if subxids $$\gt  64$$) | **`~36–48 bytes`** (`XLOG_XACT_COMMIT` record) |
| **`XLogFlush()` (`fdatasync`) Frequency** | **0 per `INSERT`!**<br/>Flushed asynchronously by `walwriter` or next commit. | **1 per `INSERT` statement** (or **1 per `64 KB` batch** in `COPY`). | **1 per `COMMIT SAVEPOINT`**. | **1 per `COMMIT AUTONOMOUS`**. |
| **SLRU Disk I/O (`pg_xact` & `pg_subtrans`)** | **0 bytes (Zero SLRU writes!)** | **0 bytes (Zero SLRU writes!)** | **Writes to BOTH `pg_subtrans` (`4 B`/subxid) and `pg_xact` (`2 bits`/subxid)**. | **Writes to `pg_xact` (`2 bits`/xid)** + burns 1 XID. |
| **Deferred Hint-Bit FPIs & Future `VACUUM` Freeze I/O** | **Zero!**<br/>Born `HEAP_XMIN_FROZEN` in L1 cache. | **Zero!**<br/>Born `HEAP_XMIN_FROZEN` in L1 cache. | **Significant deferred I/O**:<br/>Hint-bit page dirtying (up to **`8 KB` FPI**) + future `VACUUM` freeze. | **Significant deferred I/O**:<br/>Hint-bit page dirtying (up to **`8 KB` FPI**) + future `VACUUM` freeze. |

#### Live Benchmark on `ext4` (`fsync = on`, 1,000 Rows with Primary Key Index)

| Mode | Total Time (`1,000` rows) | Latency per Row | Total WAL Bytes | Extra Cost vs. Normal `INSERT` |
| :--- | :---: | :---: | :---: | :--- |
| **Baseline: Normal MVCC (1,000 single-row `INSERT`s)** | `118.4 ms` | `118.4 µs` | `146,104 B` (`146 B/row`) | — |
| **`immediate_global_insert_visibility` (`async`, 1,000 single-row `INSERT`s)** | **`120.5 ms`** | **`120.5 µs`** | `218,320 B` (`218 B/row`) | **`+2.0 µs/row` (`+1.7%` CPU/wall time)**, `0` `fdatasync`s |
| **`immediate_global_insert_visibility` (`sync`, 1,000 single-row `INSERT`s)** | `563.7 ms` | `563.7 µs` | `218,320 B` (`218 B/row`) | **`+445.3 µs/row` (`4.76x`)** due to 1,000 `XLogFlush()` calls |
| **`immediate_global_insert_visibility` (`async`, `COPY` 1,000 rows)** | **`9.0 ms`** | **`9.0 µs`** | `170,632 B` (`171 B/row`) | **`0` `fdatasync`s** |
| **`immediate_global_insert_visibility` (`sync`, `COPY` 1,000 rows)** | **`12.1 ms`** | **`12.1 µs`** | `166,528 B` (`167 B/row`) | **1 `XLogFlush()` for entire batch (`+3.1 ms` total)** |

**Disk Verdict**:

- Adding **`async` mode (`table:async` / `async:table`)** completely removes the only scenario where Immediate Writes could be heavier than `COMMIT SAVEPOINT` or `BEGIN AUTONOMOUS`: high-frequency loops of single-row `INSERT` statements. In `async` mode, 1,000 single-row `INSERT`s run in **`120.5 ms` vs `118.4 ms` (`+1.7%` overhead, `+2.0 µs/row`)** while still making every row immediately visible to all concurrent sessions and 100% rollback-proof against `ROLLBACK` or transaction errors.
- In **`sync` mode (the default)**, every single-row `INSERT` pays one `XLogFlush()` (`~443 µs` on local SSD) in exchange for **immediate, zero-window power-loss durability** before the `INSERT` returns (critical for 2PC `<gid>` logging before sending `PREPARE TRANSACTION` over the network). And when using `COPY` multi-insert, `sync` mode amortizes `XLogFlush()` across the batch (`12.1 ms` for 1,000 rows, or `12.1 µs/row`).

---

### 4.2 CPU Footprint (Instruction Path, Snapshots, and Teardown Hooks)

| CPU Phase | 1. Immediate Writes (`immediate_global_insert_visibility`) | 2. `COMMIT SAVEPOINT` | 3. `BEGIN AUTONOMOUS` |
| :--- | :--- | :--- | :--- |
| **Block / Statement Entry** | `RelationHasImmediateInsertVisibility()`: 1 branch (`if (likely(!immediate_insert_visibility_active))`) + scan of 1–2 `Oid`s in L1 cache (**`~5 ns`**). | `StartSubTransaction()`: allocates `TransactionStateData`, creates `CurTransactionContext` (`AllocSetContextCreate`), creates `ResourceOwner`, fires `SUBXACT_EVENT_START_SUB` callbacks. | Saves entire parent `TransactionState`, `ActiveSnapshot`, `CurrentResourceOwner`, and GUC stack; runs `StartTransaction()`. |
| **XID Assignment (`GetNewTransactionId`)** | **None** (reuses parent transaction's XID during index insertion). | **1 `GetNewTransactionId()` call**: acquires `XidGenLock`, advances `nextXid`, zeros CLOG bits, writes `pg_subtrans`, appends to `MyProc->subxids`. | **1 `GetNewTransactionId()` call**: acquires `XidGenLock`, advances `nextXid`, zeros CLOG bits, updates `MyProc->xid` and `ProcGlobal->xids[]`. |
| **Snapshot CPU (`GetSnapshotData`)** | **0 calls**, and **0 cost added to other backends**. | **0 calls on entry**, **but slows down concurrent `GetSnapshotData()` callers** across the server by adding active `subxids` (and if $$\gt  64$$ subxids, forces concurrent readers onto the slow `pg_subtrans` lookup path). | **1 full `GetSnapshotData()` call** ($$O(N_{\text{backends}})$$ scan across `ProcGlobal->xids[]`), **plus** adds another active top-level XID for all concurrent `GetSnapshotData()` callers to scan. |
| **Freeze / Commit Teardown CPU** | `heap_freeze_inserted_tuple()`: re-pins buffer already hot in L1/shared buffers, sets `HEAP_XMIN_FROZEN`, logs WAL, flushes. **Sub-microsecond CPU overhead (`< 1 µs`)** excluding I/O wait. | `CommitSubTransaction()`: walks 8 resource arrays/hashes in `ResourceOwner`, transfers locks/pins to parent, updates `pg_xact`, splices committed subxid out of `MyProc->subxids` under `ProcArrayLock`. | Full `CommitTransaction()`: executes **20+ `AtEOXact_*` subsystem hooks** (triggers, portals, relcache, catcache, sinval, smgr, 3-phase `ResourceOwnerRelease`, GUC pop, pgstat, `ProcArrayEndTransaction`), then restores parent transaction context. |
| **Reader Visibility Check (`HeapTupleSatisfiesMVCC`)** | **1 CPU instruction**: `HeapTupleHeaderXminFrozen(tuple)` checks `(t_infomask & HEAP_XMIN_FROZEN) == HEAP_XMIN_FROZEN` and returns `true` immediately. | Binary search in `XidInMVCCSnapshot()` + `TransactionIdDidCommit()` CLOG lookup + `SetHintBits()` buffer check. | Binary search in `XidInMVCCSnapshot()` + `TransactionIdDidCommit()` CLOG lookup + `SetHintBits()` buffer check. |

**CPU Verdict**: **Immediate Writes is the lightest by an order of magnitude**, executing ~30 lines of straight-line C on a buffer already in L1 cache. **`BEGIN AUTONOMOUS` is the heaviest on CPU** because it pays the full price of top-level transaction startup, $$O(N_{\text{backends}})$$ snapshot construction, and 20-subsystem transaction commit teardown on every invocation.

---

### 4.3 Memory Footprint (Backend-Private RAM & Shared Memory)

Let's sum the exact data structures allocated in backend memory (`src/backend/access/transam/xact.c`, `src/backend/utils/resowner/resowner.c`) and shared memory (`src/include/storage/proc.h`):

| Memory Component | 1. Immediate Writes (`immediate_global_insert_visibility`) | 2. `COMMIT SAVEPOINT` | 3. `BEGIN AUTONOMOUS` |
| :--- | :--- | :--- | :--- |
| **`TransactionStateData`** | `0 B` (uses existing transaction state) | `120 B` (allocated in `TopTransactionContext`) | `120 B` (plus saved parent state frame) |
| **Transaction `MemoryContext`** | `0 B` | **`8,192 B` (`8 KB`)** initial block for `CurTransactionContext` (`ALLOCSET_DEFAULT_SIZES`) | **`8,192 B` (`8 KB`)** initial block for autonomous `TopTransactionContext` |
| **`ResourceOwnerData`** | `0 B` | **`~544 B`** (`ResourceOwnerCreate`: 8 inline 16-item arrays + hash control) | **`~544 B`** (dedicated top-level `ResourceOwner`) |
| **`SnapshotData` & `TupleCidHash`** | `0 B` (shares parent snapshot & command counter) | `0 B` (shares parent snapshot; `ComboCid` entries added if modifying parent tuples) | **`~1.2 KB – 4.5 KB`** (`SnapshotData` + `xip` array sized to `max_connections`) + **`8 KB`** if `TupleCidHash` is created |
| **GUC / Metadata Overhead** | **`8 + 4 * N_tables` bytes** (`12 B` for 1 table, allocated once per `SET LOCAL`) | GUC stack save/restore entry if `SET LOCAL` used inside savepoint | Full GUC `tentativeVal` stack suspension & restore |
| **Shared Memory (`PGPROC` / `LOCK` / `PROCLOCK`)** | **`0 B`** | `4 B` in `MyProc->subxids.xids[64]` + `4 B` in `pg_subtrans` + `LOCALLOCK` entry for `SubTransactionId` | Requires secondary `PGPROC` / `GlobalTransaction` slot (**`~832 B`**) or duplicated `LOCALLOCK` + `PROCLOCK` entries (**`~160 B` per locked relation**) |
| **Total Memory per Invocation** | **`0 bytes` per insert** (`12 B` per `SET LOCAL`) | **`~8.8 KB`** backend RAM + subxid lock entries | **`~18 KB – 26 KB`** backend RAM + **`~1 KB+`** shared memory |

**Memory Verdict**: **Immediate Writes has a near-zero memory footprint (`12 bytes` per transaction, `0 bytes` per insert)**. `COMMIT SAVEPOINT` allocates **`~8.8 KB`** per savepoint block, and `BEGIN AUTONOMOUS` allocates **`~18–26 KB`** of backend and shared memory state per autonomous transaction.

---

### 4.4 Lock Footprint (LWLocks, Heavyweight Locks, and Self-Deadlocks)

Lock contention determines how well a feature scales across 64+ CPU cores and whether it can cause subtle production outages.

![three_ways_to_rollback_proof_writes_blog diagram 4](/assets/images/postsql_postgres/three_ways_to_rollback_proof_writes_blog_diagram_4.svg)

1. **Global Shared-Memory LWLocks (Cluster-Wide Scalability Bottlenecks)**:
   - **Immediate Writes**: Acquires **only `WALInsertLock` and `WALWriteLock`**. It never acquires `XidGenLock`, `ProcArrayLock`, `CLogControlLock`, or `SubtransControlLock`.
   - **`COMMIT SAVEPOINT`**: Acquires **5 cluster-wide LWLocks** per savepoint: `XidGenLock` (Exclusive), `SubtransControlLock`, `WALInsertLock`, `WALWriteLock`, `CLogControlLock` (Exclusive), and **`ProcArrayLock` (Exclusive)** (to remove the committed subxid from `MyProc->subxids` while the top transaction remains active).
   - **`BEGIN AUTONOMOUS`**: Acquires **5 cluster-wide LWLocks** per autonomous block: `ProcArrayLock` (Shared, during `GetSnapshotData`), `XidGenLock` (Exclusive), `WALInsertLock`, `WALWriteLock`, `CLogControlLock` (Exclusive), and **`ProcArrayLock` (Exclusive, during `ProcArrayEndTransaction`)**. Under high concurrency, `ProcArrayLock` and `XidGenLock` are two of the hottest LWLocks in PostgreSQL.

2. **Heavyweight Lock Manager (`LOCK` / `PROCLOCK` / `LOCALLOCK`)**:
   - **Immediate Writes**: Allocates **zero additional heavyweight locks**. The `RowExclusiveLock` on the target table belongs to the parent transaction (typically stored in the backend's lock-free fast-path array `fpRelId[16]`), and because the inserted tuple is immediately frozen (`HEAP_XMIN_FROZEN`), no transaction-ID lock (`XactLockTableWait`) is ever needed on the tuple.
   - **`COMMIT SAVEPOINT`**: Must insert a heavyweight `ExclusiveLock` on its `SubTransactionId` into the shared lock table (`XactLockTableInsert`) and remove it on commit. Worse, it faces the **Lock Ownership Dilemma**:
     - If relation and tuple locks acquired inside the savepoint are **transferred to the parent transaction** (standard PostgreSQL subtransaction behavior), then even though the savepoint's data is committed in CLOG, **the locks remain held until the outer transaction finishes**, blocking concurrent DDL or row updates.
     - If locks are **released at `COMMIT SAVEPOINT`**, the engine must carefully reference-count locks per subtransaction so it does not accidentally release a lock that the parent transaction *also* acquired earlier!
   - **`BEGIN AUTONOMOUS`**: Must maintain a **completely disjoint set of `LOCALLOCK` and `PROCLOCK` owners** from the suspended parent transaction. If the parent already locked 10 tables, the autonomous transaction cannot reuse those lock grants; it must acquire its own `PROCLOCK` entries (often spilling out of the 16-slot fast-path lock array into the shared-memory partition lock hash tables `LockHashPartitionLock`).

3. **Self-Deadlock Risk**:
   - **Immediate Writes**: **0% risk** (shares the parent transaction's lock space).
   - **`COMMIT SAVEPOINT`**: **Low risk** on relation locks, **but cannot modify a tuple already modified by the uncommitted parent** (because committing the newer tuple version while the older tuple version is still uncommitted breaks MVCC HOT/ctid chains).
   - **`BEGIN AUTONOMOUS`**: **High risk**. Because its lock space is completely isolated from the suspended parent transaction, if the parent transaction holds an `ExclusiveLock` on a tuple (via `UPDATE` / `SELECT FOR UPDATE`) or a conflicting table lock, and the autonomous transaction tries to touch that same tuple or table, **the single-threaded backend blocks waiting for its own suspended parent transaction—an instant self-deadlock**.

---

## 5. Summary: Where, When, and Why to Use Each

Each of the three mechanisms occupies a distinct sweet spot. Trying to use full Autonomous Transactions for a single-row 2PC prepare log pays unnecessary complexity and latency costs; trying to use Immediate Insert Visibility when you need to `UPDATE` an existing summary row simply won't work.

### Architectural Comparison Matrix

| Dimension | 1. Immediate Insert Visibility (`SET LOCAL immediate_global_insert_visibility`) | 2. Committing Savepoints (`COMMIT SAVEPOINT`) | 3. Autonomous Transactions (`BEGIN AUTONOMOUS ... COMMIT`) |
| :--- | :--- | :--- | :--- |
| **Granularity** | Individual `INSERT` / `COPY` statement (Tuple-level) | Subtransaction block (`SAVEPOINT sp ... COMMIT SAVEPOINT sp`) | Full independent transaction (`BEGIN AUTONOMOUS ... COMMIT`) |
| **When is it Crash-Proof?** | **Immediately upon completion of each `INSERT`** (`HEAP_XMIN_FROZEN` + `XLogFlush`) | **Only when execution reaches `COMMIT SAVEPOINT`** | **Only when execution reaches autonomous `COMMIT`** |
| **When is it Rollback-Proof?** | **Immediately upon completion of each `INSERT`** | **Only when execution reaches `COMMIT SAVEPOINT`** | **Only when execution reaches autonomous `COMMIT`** |
| **Supported DML** | `INSERT` only (`COPY FROM` included); later `DELETE`/`UPDATE` uses normal MVCC | `INSERT`, `UPDATE`, `DELETE` (on rows not modified by uncommitted parent) | Arbitrary `INSERT`, `UPDATE`, `DELETE`, `SELECT`, and nested savepoints |
| **Multi-Statement Atomicity?** | No (each `INSERT` is immediately frozen and permanent) | **Yes** (`ROLLBACK TO SAVEPOINT` undoes the block if an error occurs before `COMMIT SAVEPOINT`) | **Yes** (`ROLLBACK` undoes the entire autonomous transaction) |
| **Sees Parent's Uncommitted Writes?** | **Yes** (runs in the parent transaction's snapshot and command counter) | **Yes** (shares parent snapshot up to the savepoint) | **No** (uses an isolated snapshot; cannot see uncommitted parent rows) |
| **Self-Deadlock Risk with Parent?** | **None** (uses the current transaction's lock context) | **Low** (must guard against modifying parent-dirtied tuples) | **High** (independent lock space; touching a parent-locked row/table self-deadlocks) |
| **Extra XID / SubXID Consumed?** | **Zero** (reuses parent XID during index check, then stamps `HEAP_XMIN_FROZEN`) | **1 SubXID** per savepoint | **1 Top-Level XID** per autonomous transaction |
| **PostgreSQL Engine Complexity** | **Low** (~350 LOC in `heapam.c`, `nodeModifyTable.c`, `copyfrom.c`, `variable.c`) | **Medium–High** (`xact.c`, `subtrans.c`, `procarray.c`, `lmgr` lock transfer) | **Very High** (`xact.c` stack virtualization, `PGPROC` multiplexing, deadlock detector) |

---

### Decision Guide: Which One Should You Choose?

#### 1. Use `immediate_global_insert_visibility` when:
- **You need immediate crash-safety *before* the next line of code runs**: For example, recording a 2PC `<gid>` in a local table immediately before sending `PREPARE TRANSACTION '<gid>'` over the network, where waiting for a block-end commit or risking an unhandled exception later in the block is unacceptable.
- **You use the Insert-then-Delete ledger pattern**: You want the initial `INSERT` into a tracking table to be immediately frozen and crash-proof, and the cleanup `DELETE` at the end of the transaction to obey normal MVCC (committing only if the outer transaction commits).
- **Your workload is append-only (`INSERT` / `COPY`)**: Security audit logs, compliance trails, function error logs, or real-time queue/telemetry producers where rows are only ever inserted during the rollback-proof step.
- **You want zero subtransaction or XID overhead**: Because it freezes tuples in place right after index insertion and flushes WAL once per statement (or `COPY` batch), it never touches `pg_subtrans`, never allocates extra XIDs, cannot self-deadlock with the parent transaction, and still sees all of the parent transaction's uncommitted data.

#### 2. Use `COMMIT SAVEPOINT` when:
- **You need multi-statement atomicity or `UPDATE`/`DELETE` inside the rollback-proof unit**: For example, updating a status row *and* inserting a history row, where both must roll back via `ROLLBACK TO SAVEPOINT` if a constraint fails midway, or both must persist via `COMMIT SAVEPOINT` once the sub-step succeeds.
- **Your rollback-proof logic needs to read the parent transaction's uncommitted data**: Because a savepoint executes inside the parent transaction's snapshot, it can query rows just inserted by the parent transaction without foreign-key or visibility surprises.
- **Commit-boundary durability is sufficient**: You only need the writes to become crash-proof once execution reaches `COMMIT SAVEPOINT`.

#### 3. Use `BEGIN AUTONOMOUS` when:
- **You need a fully isolated transaction inside a procedure or trigger**: You are migrating complex Oracle (`PRAGMA AUTONOMOUS_TRANSACTION`) or DB2 procedural logic that performs multi-table `INSERT`/`UPDATE`/`DELETE` workflows, requires its own clean MVCC snapshot (such as `READ COMMITTED` inside a `REPEATABLE READ` or `SERIALIZABLE` parent), or uses its own internal `SAVEPOINT` and exception-handling trees.
- **You explicitly want snapshot isolation *from* the parent transaction**: The inner logic should only see globally committed data and must never accidentally read uncommitted dirty state from the caller.
- **Commit-boundary durability is sufficient and lock footprints are disjoint**: Your autonomous block modifies dedicated logging/metrics tables that the parent transaction does not lock, avoiding self-deadlocks until the autonomous `COMMIT` makes the changes durable.
