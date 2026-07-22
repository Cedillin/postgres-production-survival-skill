---
name: postgres-production-survival
description: Diagnose, review, and harden production PostgreSQL systems using evidence-first operational triage, workload-aware query and schema design, connection budgeting, autovacuum and bloat analysis, safe concurrency patterns, and low-lock online migrations. Use for slow or unstable Postgres, connection storms, lock contention, long transactions, write-heavy tables, queue workloads, vacuum or wraparound risk, index and partition decisions, large backfills, production DDL reviews, and incident or scaling runbooks on self-hosted or managed Postgres.
---

# Postgres Production Survival

Use this skill to keep PostgreSQL changes measurable, provider-aware, and recoverable. Treat every fixed threshold and sizing formula as a hypothesis until it is checked against the actual workload.

## Preserve provenance and source priority

Treat this skill as an independent synthesis of Hatchet's [The startup's Postgres survival guide](https://hatchet.run/blog/postgres-survival-guide), the external [Supabase Agent Skills](https://github.com/supabase/agent-skills), and official PostgreSQL documentation. Do not imply affiliation with or endorsement by those projects.

Resolve conflicts in this order:

1. The behavior and constraints of the deployed PostgreSQL version and managed provider
2. Official documentation for that exact PostgreSQL version
3. `$supabase-postgres-best-practices` and, for Supabase projects, `$supabase`
4. Hatchet's experience-based heuristics

Use Hatchet for operational patterns, Supabase for detailed reusable rules, and PostgreSQL/provider documentation for authoritative semantics and version-specific behavior.

## Compose the companion skills

1. Use `$supabase-postgres-best-practices` as the detailed baseline for query, index, schema, locking, access-pattern, and monitoring rules. Read only the reference files relevant to the task.
2. If the system actually uses Supabase, also use `$supabase` and verify the current Supabase changelog, documentation, pooler mode, Data API exposure, privileges, and RLS behavior.
3. Apply this skill's operational guardrails and playbooks after the baseline rules. PostgreSQL documentation for the deployed major version and the managed provider's documented limits override generic examples.
4. If a companion skill is unavailable, continue with this skill and current primary documentation; state that the companion coverage was unavailable.

Do not copy numeric recommendations from the companion skill blindly. Pool sizes, connection limits, `work_mem`, autovacuum scale factors, row-count thresholds, and performance multipliers are workload- and provider-dependent.

## Route the task

- For a query, schema, or index review, use the companion best-practices rules and read [references/hatchet-synthesis.md](references/hatchet-synthesis.md).
- For a live slowdown, saturation event, lock incident, or vacuum concern, read [references/incident-playbook.md](references/incident-playbook.md).
- For production DDL, indexes, constraints, backfills, table moves, or partitioning, read [references/online-change-playbook.md](references/online-change-playbook.md).
- For a broad production-readiness review, read all three references and prioritize correctness, recovery, and observability before micro-optimization.

## Establish the operating envelope

Collect or explicitly mark as unknown:

- PostgreSQL exact version and managed provider
- primary/replica topology, failover model, and replication lag constraints
- CPU, memory, storage/IOPS, connection limits, and reserved headroom
- pooler product and mode; application pool sizes across every process and service
- critical queries, write rate, table and index sizes, churn, retention, and growth
- latency/error SLOs, maintenance window, rollback tolerance, and deploy mechanism
- available extensions, privileges, monitoring views, backups, and restore evidence

Distinguish verified facts, provider documentation, measurements, and inference. Never present a local schema inspection as a live production diagnosis.

## Work evidence-first

1. Define the symptom and time window. Separate latency, errors, saturation, contention, replication lag, storage growth, and maintenance debt.
2. Capture a read-only baseline before recommending changes. For incidents, capture timestamps and correlate database state with deploys and application traffic.
3. Form a ranked hypothesis list. Test the cheapest and safest discriminators first.
4. Stabilize before optimizing. Bound concurrency, stop connection churn, end application-created idle transactions, or remove the offending workload before redesigning SQL.
5. Change one causal variable at a time when practical. Define success, failure, rollback, and the observation window before execution.
6. Re-measure with the same workload and metrics. Keep an audit trail of what changed.

## Apply non-negotiable safety gates

- Default to read-only diagnosis. Do not run DDL, cancel sessions, terminate backends, change settings, vacuum manually, reindex, or enable extensions unless the user authorized the mutation and the exact target is resolved.
- Treat `EXPLAIN ANALYZE` as execution, not inspection. Start with `EXPLAIN`; use `ANALYZE` only when side effects, load, and production risk are understood. DML, triggers, and functions can still cause effects even if a surrounding transaction is rolled back.
- Keep transactions short and exclude network calls or human waits. Add application retry behavior for serialization failures, deadlocks, and transient disconnects where relevant.
- Use `lock_timeout` to make migrations fail fast instead of waiting behind traffic. Do not globally set aggressive timeouts without workload evidence.
- Do not kill an autovacuum marked to prevent wraparound. Identify blockers and transaction age before intervening.
- Avoid `VACUUM FULL` on a live table unless downtime and `ACCESS EXCLUSIVE` locking are explicitly accepted. Standard `VACUUM` normally coexists with reads and writes but can consume substantial IO.
- Use `SKIP LOCKED` only for queue-like or lease-like work where an inconsistent view is acceptable. Require idempotency, retry, lease expiry, and stuck-work recovery.
- Do not introduce partitioning merely because a table is large. Require a pruning-friendly access pattern or a concrete retention/maintenance benefit.

## Use workload-shaped rules

- Design indexes from observed predicates, join keys, ordering, limit, selectivity, and write cost. Confirm with plans and production-like cardinality.
- Prefer equality columns before the first range column in a multicolumn B-tree, then account for ordering. PostgreSQL 18 skip scans broaden possible use but do not erase selectivity and leading-column considerations.
- Accept sequential scans when a large fraction of a table must be read or the table is small. A sequential scan is evidence, not automatically a defect.
- Index referencing foreign-key columns when deletes, updates, or joins need them. Choose `CASCADE`, `RESTRICT`, `SET NULL`, or application deletion from domain semantics, not table size alone.
- Batch writes to reduce round trips, but cap batch size by latency, WAL, lock duration, parameter limits, replication lag, and retry cost. Use `COPY` for controlled bulk ingestion when supported.
- Budget connections across all application instances, workers, migrations, consoles, monitoring, and failover headroom. Pooling shifts concurrency; it does not remove database capacity limits.
- Tune autovacuum per high-churn table from tuple rate, table size, worker availability, IO capacity, freeze age, and observed completion. Duration alone is not a diagnosis.

## Produce an actionable result

Return:

1. **Verdict** — healthy, at risk, degraded, or incident; include confidence.
2. **Evidence** — exact plans, metrics, settings, locks, versions, or code paths used.
3. **Diagnosis** — causal chain and competing hypotheses, not a list of generic tips.
4. **Action ladder** — immediate stabilization, short-term correction, and long-term prevention.
5. **Execution contract** — exact scope, safety checks, rollout, validation, rollback, and owners when known.
6. **Unknowns** — what could materially change the recommendation.

For reviews, identify blockers separately from optional optimizations. For incidents, lead with the safest next action and include commands only when they are appropriate for the discovered version and provider.
