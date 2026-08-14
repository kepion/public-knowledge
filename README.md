# public-knowledge

Sanitized, ecosystem-wide knowledge from an **Agentic Operating System (AOS)** — patterns,
gotchas, and machine-readable catalogs that are useful to any adopter or partner, not just
the operator who wrote them.

## Value proposition

Hard-won operational lessons usually die in one person's notes or one company's wiki. This
repo is the AOS answer: the **public plane** of a three-scope knowledge model, where a
lesson that survives sanitization gets published once and then recalled by *every* AOS in
the ecosystem — a shared memory that compounds across adopters instead of being relearned
per operator. Everything here is cleared for an unrestricted audience by design; anything
company-internal or personal lives in a different repo that is never published.

| Scope | Audience | Published? |
|---|---|---|
| `private` | one operator, one machine | never |
| `company` | exactly one company | never outside it |
| **`public`** | **the whole ecosystem** | **yes — this repo** |

The invariant is `rank(classification) <= rank(repo.ceiling)`. This repo's ceiling is
`public`, so every file here carries `classification: public`.

## What's in this repo

Start with [`INDEX.md`](INDEX.md) — one line per memory, with a hook describing what it
actually teaches. Read the index, then open only the files that match your question.

Each memory is a single markdown file with flat frontmatter (`name`, `description`, `type`,
`classification`, `scope`, `repo`) followed by the fact, and — for anything actionable —
`**Why:**` and `**How to apply:**` lines. New to the format? Read
[`aos-memory-schema-quickstart.md`](aos-memory-schema-quickstart.md) first.

### Machine surfaces

`catalog/` holds two JSON files meant to be consumed by tools, not just read:

- **`skill-repos.json`** — the vetted public agent-skill repos worth consuming, with the
  license and shape notes that matter before you adopt one.
- **`skills-index.json`** — a searchable index of the skills inside those repos, so you can
  search *before* cloning anything.

`skills-index.json` is **committed rather than regenerated per-adopter** on purpose: the
generator enumerates from local clones, so a fresh adopter regenerating it would get an
empty index — and search-before-clone is exactly the moment you have no clones. Staleness
stays readable via the `generated` date and per-repo `sourceSha`. Its `gaps` array names
cataloged repos that were not cloned when the index was built, so an absent skill reads as
"not indexed yet", never "no skills exist".

## Install

No build step — this is a content repo:

```powershell
git clone https://github.com/Kepion/public-knowledge.git C:\git\Kepion\public-knowledge
```

To wire it into an AOS so `recall_memory` searches it, register it from an AOS session with
the `register-knowledge-repo` skill (scope `public`, index `INDEX.md`). The registration
lives in `~/.aos/knowledge-repos.json` on your machine; nothing in this repo changes.

## Usage

- **As a human:** read [`INDEX.md`](INDEX.md), open the files that match your question.
- **As an AOS:** once registered, the `aos-recall` MCP server includes this repo in every
  `recall_memory` search automatically. Writes from an AOS session land as **drafts** — a
  human reviews and commits; no agent commits or pushes here.
- **As a tool author:** consume `catalog/skill-repos.json` and `catalog/skills-index.json`
  directly.

### Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The bar for landing a file here is that it
**survives sanitization and still teaches something**. If stripping machine paths,
hostnames, client names, and personal details leaves nothing useful, it belongs in a
private or company-scoped repo instead.

Never commit credentials, secrets, customer financial data, or PII — to this repo or any
other. Store a pointer to a credential manager instead of the value.

## License

Content is offered under [CC BY 4.0](LICENSE) — reuse and adapt it freely, with attribution.
