# Online change playbook

Use this playbook for DDL, backfills, table moves, and partition maintenance on a database serving traffic. Verify syntax and lock behavior against the deployed PostgreSQL version and managed provider.

## Universal preflight

Define the exact objects, size and churn, dependent queries, write paths, foreign keys, triggers, replicas, expected lock, transaction eligibility, disk/WAL headroom, statement runtime, rollout window, abort signal, validation, and rollback.

Use a deployment-session `lock_timeout` that fails fast before joining a lock queue. Set `statement_timeout` only from a measured runtime budget. Rehearse on production-like cardinality and concurrency; a schema-only test is insufficient.

## Add an index

- Use a regular build for a new or empty table, an offline operation, or an accepted write-blocking window.
- Use `CREATE INDEX CONCURRENTLY` for a populated live table when writes must continue.
- Run a concurrent build outside a transaction block. Expect extra scans, longer runtime, and additional IO.
- Check catalog validity after failure or cancellation. A failed concurrent build can leave an invalid index that still adds maintenance overhead.
- Validate with representative `EXPLAIN` plans and production metrics. Remove a superseded index only after proving no important plan needs it.
- On partitioned tables, build concurrently on individual partitions and attach them to the parent index when the server version requires that pattern.

## Add constraints with low impact

- Add `CHECK` and foreign-key constraints as `NOT VALID`, then run `VALIDATE CONSTRAINT` separately. New writes are enforced while validation checks existing rows with a lower lock level than the initial blocking form.
- Build a unique index concurrently, then attach it as a unique or primary-key constraint where PostgreSQL supports the exact form.
- For `NOT NULL` on a large populated table, use an equivalent validated check to prove the invariant before setting `NOT NULL`; confirm version-specific lock and scan behavior.
- Index referencing foreign-key columns when parent updates/deletes or child joins require it.

## Add or transform a column

1. Add the column in a compatible form. Modern PostgreSQL can add a non-volatile constant default without rewriting existing rows; volatile defaults and type changes may rewrite the table.
2. Deploy code that tolerates both old and new states and writes the new representation.
3. Backfill in bounded, resumable, idempotent batches using a stable key range rather than deep `OFFSET` pagination.
4. Throttle from observed lock time, WAL, replica lag, IO, vacuum debt, and foreground latency.
5. Reconcile the invariant, validate constraints, switch reads, and only later remove the old representation.

Do not hold the whole backfill in one transaction. Commit progress, persist a cursor, and make retries safe.

## Move a very large table

Use expand/capture/backfill/reconcile/cutover/contract:

1. Create the target schema, indexes, and constraints without changing readers.
2. Capture every new insert, update, and delete through an application dual-write, transactional outbox/CDC, or carefully reviewed trigger. Preserve transaction ordering and idempotency.
3. Backfill stable key ranges with bounded transactions. Analyze the target after exceptional bulk loading.
4. Reconcile counts by range plus domain invariants and checksums. Investigate every mismatch.
5. Shadow reads or compare results before cutover. Freeze or strictly order the final delta if the capture mechanism cannot prove convergence.
6. Switch reads and writes behind a reversible release or flag. Monitor both correctness and load.
7. Keep the old table for the agreed rollback window; stop capture and remove old objects only after acceptance.

Triggers are not free: include all writers and mutation types, avoid recursion, account for added latency and WAL, test failure semantics, and ensure operational jobs cannot silently disable them.

## Partition deliberately

Require one or more concrete benefits: partition pruning for common predicates, cheap retention by detach/drop, maintenance isolation, or data placement. Ensure queries include the partition key in a pruneable form.

Plan future partition creation, a safe default-partition strategy, unique/primary-key constraints, foreign keys, per-partition indexes, statistics, and retention automation. Too many partitions increase planning and catalog overhead.

Prefer `DETACH PARTITION CONCURRENTLY` when supported and its restrictions fit. Attaching a preloaded partition with a matching validated `CHECK` can avoid a validation scan under a stronger lock.

## Recover space and index health

- Standard `VACUUM` makes dead space reusable inside the relation and normally allows concurrent reads/writes; it usually does not return space to the operating system.
- `VACUUM FULL` rewrites the table and takes `ACCESS EXCLUSIVE`; reserve it for an explicit downtime plan.
- Use `REINDEX ... CONCURRENTLY` for live index rebuilds when supported, then verify validity and redundant artifacts.
- Use `pg_repack` only if the managed provider supports it and its additional space, locks, triggers, and failure modes are understood.
- Prevent recurrence by fixing churn, long snapshots, worker capacity, and per-table autovacuum thresholds rather than scheduling endless rewrites.

## Use queue-style concurrency safely

Use a single atomic claim-and-update statement with `FOR UPDATE SKIP LOCKED` for queue-like work. Add deterministic priority, a unique job identity, idempotent handlers, lease owner and expiry, retry count, dead-letter policy, and a reaper for abandoned claims. Monitor starvation and accept that `SKIP LOCKED` is not a consistent general-purpose read.

## Primary sources

- [PostgreSQL 18: ALTER TABLE](https://www.postgresql.org/docs/18/sql-altertable.html)
- [PostgreSQL 18: CREATE INDEX](https://www.postgresql.org/docs/18/sql-createindex.html)
- [PostgreSQL 18: Table partitioning](https://www.postgresql.org/docs/18/ddl-partitioning.html)
- [PostgreSQL 18: REINDEX](https://www.postgresql.org/docs/18/sql-reindex.html)
- [PostgreSQL 18: SELECT locking clause](https://www.postgresql.org/docs/18/sql-select.html)
- [Parallel change / expand and contract](https://martinfowler.com/bliki/ParallelChange.html)
