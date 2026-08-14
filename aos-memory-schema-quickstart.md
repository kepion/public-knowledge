---
name: aos-memory-schema-quickstart
description: How to write an AOS memory file that routes to the correct scope
type: reference
classification: public
scope: public
repo: public-knowledge
---

Every memory is one markdown file with flat frontmatter (`core/templates/memory.md` in
the AOS repo). Ask two questions before saving: **how sensitive is it?**
(`classification`) and **who is it for?** (`scope`). Credentials and customer financials
are never stored — that's `restricted`, and the kernel refuses. A client-specific lesson
is `confidential` → the private plane (`~/.aos/memory`) or a company-scope repo. A
company practice useful to colleagues is `internal` → a company repo. A reusable pattern
with all specifics stripped is `public` → a public repo like this one — one line added to
its `INDEX.md`.

**Why:** the scope/classification split (audience vs sensitivity, enforced as
`rank(classification) <= rank(repo.ceiling)`) is what lets a company share a common brain
— and an ecosystem share best practices — without anyone's personal or
company-confidential context leaking outward.
**How to apply:** when in doubt, save to the private plane first; promote a sanitized
version to a company or public repo later (`scripts/promote-to-org.mjs --to <repo>`).
