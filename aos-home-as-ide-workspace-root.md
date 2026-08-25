---
name: aos-home-as-ide-workspace-root
description: Add the AOS operator home (~/.aos) as a folder in every multi-root IDE workspace, or agent-emitted links to memory, drafts and initiatives silently go nowhere
type: feedback
classification: public
scope: public
repo: public-knowledge
---

Add `~/.aos` (the AOS operator home) as a folder in **every** multi-root workspace you
open in your IDE. Without it, links the agent emits to memory, drafts, initiatives, and
learnings render as normal clickable links and do nothing when clicked.

**Why:** IDE markdown link handling resolves targets against *workspace roots*. The
operator home lives under the user profile (`~/.aos`), while sessions typically run in a
code repo somewhere else on disk. When those two trees share no common ancestor below the
drive root, **no relative path can reach the operator home** — the link must escape the
workspace, and the IDE has nothing to resolve it against.

Three fallbacks all fail, so don't spend time on them:

- **Deeper `../` walk-ups** — the arithmetic can be right and the link still dead. Correct
  depth is necessary but not sufficient; escaping the workspace is what breaks it.
- **`file://` URIs** — render as links, do nothing on click.
- **Bare absolute paths** — don't linkify at all.

The failure is *silent*, which is why it survives so long: nothing errors, the link just
goes nowhere, and it looks identical to a working one. An agent can "verify" the target
file exists on disk and still emit an unclickable link — file existence and link
resolution are different checks.

Note that an agent CLI listing the operator home as an accessible working directory does
**not** fix this. That governs the agent's file access; link resolution is the IDE's, and
the two are unrelated mechanisms.

**How to apply:** when setting up or joining a multi-root workspace, add `~/.aos` as a
folder alongside the code repos (in VS Code: File → Add Folder to Workspace, then save the
workspace so it persists). Keep a saved workspace file that pairs your repos with the
operator home rather than relying on an untitled workspace, which drifts. Verify once by
clicking an emitted memory link — not by checking that the file exists.

For headless or scheduled runs outside an IDE there is no workspace at all; emit
out-of-workspace paths as plain text there and say once that they aren't clickable.
