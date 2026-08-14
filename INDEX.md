# Public knowledge index

Public-scope AOS knowledge repo (`public-knowledge`): open to the whole ecosystem —
every adopter, every partner. Ceiling: public (sanitized; nothing company-internal).
One line per memory: `- [Title](file.md) — hook`. Index-first: add the line in the
same change that adds the file.

Every memory file carries flat frontmatter (`name`, `description`, `type`,
`classification`, `scope: public`, `repo: public-knowledge`) per the AOS
`core/templates/memory.md`.

<!-- entries -->
- [contributing-public-knowledge](CONTRIBUTING.md) — draft: what a public knowledge contribution must contain and how it is reviewed
- [aos-memory-schema-quickstart](aos-memory-schema-quickstart.md) — how to write a memory file that routes to the correct scope
- [kepion-true-twin-vs-delta-promote](kepion-true-twin-vs-delta-promote.md) — full-app twin via backup/restore vs object-level delta promote with compare/sync gates
- [voice-clone-privacy-boundary](voice-clone-privacy-boundary.md) — capability ships in the repo, voice biometrics never do
- [lifecycle-bookend-events-mask-completion](lifecycle-bookend-events-mask-completion.md) — a resumed session_start outranks the stop that ended the turn, and every safety net skips the row for its own good reason
- [skill-repo-catalog](skill-repo-catalog.md) — the vetted public skill repos worth consuming, the handful to enable first in each, and the license and shape gotchas that already cost someone time
- [adopting-skill-repos](adopting-skill-repos.md) — the five-step partner path from "what skills exist?" to skills running on your own hosts: discover, register, allowlist deliberately, sync attended, contribute back
- [aos-db-telemetry-schema](aos-db-telemetry-schema.md) — querying the local AOS telemetry db from a headless pass: node:sqlite, forward-slash paths, the real column names, and ISO-8601 timestamps
- [scanner-calibration-per-corpus](scanner-calibration-per-corpus.md) — a secret scanner's patterns don't transfer between corpora: case anchoring, structural slug rejection, why entropy barely helps, and why the report must never quote what it found
- [localhost-tools-drift-to-all-interfaces](localhost-tools-drift-to-all-interfaces.md) — an unauthenticated local dashboard bound to 0.0.0.0 is LAN-wide RCE; why every signal hides it, and how the test suite ends up asserting the vulnerability as a requirement
- [document-solution-pattern](document-solution-pattern.md) — interview consultant then branded Kepion Office packs + journey walkthrough (multi-select deliverables)
