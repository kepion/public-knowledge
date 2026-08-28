---
name: porting-derived-measures-between-olap-engines
description: Porting calculated measures from one OLAP engine to another rule-by-rule is unsafe because the engines disagree about which cells a rule applies to; a coordinate that comes from an attribute lookup is usually a loader problem rather than a calculation problem, and a ratio measure is only proven correct when its value VARIES up the hierarchy
type: reference
classification: public
scope: public
repo: public-knowledge
---

Rebuilding a planning application on a different OLAP engine means re-expressing its
calculated measures. The tempting approach — translate each rule one at a time, keeping the
source's structure — is unsafe for a reason that is invisible until the numbers are wrong: the
two engines disagree about **which cells a rule applies to**, not just about syntax.

## Cell-scope qualifiers do not survive translation

Some engines let a rule declare that it applies **only to leaf cells** or **only to
consolidated cells**. That qualifier *partitions* the cube: two rules that would otherwise
contradict each other can coexist, because no cell is ever in scope for both.

Engines whose assignment blocks have no equivalent partition apply both rules to overlapping
cells and resolve the contradiction by iterating. A faithful rule-by-rule port therefore
compiles, deploys cleanly, reports success, and returns inflated values.

**What made it visible:** a **leaf larger than its own parent**. That is structurally
impossible for a sum, so it is a reliable detector for this class of failure — much better
than eyeballing whether a total "looks big". In one measured case the inflation was ~79x, and
every intermediate step reported success.

**What did not rescue it:** reaching for the target engine's "data member" construct to
recover the leaf/consolidated distinction. Flat (non-hierarchical) dimensions have no data
member, and asking for one raises a hierarchy-appears-twice error. Some rules genuinely do not
port; the honest outcome is to leave them unbuilt, document why, and name the architectural
fix (usually: store as a fact what the source derived per-cell) rather than ship inflated
numbers.

## A data-driven coordinate is often a loader problem

The rules that look *most* unportable are often the easiest, if you move them out of the
calculation layer entirely.

A rule whose source coordinate comes from an **attribute lookup** — "read the cost cube at
whichever plant this organization's `Default Plant` attribute names" — has no clean equivalent
in a static assignment language. But nothing requires it to stay a calculation. Evaluate the
attribute **once per member at load time**, write the result as an ordinary fact, and the
calculation collapses to a plain sum over leaves with no cross-cube read at all.

In one migration this single reframing converted the largest category of "unportable"
statements into a loader change. **Check whether the dynamic part can be resolved to a static
fact before declaring a rule unportable.**

Two traps when resolving one, both of which require reading the source metadata rather than
trusting a name:

1. **The value you need may be a consolidation the source never stores.** A rollup member is
   computed on demand, so the binary/extract holds only its leaf components. You must
   reproduce the rollup yourself, which means parsing the dimension, not just the fact data.
2. **Watch for alternate rollups.** If the same leaves also hang under a second, parallel
   parent, summing every parent double-counts them. Only the primary path contributes. A
   dimension parser that reports one parent per member hides this completely — it must report
   *all* parents, with weights.

Also: parent-child hierarchies in the target may carry no **weight** column, in which case a
source rollup with a negative weight (a subtraction expressed structurally) cannot be
expressed structurally at all and must become an explicit calculation. That is a real
deviation from source; record it rather than letting it look like a faithful copy.

## A ratio measure is only proven correct when it VARIES

The verification that matters for a ratio (margin %, rate, average) is **not** that it matches
at one cell. It is that the value **changes as you go up the hierarchy**, and that at every
level it equals the ratio recomputed from *that level's own* numerator and denominator.

A percentage is not additive. If the ratio is being aggregated rather than re-evaluated, a
top-level total reads as several hundred percent — but a single-cell check passes happily,
because at a leaf the two behaviours agree.

Measured across four levels of one ported hierarchy: 30.72%, 31.06%, 35.43%, 24.75% — each
matching `(numerator - cost) / numerator` recomputed at that level to ~1e-14. **The assertion
must compare against a recomputed ratio, never a fixed number**, or it stops testing the thing
that can break.

Two adjacent habits worth keeping:

- **Assert against arithmetic computed inside the test, not values read back from the cube.**
  Comparing the cube to itself proves nothing. Recompute from the stored inputs.
- **Mutation-test each new assertion.** Perturb the expected input and confirm the check fails.
  A check that cannot fail is worse than no check, because it reports confidence.

## When a test fails after a correct change, suspect the test

A verification suite written against a small pilot subset can encode the subset's accidents as
requirements. One example: with a single leaf carrying data, every consolidation *equals* the
leaf, and an equality assertion looks like a strong correctness check. Load the full data and
that assertion fails — correctly, because consolidations now legitimately exceed the leaf.

The discipline is to **confirm the new behaviour independently before touching the test** —
here, verifying in direct SQL that two children summed to the parent to ten decimals — and
then replace the assertion with a **scale-independent** form that still catches the real
failure mode (monotonic non-decreasing, which a child exceeding its parent violates).
Weakening a test to green is the failure mode to avoid; generalizing one whose premise expired
is not.

## Report combinations, not marginals

When describing how much data a sparse source holds, report **distinct combinations**, not each
axis independently. "12 months, 7 products, 5 versions" reads as ~420 slices; the actual count
was **19**, concentrated so that the most-queried scenario had no data at all.

The marginal summary implied broad coverage, so an empty query result looked like a load defect
and triggered a diagnosis of a failure that had not occurred — the rows had loaded correctly and
the queried slice simply had no source data. **Before treating an empty result as a bug, verify
the source has data at that exact coordinate.** Empty output is the correct output for empty
input.

## Bulk fact loading: check the payload arithmetic first

A per-row API is the obvious way to load facts and often the wrong one. Row-by-row tool calls
that pass through an agent's context cost roughly 150–200 bytes per row *as serialized
arguments*; a few hundred thousand rows becomes tens of megabytes and millions of tokens, which
no amount of patience fixes. A small pilot load succeeds and tells you nothing, because the
constraint is volume, not correctness.

Bulk-load into a staging table and resolve labels to surrogate keys with a set-based insert.
The same data that would have taken hundreds of API calls loads in under a second. **Do the
size arithmetic before recommending the per-row path**, and check whether the platform
sanctions direct writes to the fact layer — many explicitly do, while still prohibiting direct
writes to metadata.

Validate before writing, inside a rolled-back transaction:

- labels that resolve to **nothing** (dropped rows — know exactly which and why),
- labels that resolve to **more than one** key (these silently fan out and *multiply* rows —
  the most dangerous case, because the load still "succeeds"),
- duplicate coordinates within the batch,
- collisions with rows already present.

Then check the predicted row count, the distinct-coordinate count, and one pinned slice before
committing. Note whether the import path **appends or upserts**: an append-only importer turns
every correction into a duplicated coordinate.

## Faithfulness beats tidiness

Sample and legacy applications contain genuine oddities: a forecast priced below cost, a
currency the source never converts, dimension members present in the fact data but absent from
the dimension definition. The instinct to clamp a negative margin to zero, or to add the
"missing" currency conversion, produces output that looks more sensible and is **less correct**.

Reproduce the source, and record each such call explicitly in the artifact — including the
evidence that the source really does behave that way (for a missing conversion: that the source
rules contain no conversion at all). A deviation that is documented is a finding; the same
deviation applied silently is a defect that surfaces later as a reconciliation gap nobody can
explain.
