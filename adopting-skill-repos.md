---
name: adopting-skill-repos
description: Partner how-to — discover, register, allowlist, and sync public skill repos using the catalog
type: reference
classification: public
scope: public
repo: public-knowledge
---

# Adopting skill repos — the partner how-to

How an AOS adopter (any partner, any operator) goes from "what skills exist?" to
skills running on their own hosts, without drowning in hundreds of options.

## 1. Discover

Read [skill-repo-catalog](skill-repo-catalog.md) — the curated repos with starter
picks. Looking for something specific? Search `catalog/skills-index.json` for skill
names and descriptions across every cataloged repo before cloning anything.

## 2. Register

Run the `consume-skill-repo` skill in your AOS session. For a cataloged repo it
prefills the remote, branch, and skills root. Registration always includes, catalog
or not:

- a **secrets scan** of the clone (`tools/scan-knowledge-repo.mjs`);
- a **per-folder license check** — licenses can differ per skill inside one repo;
- your review of each skill's content before first use — enabling a skill means an
  agent will follow it.

## 3. Allowlist deliberately

Nothing is enabled by default. Start from the catalog entry's `recommendedAllow`
(confirm each one), and grow the list one deliberate choice at a time. Never enable
a whole repo — the biggest cataloged repo has ~190 skills, and every enabled skill
is executable prompt content.

## 4. Sync

`npm run sync:skills` materializes your allowlisted skills onto your hosts and, on
later runs, pulls updates and prints what changed upstream — that change report is
your review surface. Pulls are **attended**: you run the sync, never a scheduler.
Toggle individual skills later from the dashboard Skills page.

## 5. Contribute back

Registered a repo that isn't in the catalog and proved worth it? Draft a catalog
entry (the `consume-skill-repo` flow offers this at the end) and propose it to this
repo. Keep it sanitized: public remotes only, no machine paths, no company-internal
detail.

**Why:** the allowlist-plus-catalog model is what keeps a shared ecosystem usable —
partners see three vetted repos with a handful of starter picks, not hundreds of raw
skills, and the safety boundary (scan, license check, attended pulls) never moves.
**How to apply:** follow the five steps in order; never skip the scan because a repo
"is in the catalog".
