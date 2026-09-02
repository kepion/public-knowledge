---
name: filename-guards-need-both-directions
description: A rule that matches files by NAME can be too broad and too narrow at once — a read-gate denied every value-free .env.example while allowing envs/cloud.env holding real connection strings; deny broadly and re-admit templates by negation, test must-allow cases beside must-deny, and treat a false positive as the early warning for a false negative
type: reference
classification: public
scope: public
repo: public-knowledge
---

A guard that enumerates one filename spelling leaves its neighbours open. Rules that match
by **name** rather than by **location** are the ones that can be wrong in both directions
simultaneously — and a deny-only test suite cannot see either failure, because it goes green
on a rule that blocks everything.

Four instances of the same bug in a single session (2026-09-02) — the first three each found
only because the previous one was fixed, and the fourth inside this very write-up:

1. **A read-blocking hook** anchored its pattern on a leading dot — `(^|/)\.env(\.|$)`. It
   therefore **denied** every value-free `.env.example` (files that exist precisely to be
   read, holding no values by construction) while **allowing** `envs/cloud.env` and
   `Sample.env`, the per-environment and template conventions that held real connection
   strings. Too broad and too narrow in one expression.
2. **Five repos' `.gitignore`** files listed spellings individually — `.env`, `.env.prod`,
   `envs/*.env`. A stray `test.env` was exposed in all five.
3. **After** widening to `.env.*` and `*.env`, a backup written to `envs/cloud.env.bak-<ts>`
   *still* slipped through: the filename ends in `.bak-<ts>`, not `.env`. That needed
   `*.env.*` as well.

## The pattern

**Deny broadly, then re-admit the safe cases by negation** — never enumerate the dangerous
spellings, because the one you forget is the one that leaks:

```gitignore
.env
.env.*
*.env
*.env.*
!.env.example
!*.env.example
```

For a regex guard, anchor an exception to the **filename segment**, immediately after the
path separator. Placed after the alternation it silently fails, because an earlier arm can
match at an offset where the lookahead is satisfied by a different part of the string:

```js
// exception evaluated against the filename segment only
/(^|\/)(?!.*\.(?:example|sample|template)$)(?:\.env(?:\.|$)|[^/]*\.env(?:\.|$))[^/]*$/
```

Note the `(?:\.|$)` on **both** arms. Instance 4: an earlier draft of this very note ended
the second arm at `[^/]*\.env$`, re-creating instance 3 — `envs/cloud.env.bak-<ts>` does not
*end* in `.env`, so it sailed through. The same hole was still live in the hook from instance
1, which had been declared fixed. Both were caught by running the table against the published
text, not by re-reading the pattern.

The lesson reproducing itself inside the write-up of the lesson is the argument for the
table. Verify the artifact you are shipping — extract the pattern from the published file and
run the cases against *that*, because the version in your head is not the version a reader
copies.

Recognise a template only by an explicit `example` / `sample` / `template` suffix. A bare
`Sample.env` should stay **denied**: no filename rule can distinguish it from a real one, and
that is the safe direction to be wrong in.

## Why a deny-only suite misses this

Assertions that a guard blocks secrets all pass on a rule that blocks *everything*. The
must-**allow** cases are what pin the other edge. Write both tables, and watch the new case
fail before it passes — a test that has never failed has not been shown to test anything.

**The false positive is the cheap early warning for the false negative.** The hole above
surfaced only because the over-broad half blocked an ordinary question ("do these
`.env.example` files still represent the right examples?"). Treat a guard that blocks
something harmless as evidence the rule is mis-shaped, not as an acceptable cost — the same
mis-shaping is usually letting something real through.

Related: [[scanner-calibration-per-corpus]] — the sibling failure, where a secret scanner's
patterns are tuned to the wrong corpus and its PASS proves nothing.
