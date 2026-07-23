# Production incident playbook

Use this playbook for a live or suspected PostgreSQL incident. Start read-only, timestamp every snapshot, and adapt catalog queries to the deployed major version and provider permissions.

## Contents

- [Classify before changing anything](#1-classify-before-changing-anything)
- [Capture a read-only snapshot](#2-capture-a-read-only-snapshot)
- [Build the causal chain](#3-build-the-causal-chain)
- [Stabilize safely](#4-stabilize-in-the-least-destructive-order)
- [Correct and verify](#5-correct-and-verify)

## 1. Classify before changing anything

Record the start time, affected operations, error classes, deploys, traffic changes, primary/replica roles, provider events, and whether recovery is already underway. Distinguish:

- connection exhaustion or churn
- lock queue or deadlock pressure
- long or idle transactions
- CPU, IO, memory, WAL, or storage saturation
- query-plan regression or stale statistics
- vacuum debt, bloat, XID/MXID risk, or replication-slot retention
- replica lag, failover, or network/provider trouble

## 2. Capture a read-only snapshot

Confirm identity and capacity first:

```sql
select now() as captured_at, version(),
       current_setting('max_connections') as max_connections;

select state, wait_event_type, wait_event, count(*)
from pg_stat_activity
group by state, wait_event_type, wait_event
order by count(*) desc;
```

Find old transactions and idle-in-transaction sessions. Avoid exposing full query text in shared artifacts when it may contain sensitive literals.

```sql
select pid, usename, application_name, client_addr, state,
       now() - xact_start as xact_age,
       now() - query_start as query_age,
       wait_event_type, wait_event,
       left(query, 300) as query_sample
from pg_stat_activity
where xact_start is not null
order by xact_start;
```

Find blocking relationships:

```sql
select a.pid as blocked_pid,
       pg_blocking_pids(a.pid) as blocker_pids,
       now() - a.query_start as blocked_for,
       a.wait_event_type, a.wait_event,
       left(a.query, 300) as blocked_query
from pg_stat_activity a
where cardinality(pg_blocking_pids(a.pid)) > 0
order by a.query_start;
```

Inspect table maintenance pressure:

```sql
select relid::regclass as table_name, n_live_tup, n_dead_tup,
       last_autovacuum, last_autoanalyze,
       autovacuum_count, autoanalyze_count
from pg_stat_user_tables
order by n_dead_tup desc
limit 30;

-- The query below uses columns shared by supported majors. Dead-tuple
-- accounting is version-specific: PG17+ exposes `max_dead_tuple_bytes`,
-- `dead_tuple_bytes`, and `num_dead_item_ids`; PG16 and earlier expose
-- `max_dead_tuples` and `num_dead_tuples`. Selecting the wrong set errors
-- mid-incident, so begin with the stable columns below.
select pid, datname, relid::regclass as table_name, phase,
       heap_blks_total, heap_blks_scanned, heap_blks_vacuumed
from pg_stat_progress_vacuum;
```

Inspect XID age. Compare it with the actual server settings; do not use a memorized percentage as the sole decision criterion.

```sql
select datname,
       age(datfrozenxid) as xid_age,
       mxid_age(datminmxid) as mxid_age
from pg_database
order by xid_age desc;

select c.oid::regclass as table_name,
       t.oid::regclass as toast_table,
       greatest(
         age(c.relfrozenxid),
         coalesce(age(t.relfrozenxid), 0)
       ) as xid_age,
       greatest(
         mxid_age(c.relminmxid),
         coalesce(mxid_age(t.relminmxid), 0)
       ) as mxid_age
from pg_class c
left join pg_class t on t.oid = c.reltoastrelid
where c.relkind in ('r', 'm')
order by xid_age desc
limit 30;
```

Inspect replication and slots. A stale replica (via `hot_standby_feedback`) or a
lagging/inactive replication slot can pin the cleanup horizon and drive WAL and
storage growth — the mechanism the classification and causal-chain sections
already ask you to consider. Run these on the primary; adapt to role and
provider permissions.

```sql
-- Replication slots: inspect both cleanup horizons and WAL state. The LSN
-- difference is distance from current WAL to `restart_lsn`, not verified
-- on-disk retained bytes, especially when `wal_status` is `unreserved` or
-- `lost`. A `lost` slot is no longer usable.
select slot_name, slot_type, active, wal_status,
       xmin, catalog_xmin, restart_lsn,
       pg_wal_lsn_diff(
         pg_current_wal_lsn(), restart_lsn
       ) as wal_distance_bytes
from pg_replication_slots
order by wal_distance_bytes desc nulls last;

-- `backend_xmin` shows the standby cleanup horizon reported by
-- `hot_standby_feedback`. LSN distance estimates replay backlog in bytes.
-- The lag intervals describe recent commit acknowledgement/visibility delay;
-- they are neither backlog size nor a catch-up ETA.
select application_name, client_addr, state, sync_state,
       backend_xmin,
       age(backend_xmin) as feedback_xmin_age,
       sent_lsn, write_lsn, flush_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) as replay_backlog_bytes,
       write_lag, flush_lag, replay_lag
from pg_stat_replication
order by replay_backlog_bytes desc nulls last;
```

If `pg_stat_statements` is already enabled, rank normalized statements by total time, mean time, calls, shared blocks read, temp blocks, and WAL for the incident window. Do not enable or reset it during an incident without authorization.

## 3. Build the causal chain

Correlate rather than merely list observations. Examples:

- deploy increases application pools → connections queue inside Postgres → CPU context switching and latency rise
- external call inside a transaction → row lock retained → blocking tree grows → pool exhausts
- bulk update creates dead tuples and WAL → replica lags and autovacuum competes for IO → plans degrade after stale statistics
- old transaction or replication slot pins cleanup horizon → dead space cannot be reclaimed → table and index bloat grow
- cardinality distribution changes → planner estimate diverges from actual rows → join or scan strategy regresses

## 4. Stabilize in the least destructive order

1. Stop or throttle the causal workload at the application, worker, or pooler edge.
2. Prevent new connection churn and reserve operator headroom; do not blindly increase `max_connections`.
3. Let short transactions drain. Canceling a query or terminating a backend requires explicit authorization and impact review.
4. Remove application-created idle transactions and fix their lifecycle. Never terminate an unknown session solely from age.
5. Protect the primary from optional reads, batch jobs, and excess worker concurrency.
6. For vacuum pressure, remove blockers and assess progress before tuning or manual vacuum. Do not interrupt anti-wraparound vacuum.
7. Use failover only under the provider's runbook and after checking replication state and data-loss tolerance.

## 5. Correct and verify

Define the before/after workload, p50/p95/p99 latency, error rate, throughput, lock wait, connection queue, resource saturation, replication lag, plan shape, and maintenance metrics. Observe long enough to cover peak traffic and at least one relevant maintenance cycle.

Add prevention at the causal boundary: bounded pools, timeouts, short transactions, retries with jitter, query-plan monitoring, per-table autovacuum settings, deploy checks, or capacity alerts. Document rollback and remove temporary mitigations.

## Primary sources

- [PostgreSQL 16: VACUUM progress reporting](https://www.postgresql.org/docs/16/progress-reporting.html#VACUUM-PROGRESS-REPORTING)
- [PostgreSQL 17: VACUUM progress reporting](https://www.postgresql.org/docs/17/progress-reporting.html#VACUUM-PROGRESS-REPORTING)
- [PostgreSQL 16: Replication monitoring](https://www.postgresql.org/docs/16/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION-VIEW)
- [PostgreSQL 16: Replication slots](https://www.postgresql.org/docs/16/view-pg-replication-slots.html)
- [PostgreSQL 18: Monitoring database activity](https://www.postgresql.org/docs/18/monitoring-stats.html)
- [PostgreSQL 18: Routine vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html)
- [PostgreSQL 18: Explicit locking](https://www.postgresql.org/docs/18/explicit-locking.html)
- [PostgreSQL 18: VACUUM](https://www.postgresql.org/docs/18/sql-vacuum.html)
