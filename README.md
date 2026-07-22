# Postgres Production Survival

An evidence-first Agent Skill for diagnosing, reviewing, and hardening production PostgreSQL systems.

It combines the detailed SQL and schema guidance from [Supabase Agent Skills](https://github.com/supabase/agent-skills) with operational lessons synthesized from Hatchet's [The startup's Postgres survival guide](https://hatchet.run/blog/postgres-survival-guide) and current PostgreSQL documentation.

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
