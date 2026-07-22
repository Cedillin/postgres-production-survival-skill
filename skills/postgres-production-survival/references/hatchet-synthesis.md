# Hatchet guide synthesis

This reference distills the transferable operational lessons from Alexander Belanger's “The startup's Postgres survival guide” (Hatchet, 2026-07-22), then tightens them against PostgreSQL 18 documentation. It complements, rather than duplicates, `$supabase-postgres-best-practices`.

## Keep these principles

- Shape the schema around real read and write paths before it becomes expensive to change.
- Use primary keys, timezone-aware timestamps, appropriate types, and constraints that encode real invariants.
- Make indexes match filters, joins, ordering, and pagination. Account for their write, vacuum, memory, and storage cost.
- Keep write transactions short and lock only the rows required for the invariant.
- Prefer additive expand/contract changes. Ask which lock a migration takes and for how long.
- Reuse long-lived connections through an application pool and, where appropriate, an external pooler.
- Inspect planner estimates and actual rows rather than guessing. Refresh statistics after exceptional bulk changes.
- Batch high-volume writes to amortize round trips and server overhead.
- Treat autovacuum, dead tuples, index health, freeze age, and transaction age as production concerns.
- Use partitioning when pruning or data lifecycle operations provide a measurable benefit.
- Use bounded, idempotent batches plus write capture and reconciliation for very large table moves.

## Replace these useful simplifications with precise rules

| Simplification | Production rule |
|---|---|
| Reads either use an index or sequentially scan | Plans also include bitmap, index-only, parallel, and many join strategies. Judge the complete plan, buffers, row estimates, loops, and elapsed time. |
| A sequential scan is slow | It is often optimal for small tables or queries returning a large fraction of rows. |
| Inner joins should use primary keys | Join on the domain relationship; index the relevant keys when cardinality and workload justify it. Primary or unique keys are common, not mandatory. |
| Put `ORDER BY` columns last in every compound index | Put leading equality predicates first, then the first range dimension, and design remaining key direction for the exact filter/order/limit pattern. Validate with `EXPLAIN`. |
| Always use `CREATE INDEX CONCURRENTLY` | Use it for populated live tables when write availability matters. It takes longer, does more work, cannot run inside a transaction block, and can leave an invalid index after failure. Regular builds can be right for new, empty, offline, or maintenance-window tables. |
| Run migrations in transactions | Prefer transactional DDL when supported, but split operations that prohibit transactions, including concurrent index builds. Make the overall rollout resumable. |
| An autovacuum running for an hour means bad tuning | Duration depends on table size, IO, cost delay, blockers, and progress. Inspect progress, freeze age, dead tuples, oldest transactions, worker pressure, and provider metrics. |
| Partition tables after a fixed row count | Partition for pruning, retention, maintenance isolation, or data placement. Size relative to memory and access pattern matters more than one row threshold. |
| UUIDv7 requires an extension | PostgreSQL 18 includes `uuidv7()`. Older versions may need a provider-supported extension or application generation. |

## Correct companion examples when necessary

- Standard `VACUUM` does not normally block `SELECT`, `INSERT`, `UPDATE`, or `DELETE`; `VACUUM FULL` takes `ACCESS EXCLUSIVE` and rewrites the table.
- `work_mem` is consumed per eligible plan operation and can be multiplied by parallel workers; it is not simply `work_mem × max_connections`.
- Pool sizing formulas are starting hypotheses, not configuration rules. Measure throughput, queueing, memory, IO, and tail latency across all clients.
- PostgreSQL does not auto-index referencing foreign-key columns. Index them based on join and parent-update/delete paths, and make cascading behavior a domain decision.
- A claimed speedup such as “10×” is workload evidence from one system, not a portable promise.

## Additional production lessons

- Add `lock_timeout` to deployment sessions so a migration fails rather than unexpectedly waiting and then blocking a queue of traffic.
- Watch idle-in-transaction sessions: they retain snapshots and can hold locks while doing no work.
- Design queue claims with `FOR UPDATE SKIP LOCKED` only when skipped rows and inconsistent visibility are acceptable; add lease expiry and recovery.
- Reconcile large table moves by invariants, counts by stable ranges, and sampled or full checksums. A successful copy command is not proof of equivalence.
- Treat triggers used for migration capture as production code: cover inserts, updates, deletes, retries, ordering, and every writer.
- Reserve connection and resource headroom for failover, migrations, vacuum, monitoring, and operators.

## Primary sources

- [Hatchet: The startup's Postgres survival guide](https://hatchet.run/blog/postgres-survival-guide)
- [PostgreSQL 18: Routine vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html)
- [PostgreSQL 18: CREATE INDEX](https://www.postgresql.org/docs/18/sql-createindex.html)
- [PostgreSQL 18: Multicolumn indexes](https://www.postgresql.org/docs/18/indexes-multicolumn.html)
- [PostgreSQL 18: Indexes and ORDER BY](https://www.postgresql.org/docs/18/indexes-ordering.html)
- [PostgreSQL 18: Table partitioning](https://www.postgresql.org/docs/18/ddl-partitioning.html)
- [PostgreSQL 18: UUID functions](https://www.postgresql.org/docs/18/functions-uuid.html)
- [PostgreSQL 18: SELECT locking clause](https://www.postgresql.org/docs/18/sql-select.html)
