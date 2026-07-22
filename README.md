# Postgres Production Survival

An evidence-first Agent Skill for diagnosing, reviewing, and hardening production PostgreSQL systems.

It combines the detailed SQL and schema guidance from [Supabase Agent Skills](https://github.com/supabase/agent-skills) with operational lessons synthesized from Hatchet's [The startup's Postgres survival guide](https://hatchet.run/blog/postgres-survival-guide) and current PostgreSQL documentation.

## Sources and composition

| Source | How this skill uses it |
|---|---|
| [Hatchet — The startup's Postgres survival guide](https://hatchet.run/blog/postgres-survival-guide) | Primary operational inspiration: workload-shaped schemas, short transactions, connection pooling, planner diagnosis, batching, autovacuum, bloat, `SKIP LOCKED`, partitioning, and large-table backfills. Its experience-based rules are retained where useful and qualified where they are too absolute. |
| [Supabase Agent Skills](https://github.com/supabase/agent-skills) | External companion rules for queries, indexes, schema design, locking, access patterns, monitoring, RLS, and Supabase-specific behavior. This repository does not copy or vendor those skills. |
| [PostgreSQL 18 documentation](https://www.postgresql.org/docs/18/) | Primary technical authority used to verify and correct version-sensitive behavior such as `CREATE INDEX CONCURRENTLY`, `VACUUM`, UUIDv7, multicolumn indexes, locking, and partition maintenance. |
| Managed-provider documentation | Final authority for hosted limits, supported extensions, pooler behavior, privileges, observability, and operational procedures. |

Source priority is: the deployed PostgreSQL version and provider documentation, official PostgreSQL documentation, Supabase's companion rules, then Hatchet's experiential heuristics. Fixed thresholds and performance claims are treated as hypotheses until measured against the real workload.

This is an independent synthesis and is not an official Hatchet, Supabase, or PostgreSQL project.

## What it covers

- production incident triage and causal diagnosis
- connection budgeting, pooling, and connection storms
- lock contention, long transactions, and queue concurrency
- query plans, indexes, statistics, and sequential-scan decisions
- autovacuum, bloat, XID/MXID risk, and maintenance health
- low-lock indexes, constraints, backfills, and table migrations
- workload-driven partitioning and retention operations
- explicit rollout, validation, rollback, and safety gates

## Install

Install the Postgres companion skill from Supabase:

```bash
npx skills add supabase/agent-skills --skill supabase-postgres-best-practices
```

Install this skill:

```bash
npx skills add Cedillin/postgres-production-survival-skill --skill postgres-production-survival
```

For projects that use the Supabase platform, also install its product skill:

```bash
npx skills add supabase/agent-skills --skill supabase
```

## Use

Invoke it explicitly:

```text
Use $postgres-production-survival to diagnose this Postgres risk and propose a safe, evidence-backed remediation.
```

Agents may also select it automatically for production Postgres incidents, scaling reviews, online migrations, vacuum problems, lock contention, or large backfills.

## Structure

```text
skills/postgres-production-survival/
├── SKILL.md
├── agents/openai.yaml
└── references/
    ├── hatchet-synthesis.md
    ├── incident-playbook.md
    └── online-change-playbook.md
```

## Design principles

- Diagnose read-only before mutating production.
- Treat fixed thresholds and sizing formulas as hypotheses.
- Verify behavior against the deployed PostgreSQL version and provider.
- Stabilize an incident before optimizing it.
- Make every production change measurable, resumable, and reversible.

## License

MIT
