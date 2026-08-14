---
name: contributing-public-knowledge
description: Draft contributor guidance for adding sanitized, reusable knowledge to the public AOS commons
type: reference
classification: public
scope: public
repo: public-knowledge
---

# Contributing public knowledge (draft)

This repository is the ecosystem-public AOS knowledge commons. Add a file only when it teaches a durable lesson another adopter can use without knowing who discovered it or where it was first observed.

This guide is a draft for iteration. It describes the contribution standard; the repository does not accept unattended or autonomous writes.

## Decide the destination first

| If the lesson is... | Put it... |
|---|---|
| personal to one operator, or needs private context | their private AOS memory |
| useful inside one company but not outside it | that company's knowledge repo |
| reusable by any adopter after all sensitive specifics are removed | this repository |

Only `classification: public` belongs here. Never add credentials, secrets, customer financial data, PII, client names, company-internal decisions, machine paths, hostnames, IP addresses, account identifiers, or details that make a person, client, or company identifiable. If the pattern cannot survive sanitization, do not publish it.

## What makes a contribution useful

Add one fact or pattern per file. It needs a recognizable problem, decision, pattern, or failure mode; evidence (a source link, executed check, reproducible observation, or plainly labelled inference); boundaries for security, compliance, or automation; and an actionable outcome.

Prefer an update to an existing file when the lesson refines the same fact. Do not add a case diary, raw transcript, long chat log, or organization-specific procedure.

## File format

Create a short kebab-case Markdown file in the repository root. Frontmatter must be flat; the AOS semantic indexer does not read nested metadata.

```markdown
---
name: <short-kebab-case-slug>
description: <one sentence that makes the file findable in recall>
type: user | feedback | project | reference
classification: public
scope: public
repo: public-knowledge
---

<one durable fact or pattern, with enough context to use it safely>

**Why:** <why this matters>
**How to apply:** <what a future operator should do>

## Evidence and limits

<source, check, or observation; state limits and exceptions>
```

Use precise language. Separate observed facts from recommendations. Link related memories with `[[other-memory-slug]]` when that connection improves retrieval.

## Contribution workflow

1. Read `INDEX.md` and search before writing; refine the existing file if it already covers the lesson.
2. Classify and sanitize before drafting. When uncertain, keep the source in private or company knowledge and ask for review rather than publishing.
3. Write the file using the template and add its one-line hook to `INDEX.md` in the same change: `- [Title](file.md) — what it teaches`.
4. From the AOS repository, run `node tools/scan-knowledge-repo.mjs --repo public-knowledge`.
5. Refresh local recall with `node scripts/reindex-memory.mjs` from the AOS repository.
6. Treat the change as a draft. Review the diff, then create a pull request or ask a maintainer to review it. The AOS never commits or pushes a knowledge repo for you.

The Git hook is a safeguard, not proof that content is safe. A passing scan does not make a client-specific lesson public.
