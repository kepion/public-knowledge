---
name: document-solution-pattern
description: Interview consultant then produce branded Kepion docs and journey walkthrough video
type: reference
classification: public
scope: public
repo: public-knowledge
---

AOS skill `document-solution` orchestrates Kepion solution documentation and walkthroughs: ask which deliverables to build each run (multi-select: Internal Word, Client Word, Client PPT, Journey video), interview the consultant (client context, value proposition, user journey, end products, content menu, sensitivity), then twin/enrichment → branded Office via `core/brand/kepion/` → optional journey video via `make-video` Path B.

**Why:** Structure lives in the twin; meaning lives with the consultant. Inventory-only packs and full-sitemap videos do not match how stakeholders use the app.

**How to apply:** Never invent journey or value-prop. Never silently build all four deliverables. Default content when docs are selected: member-count sizing, one continuous model×dim matrix (Word table + one PPT PNG), full sitemap, forms-by-module, revision dims with attributes, Account/Project hierarchies indented, honest variables/sets. Skip measure groups / integrations / env topology unless asked. Internal packs get pills + security/rule appendices; client packs get brand chrome without those dumps. Karaoke captions follow brand tokens and are baked into the recording. Journey video ≈ 8–12 screens on the revision path. Prewarm before intro narration.

Related skills: `system-documentation` (inventory feed), `make-video`, `kepion-solution-twin`, `generate-document`.
