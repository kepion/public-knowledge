---
name: retrieval-eval-saturates-before-it-discriminates
description: A retrieval benchmark whose baseline already scores ~100% Recall@5 cannot rank anything — "no candidate won" then means the golden set is too easy, not that the models are equal; label the queries the incumbent gets WRONG, and treat a vendor-recommended prompt prefix as a hypothesis to measure rather than a free win
type: reference
classification: public
scope: public
repo: public-knowledge
---

Before swapping the embedding model behind a semantic-recall system, we built a golden set from
real logged queries and A/B'd four candidates against the incumbent through the production
retrieval pipeline. The harness worked. The **measurement design** was what nearly produced a
confident, worthless answer.

## The saturation trap

Eighteen labeled queries, five arms. Four of the five scored **100% Recall@5**. The decision gate
was expressed as "≥10 points of Recall@5 improvement" — and against a baseline already at ceiling,
**no candidate can ever gain a single point.** The report cheerfully concluded "no candidate
cleared the bar; keep the incumbent."

That conclusion was unfalsifiable, not empirical. The metric had no headroom.

Only **7 of 18 queries discriminated at all** between models; the rest were answered correctly by
everything, contributing nothing but a reassuring number. On the metrics that still had room —
Recall@1 and MRR — a real ordering appeared that Recall@5 had completely flattened:

| arm | R@1 | R@5 | MRR |
|---|---|---|---|
| incumbent | 89% | 100% | 0.847 |
| candidate A | **94%** | 100% | **0.889** |
| candidate B | 89% | 100% | 0.783 |
| candidate C | 67% | 94% | 0.724 |

**Why:** a golden set built by labeling what the current system *already returns* inherits the
current system's competence. Every query it answers well becomes an easy question. The set
measures agreement with the incumbent, and the incumbent scores perfectly on it by construction.

**How to apply:**
- Pick k so the **baseline lands well below ceiling** — if R@5 is saturated, gate on R@1 or MRR.
- Deliberately label the queries the incumbent gets **wrong**. Those carry all the signal; easy
  queries are ballast.
- Make the harness *say so*: emit a saturation warning when the baseline is at ceiling, and state
  how many points one query is worth (at n=18, one query = 5.6 points — so any difference under
  ~6 points is noise). A clean-looking table invites more confidence than a small sample earns.
- "No winner" is only a real finding once the set can *express* a winner.

## A vendor-recommended prefix is a hypothesis, not a free win

Several embedding models ship a recommended query prefix (`query: `, `search_query: `, an
instruct template). The incumbent's own model card recommended one **specifically for short
queries** — exactly the query shape in use — and the pipeline applied none. That looked like free
accuracy: no re-embed, no migration, only the query embedding changes.

Measured, it made retrieval **worse**: Recall@1 fell 89% → 83%, MRR 0.847 → 0.790.

**Why:** the prefix shifts the query vector into the region the model was instruction-tuned to
place queries — which only helps if the *documents* were embedded under the matching convention.
Applied to an index built without it, it adds a systematic offset between query and document
space. A recommendation from a model card describes the model's training setup, not your index.

**How to apply:** run any prompt-prefix change as its **own isolated arm** against the existing
index. It is cheap to test and, when it loses, it would otherwise have silently contaminated
every model comparison run alongside it.

## Two costs worth measuring instead of assuming

- **Full re-embed was assumed at ~30 minutes; measured at 1.2–2.2 minutes** for ~4,400 chunks on a
  consumer GPU. The plan had gated on that ceiling. At this corpus size it is non-binding, and
  accuracy is the only real gate. Measure the swap cost before letting it constrain the decision.
- **Latency is not uniform across candidates**: one otherwise-plausible model showed a **14x p95
  latency blowup** while its median looked normal. Report p95, not just p50 — a good median hides
  a tail that users feel.

## The log field that means the opposite of how it reads

The recall log recorded `floored` — read as a boolean, it made a healthy system look broken:
"98% of queries floored." It is a **count of weak candidates discarded** (typical value: several
hundred). A large number means the relevance floor is doing its job.

**How to apply:** before computing a rate over a log field, print raw sample values. A field named
like a flag may be a counter, and the resulting metric is not merely wrong — it is alarming, which
is worse. Pin the semantics with a regression test once you learn them.

Related: [[scanner-calibration-per-corpus]] — the same shape of error, where a metric tuned on one
corpus reports confidently and uselessly on another.
