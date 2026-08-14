---
name: lifecycle-bookend-events-mask-completion
description: Why interactive AOS turns hang forever as "running" — a resumed session re-emits session_start newer than the stop that ended the turn, and every independent safety net has its own good reason to skip the row
type: reference
classification: public
scope: public
repo: public-knowledge
---

A resumed terminal re-emits `session_start` against the **same** `session_id`. That event
lands *newer* than the `stop` which genuinely ended the previous turn. Any lifecycle logic
that asks "what is the newest event for this session?" then reads the reattach as if it
were work, and the finished turn never closes.

Symptom: a thread in the dashboard sits at `running` indefinitely, and the follow-up box is
greyed out with "the current turn is still running". `POST /api/threads/:id/messages`
returns **409** whenever the newest run in the chain is `running` or `queued`, so one
un-closed row makes the whole thread unusable — the work is long done.

**The trap is that every safety net independently declines the row, each for a correct
reason.** In the AOS run contract there are three, and a stuck turn slips all of them:

| Guard | Why it skipped |
|---|---|
| `closeInteractiveTurnsAtStop` | Looked only at the single newest event. Saw `session_start`, not `stop`. |
| `closeIdleInteractiveRuns` | Measured quiet time from `MAX(ts)`, which the fresh `session_start` had just refreshed — the run looked *active*. |
| `reconcileOrphanedRuns` | Deliberately skips `origin='interactive'` rows: they have no pid, so pid-liveness cannot judge them. |

No single guard is wrong. The gap is that they all treat "newest event" as a proxy for
"still working", and one event type breaks that proxy.

**The rule:** classify lifecycle events into *work* vs *bookends*. A bookend (`session_start`,
and any attach/detach signal) records a terminal connecting — it carries no work of its own
and must never stand in for activity. Both "did this end?" and "how long has this been
quiet?" have to look **past** bookends to the last event that represents real work.

Watch for the same shape anywhere a session can be resumed under a stable id: presence
tracking, idle timeouts, "last active" columns, lock expiry. Resume is not activity.

Two implementation notes that cost real debugging time:

- **Normalizing event names:** `String(e).toLowerCase().replace(/[^a-z]/g,'')` maps both
  `session_start` and `SessionStart` to `sessionstart`. A `[^a-z_]` character class *keeps*
  the underscore, so `session_start` never equals `sessionstart` and the filter silently
  matches nothing. Pick one and be sure which — the tests pass either way until a real
  snake_case event arrives.
- **Verify against events, not status.** The `runs.status` column is the thing that is
  wrong; it cannot diagnose itself. Read the session's raw event tail — a `stop` followed
  only by a much-later `session_start`, with nothing in between, is the signature.

Distinguish a genuinely idle terminal from a stuck one before mass-closing rows: a session
whose *only* event is `session_start` never did work and will age out of the normal idle
sweep on its own. It is not the same defect and does not need intervention.
