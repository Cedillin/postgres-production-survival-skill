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

-- Column set is version-stable across supported majors. PG17 renamed the
-- dead-tuple accounting: on PG17+ you may also select
-- `dead_tuple_bytes, num_dead_item_ids`; on PG16 and earlier those columns do
-- not exist (use `max_dead_tuples, num_dead_tuples`). Selecting the version
-- wrong set errors mid-incident, so default to the stable columns below.
select pid, datname, relid::regclass as table_name, phase,
       heap_blks_total, heap_blks_scanned, heap_blks_vacuumed
from pg_stat_progress_vacuum;
```

Inspect XID age. Compare it with the actual server settings; do not use a memorized percentage as the sole decision criterion.

```sql
select datname, age(datfrozenxid) as xid_age
from pg_database
order by xid_age desc;

select c.oid::regclass as table_name,
       age(c.relfrozenxid) as xid_age,
       mxid_age(c.relminmxid) as mxid_age
from pg_class c
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
-- Replication slots: inactive slots and retained WAL are common wraparound and
-- disk-growth drivers. `wal_status = 'lost'` means required WAL was already
-- removed, so the slot cannot catch up.
select slot_name, slot_type, active, wal_status,
       xmin, catalog_xmin,
       pg_size_pretty(
         pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       ) as retained_wal
from pg_replication_slots
order by pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) desc nulls last;

-- Connected replicas and lag. Large write/flush/replay lag explains replica
-- staleness and, with feedback enabled, cleanup pinned on the primary.
select application_name, client_addr, state, sync_state,
       write_lag, flush_lag, replay_lag
from pg_stat_replication
order by coalesce(replay_lag, flush_lag, write_lag) desc nulls last;
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

- [PostgreSQL 18: Monitoring database activity](https://www.postgresql.org/docs/18/monitoring-stats.html)
- [PostgreSQL 18: Routine vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html)
- [PostgreSQL 18: Explicit locking](https://www.postgresql.org/docs/18/explicit-locking.html)
- [PostgreSQL 18: VACUUM](https://www.postgresql.org/docs/18/sql-vacuum.html)
