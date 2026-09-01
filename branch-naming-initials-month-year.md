---
name: branch-naming-initials-month-year
description: Feature branches are named <initials>_<mon><yy> — one predictable branch per operator per month, instead of per-task branch names nobody can guess
type: reference
classification: public
scope: public
repo: public-knowledge
---

Name a feature branch `<initials>_<mon><yy>`: the operator's initials, an underscore, the
short lowercase month, and a two-digit year. `xy_aug26`. `xy_sep26`. `xy_jan27`.

One branch per operator per month, not one per task. The branch is a *workspace* — whatever
that person is working on this month lands on it, and the pull request explains the content.

**Why:** per-task branch names (`agent/fix-crlf-spec-fence`, `feature/repoint-paths`) each
read fine alone and become unusable in aggregate — nobody can guess a collaborator's branch
without listing them, and the names drift in style the moment more than one person or agent
creates them. `<initials>_<mon><yy>` is derivable without looking: you always know whose it
is and how current it is, and a stale `_may26` branch is visibly stale on sight. Agents
generate branch names too, so a convention that has to be *remembered* rather than *derived*
will not hold.

**How to apply:**

- Create with the convention from the start: `git checkout -b xy_aug26`. Renaming later is
  the expensive path — see the warning below.
- Substitute your own initials. Everyone gets their own branch; two people never share one.
- Roll to a new branch when the month turns. Don't rename last month's — leave it as the
  historical record and open a fresh one.
- Keep `main` protected and merge through pull requests. The branch is where work
  accumulates; the PR is where it gets reviewed.
- Use a distinct name for genuinely long-lived work that spans months (a migration, a spike),
  so it isn't mistaken for a month branch that someone forgot to close.

**Renaming an existing branch closes its open pull requests.** GitHub's branch-rename API
does *not* retarget an open PR to the new name — the PR is closed, and it cannot simply be
reopened afterwards because its original head branch no longer exists. Recreating it from
the renamed branch works and the commits are never at risk, but the original PR number,
review history, and inline comments stay on the closed one. If a PR is open and its review
history matters, either merge it first and apply the convention to the next branch, or accept
a new PR number. Verify the outcome afterwards rather than assuming the rename was
transparent.
