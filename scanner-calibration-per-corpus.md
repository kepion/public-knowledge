---
name: scanner-calibration-per-corpus
description: A secret scanner's patterns do not transfer between corpora — reusing commit-scanner regexes on a 500 MB conversation-transcript tree gave a ~99% false-positive rate; the three fixes (case anchoring, structural slug rejection, real JWT parsing) and why entropy barely helps
type: reference
classification: public
scope: public
repo: public-knowledge
---

Reusing a working secret scanner on a *different kind* of text is where secret detection
quietly stops working. Measured: patterns lifted verbatim from a repo pre-commit scanner and
run over ~500 MB of AI-agent session transcripts reported **695 API-key hits at roughly a 1%
true-positive rate**. After calibration the same corpus reported 153 restricted findings
across 47 files — all credible.

**Why:** a pre-commit scanner sees a few hundred staged, human-authored text files, and it
*hard-fails*, so over-triggering is the safe direction — a false positive costs one
annoyed developer. A conversation transcript is machine-written, enormous, and full of
base64 blobs, regex sources, tool output and pasted config. There over-triggering costs the
whole tool: a report with 690 false entries is indistinguishable from no report, and the six
real findings are never read. **Sensitivity and specificity trade differently per corpus, so
a scanner must be calibrated against the corpus it will actually run on.**

## The three defects, in order of yield

**1. Case-insensitive matching (≈66% of the false positives).** Every vendor credential
prefix is fixed-case *by specification* — `AKIA`/`ASIA` (AWS), `AIza` (Google), `ghp_`/`gho_`
(GitHub), lowercase `sk-` (OpenAI/Anthropic), `xox[baprs]-` (Slack). Compiling with `i` turns
each into a 4-letter substring search. **460 of 695 hits were case variants (`aKiAEIEB`,
`AkIABAwY`, `Sk-xHRMV`), and 432 of those sat inside base64 image data** — a long base64 blob
contains `akia` somewhere with near-certainty. Compile these patterns **case-sensitively**.

**2. Hyphen/underscore-permissive bodies.** `sk-[A-Za-z0-9_-]{20,}` matches any hyphenated
identifier that happens to start `sk-`. Real hits were ordinary tool and doc slugs of the
shape `sk-` + `search_<tool>_<name>`, `scoped-<topic>`, `encryption-<detail>`. (Deliberately
written as fragments: spelled out in full they are credential-shaped, and a pre-commit
scanner will flag this very paragraph — which is the point being made.) Fix structurally:
accept a known vendor infix (`ant-api03-`, `proj-`, `live-`), otherwise **reject bodies
containing 2+ all-letter segments of length ≥3** when split on `-`/`_`. Real credentials do
not contain English words.

**3. A vacuous JWT check.** `eyJ` **is** base64 for `{"`. So verifying that "the header
segment decodes to something starting with `{`" is true for *every* string the pattern
matches, by construction — the guard does nothing. Instead `JSON.parse` the decoded header
and require an object carrying `alg` or `typ`.

## Entropy is a much weaker filter than it looks

The obvious instinct — "real keys are high-entropy, slugs are not" — barely works at
credential length. Shannon entropy of `search-kepion-knowledge` is **~3.9 bits/char**, above
any floor a real key would also clear (real keys land ~4.5–6). There is no safe absolute
threshold separating them. Use entropy **only** to catch degenerate bodies (`sk-abababab…`,
floor ≈3.0) and let *structure* reject slugs.

Also cheap and high-yield: reject sequential and repeated-run fixtures
(`0123456789`, `abcdefghij`, `(.)\1{7,}`). Committed test fixtures like
`ghp_0123456789…` show up in real corpora.

## How to apply

- **Never reuse patterns across corpora without measuring.** Before trusting any count,
  sample the hits and compute the false-positive rate. A 97% FP rate is invisible in a
  summary line and obvious in 20 sampled fingerprints.
- **Keep the original scanner alone.** Calibrating for the noisy corpus and back-porting
  those restrictions into the pre-commit scanner would *reduce* its sensitivity where
  over-triggering was the correct bias. Two scanners, one shared pattern source, different
  compilation flags and validators.
- **Share the pattern source, not the compiled regex.** Export the pattern as a string and
  let each caller build its own instance: a module-level `g`-flagged regex shared across
  callers carries `lastIndex` and silently skips matches — a scanner reporting clean and
  being wrong.
- **Suppress false positives at match scope, not line scope.** A line holding both a
  placeholder and a real pasted token must still report the token. Line-level suppression is
  how a scanner gets talked out of its one true finding. (Whole-line suppression is right for
  exactly one case: the line is a regex *source* being discussed, detectable by regex
  metacharacter syntax that neither prose nor credentials contain.)
- **Never let the report quote what it found.** A findings file that pastes discovered tokens
  creates a second copy of the exposure somewhere more discoverable than the original. Emit a
  fingerprint (`sk-ant-a…(41 chars)`) plus a `file:record` locator, and make the redaction
  check *throw* rather than warn — a redaction failure is a policy violation, not a data
  quality warning.

Related: [[aos-memory-schema-quickstart]].
