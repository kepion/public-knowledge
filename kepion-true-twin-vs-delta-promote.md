---
name: kepion-true-twin-vs-delta-promote
description: Sanitized pattern — full-app twin via SQL backup/restore vs object-level delta promote with compare/sync gates
type: reference
classification: public
scope: public
repo: public-knowledge
---

When promoting Kepion (or similar planning-app) configuration between environments, separate **full twin** work from **delta promote** work.

**Full twin:** SQL `COPY_ONLY` backup → restore under a new database name → rebind the app registration → recycle the app host (IIS/app pool) → deploy OLAP / process models → verify with both API compares and SQL row-count gates on dimension/hierarchy/fact tables.

**Delta promote:** run object compares → build a promotion manifest → `sync` with dry-run first → apply only after approval → redeploy/process → re-compare.

Do not mix lanes mid-flight. Do not promote fact/partition data with config changes unless that is an explicit signed-off step. After any connection-string rebind, recycle the host or the app will report unreachable until cache clears.

**Why:** Scripted row-by-row clones of application metadata are brittle across schema versions; backup/restore is the durable twin. Object-level sync is the durable delta.

**How to apply:** Choose the lane before touching a shared environment; always dry-run syncs; always pair API compares with relational count gates.
