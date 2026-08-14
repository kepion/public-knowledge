---
name: skill-repo-catalog
description: Curated catalog of public skill repos worth consuming, with starter-pick allowlists per repo
type: reference
classification: public
scope: public
repo: public-knowledge
---

# Skill-repo catalog

The curated list of public skill repos the ecosystem has vetted and found worth
consuming into an AOS. Start here instead of searching GitHub: each entry names the
repo, its license posture, the handful of skills worth enabling first, and the
gotchas that cost someone time already.

**How to use it:** pick a repo below, then run the `consume-skill-repo` flow — it reads
this catalog, prefills the registration, and proposes the starter picks as your initial
allowlist. Enabling everything is never the move; every enabled skill is prompt content
an agent will follow, so each one is a deliberate, reviewed choice.

Machine surfaces (both in `catalog/`):

- `catalog/skill-repos.json` — this catalog as structured data; the source of truth
  tooling reads.
- `catalog/skills-index.json` — GENERATED index: every skill's name, description, and
  license for each cataloged repo that was cloned when the index was built. Search it to
  find a skill *before* cloning anything. It is **committed here on purpose** — an index
  you can only build after cloning cannot help you decide what to clone; `generated` and
  each repo's `sourceSha` tell you how stale it is. Repos missing from it are named in
  its `gaps` array: absent means "not indexed yet", never "has no skills". Regenerate
  against your own clones with `npm run catalog:skills` in the AOS repo.

The catalog is advisory. Registration still runs the secrets scan and the per-folder
license check — a catalog entry is a recommendation, not a trust grant.

## Starter picks

### superpowers — engineering process

- **Repo:** https://github.com/obra/superpowers · MIT
- **What it is:** Jesse Vincent's core engineering process skills — the discipline
  layer (debug systematically, test first, verify before claiming done, plan before
  building).
- **Start with:** `systematic-debugging`, `test-driven-development`,
  `verification-before-completion`, `writing-plans`.
- **Gotchas:** none known.

### anthropic-skills — documents and tooling

- **Repo:** https://github.com/anthropics/skills · Apache-2.0 (examples) /
  source-available (document skills)
- **What it is:** Anthropic's official skills repo — the production document skills
  (docx/pdf/pptx/xlsx) plus example skills like `mcp-builder` and `skill-creator`.
- **Start with:** `docx`, `pdf`, `pptx`, `xlsx` if you produce documents;
  `mcp-builder`, `skill-creator`, `webapp-testing` if you build tooling.
- **Gotchas:** license varies **per skill folder** — check before enabling. The
  document skills are source-available: consume-in-place only, never republish.

### microsoft-agent-skills — Microsoft/Azure reference

- **Repo:** https://github.com/MicrosoftDocs/Agent-Skills · MIT (code) / CC-BY-4.0 (docs)
- **What it is:** MicrosoftDocs' curated Agent Skills — roughly one skill per
  Microsoft/Azure service.
- **Start with:** nothing by default. Search `catalog/skills-index.json` for the
  services you actually use and enable only those.
- **Gotchas:** ~190 skills — never enable broadly.

### knowledge-work-plugins — finance and business functions

- **Repo:** https://github.com/anthropics/knowledge-work-plugins · Apache-2.0
- **What it is:** Anthropic's business-function plugin bundles. The **finance plugin**
  speaks the planning/close language directly: reconciliation, variance analysis,
  close management, financial statements, journal entries, SOX testing. Sibling
  plugins cover sales, productivity, customer support, data, legal, marketing.
- **Start with:** the whole finance plugin — `audit-support`, `close-management`,
  `financial-statements`, `journal-entry`, `journal-entry-prep`, `reconciliation`,
  `sox-testing`, `variance-analysis` (all markdown-only, no scripts).
- **Gotchas:** marketplace shape — plugins sit at the repo root, each with its own
  `skills/`; register one entry per plugin with `skillsRoot: "<plugin>/skills"`.
  Skills reference MCP connectors you'll swap for your own stack. Example content in
  other plugins trips secrets scanners on fictional persona emails (known false
  positives; the finance plugin scans clean).

### financial-services — financial modeling benchmark

- **Repo:** https://github.com/anthropics/financial-services · Apache-2.0
- **What it is:** Anthropic's Claude for Financial Services — the highest-quality
  finance skills published anywhere: DCF/comps/LBO/3-statement modeling, Excel audit,
  deck QC, GL reconciliation and month-end close agents, partner-built LSEG and
  S&P Global verticals.
- **Start with:** browse the `financial-analysis` vertical
  (`plugins/vertical-plugins/financial-analysis/skills`) — the modeling and Excel
  skills work without paid data connectors.
- **Gotchas:** one vertical per registry entry (marketplace shape). Many skills assume
  paid market-data MCPs (FactSet, Morningstar...). Output is draft analyst work
  product — keep the qualified-professional-review disclaimer when client-facing.

### trailofbits-skills — security audit

- **Repo:** https://github.com/trailofbits/skills · CC-BY-SA-4.0
- **What it is:** Trail of Bits' professional security-audit methodology as skills:
  C/C++ and Rust review, semgrep rule authoring, supply-chain risk, and
  `agentic-actions-auditor` — AI-agent injection vectors in CI, directly relevant to
  anyone building MCP servers or agent workflows.
- **Start with:** the plugin matching your stack; `agentic-actions-auditor` for
  agent/MCP builders.
- **Gotchas:** ~40 plugins, each its own `plugins/<plugin>/skills` root — register
  per plugin. Share-alike license (derivative skill docs stay CC-BY-SA). Skip the
  smart-contract plugins.

### vercel-agent-skills — frontend and writing rulebooks

- **Repo:** https://github.com/vercel-labs/agent-skills · MIT
- **What it is:** Vercel's engineering-derived rulebooks — React performance,
  web accessibility/UX, and documentation-writing checklists.
- **Start with:** `react-best-practices`, `web-design-guidelines`,
  `writing-guidelines` — all platform-neutral.
- **Gotchas:** skip the Vercel deploy/optimize skills unless you deploy on Vercel.

### hubspot-admin-skills — CRM operations

- **Repo:** https://github.com/TomGranot/hubspot-admin-skills · MIT
- **What it is:** 37 HubSpot admin skills with the best safety engineering seen in a
  community repo: portal audit, enrichment, workflows-as-code export, sandbox
  self-test harness; account-type checks fail closed.
- **Start with:** `hubspot-audit`, `sandbox-self-test`, `workflows-as-code`.
- **Gotchas:** most skills bundle Python scripts — review per skill before enabling.
  Needs a HubSpot private-app token; test in a sandbox portal first. Single active
  maintainer — re-check health before broad rollout.

## Contributing an entry

Found a repo worth cataloging? Add an entry to `catalog/skill-repos.json` and a
section here — sanitized (public remotes only, no machine paths, no company detail)
— and propose it as a draft/PR. The `consume-skill-repo` flow offers this
contribute-back step automatically after you register a repo that proved its worth.

**Why:** the curation knowledge (which repos are good, what to enable, where the traps
are) otherwise lives only in each operator's machine-local registry — invisible to the
next adopter.
**How to apply:** read the starter picks before registering anything; treat
`recommendedAllow` as your first allowlist and grow it deliberately.
