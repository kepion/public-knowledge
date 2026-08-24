---
name: mcp-stateless-migration-pattern
description: Making a large stateful MCP server stateless per the 2026-07-28 spec — inject the routing parameter centrally so no tool handler changes, and never let a deprecated state-setter become a silent no-op
type: reference
classification: public
scope: public
repo: public-knowledge
---

The MCP revision **2026-07-28** removes protocol-level sessions: SEP-2567 deletes the
`Mcp-Session-Id` header, SEP-2575 removes the `initialize` handshake, and list endpoints
may no longer vary per connection. Cross-call state is expected to travel as **explicit,
server-minted handles passed as ordinary tool arguments**. Stdio servers keep working with
process-lifetime state, but the spec calls that a portability liability.

This is what it actually takes to migrate a server with hundreds of tools built around an
implicit "current selection", and the two traps that cost the most.

## Central parameter injection beats touching every handler

A server with a "current X" concept usually has two halves already:

- the **entity** parameter (`appId`, `projectId`, `tenantId`) is often *already* accepted
  per call, resolved as `provided ?? sessionDefault`. If every resolve site takes an
  override, that half needs **no code change at all** — statelessness there is about
  *requiring* what is already optional, and rewording the descriptions.
- the **connection/route** parameter (which backend/server/environment this call targets)
  is usually the real gap, and it is the one people try to thread through every handler.

Don't thread it. If every tool registers through one wrapper (and in a well-built server
it does), inject there:

1. In the registration wrapper, merge an optional routing parameter into the tool's schema
   before finalizing it. Add an opt-out flag for tools where it would be a no-op.
2. Strip that parameter from the arguments before invoking the handler, so handler
   signatures and their inferred types are untouched.
3. Carry the value to the routing client via **AsyncLocalStorage** (or your language's
   equivalent task-local storage), and resolve per call:
   `explicit argument > session default > first configured`.

The result: hundreds of tools gain stateless addressing with zero handler edits, and one
place to reason about routing. Concurrency is safe because task-local storage is scoped to
a single call's async chain — two in-flight calls cannot observe each other's routing.

**Pin the opt-out list in a test.** Assert that every non-exempt tool exposes the injected
parameter and that the exempt set is exactly what you listed. Otherwise a new tool silently
misses routing, or an exemption gets added by accident.

**Watch for strict schemas.** If the server rejects unknown parameters (it should), the
new parameter must land server-side *before* any caller sends it. That forces the ordering:
ship the server first, then the callers. There is no gradual "ignored for now" rollout.

## A deprecated state-setter must never become a silent no-op

The tempting shortcut is to keep the old `use_x` tools as harmless stubs. That is the most
dangerous option available.

Downstream automation may have built **correctness guards** on top of the implicit state.
A real example: a test runner switched to a second entity for a control measurement, then
switched back, and *raised* if the switch-back failed — because otherwise every later case
would run against the control entity while being reported under the original. A no-op
version of that tool keeps the guard's happy path green while breaking exactly the
invariant it was written to defend. Results get silently misattributed.

Two modes, both honest:

- **Process-lifetime state available** (stdio): the deprecated tools keep genuinely
  working. The guard still functions on older callers.
- **Stateless deployment**: the state-setters return a **loud error** naming the parameter
  to pass instead; freeze the context object so an internal `set()` throws rather than
  quietly succeeding. Tools that *resolve* something useful (find-by-name) should still
  resolve, and report the id for the caller to pass explicitly — just not mutate.

## Migrate downstream in belt-and-suspenders order

Callers must work against both the old and new server for the length of the rollout. The
pattern that holds: **send the explicit arguments AND keep the old state-setting call**,
then drop the latter once every host runs the new build.

Where a client can read the server's own `tools/list` schemas, gate injection on what each
tool actually advertises. That single check makes one client compatible with both builds
and prevents blind injection into tools that legitimately take no entity parameter — most
servers have some **system-scope** tools where passing one is a validation error.

Sequence: server (additive) → executable callers → prose/instructions → trajectory or
integration assertions last, so nothing asserts behavior that has not shipped.

## Keep the tool listing static

Under the new spec, list results are cacheable and must not vary per connection. Two
consequences that are easy to miss:

- Descriptions cannot describe *current* state ("the active project"). Reword to name the
  mechanism instead ("the session-default project, stdio only; required in stateless
  deployments"). One static description per tool, identical in every mode.
- Prove it: build the server in each mode and assert the serialized listing is
  byte-identical.

Budget for the reword's blast radius. Tool RESPONSE text changes too ("no active project"
becomes "no session-default project"), and any test matching on a response substring breaks
— in ours, in two separate files, found days apart. Prefer matching the stable half of a
phrase over the whole sentence.

## The test suite will fight you, and it is worth listening to

Two failure modes surfaced repeatedly while migrating, both worth checking for before you
start:

**Environment-gated tests hide contract drift.** A response-format change ships green if
the assertions pinning the old format sit behind a `WRITE_TESTS`-style flag, or skip for a
fixture reason. A skipped test is an UNMEASURED test but renders identically to a pass.
Make the run report what it skipped and why, grouped by cause — and give the gated tier a
one-command entry point so it is exercised deliberately rather than never.

**`try { call(); expect(...) } catch { skip() }` is a test that can never fail.** This
idiom is common in suites that run against a shared or partially-built environment, and it
converts assertion failures into skips. Narrow it so the catch wraps only the *call* (which
may legitimately fail on a given instance) and the assertions sit outside it. Response
shape and internal self-consistency never depend on instance state, so they must be able to
fail. Where such blocks were hiding failures for us, the tests — not the product — were
wrong: each demanded a non-null primitive where the tool had deliberately adopted honest
nullable degradation (`number | null` meaning "could not read", paired with a field that
says so). A `typeof` assertion failing with `"object"` is the tell, since `typeof null` is
`"object"`.

## Diagnose before you repair

Late in the migration a batch of failures looked exactly like a drifted environment (a
cube-side 500 naming a missing measure). Reproducing them at a pre-change commit proved
only that the migration had not caused them — **not** that the environment was at fault.
Asking the system what it actually contained (one metadata query listing the cube's
measures) showed the environment was healthy and the tests referenced a literal that never
existed there.

"Reproduces at an older commit" means *not mine*, never *not a bug*. And the genuinely
useful comparison is the **failure-set diff**: sort the failing test names before and after
and `comm` them, which answers "did I introduce anything?" precisely, instead of reasoning
from a total that moves for several reasons at once.

## The SDK upgrade is a separable decision

Implementing statelessness in *tool design* is independent of speaking the new wire
protocol. If your clients still speak the older revision, staying on the 1.x-era SDK is
reasonable — provided the server logic is per-call stateless, the later SDK swap touches
only the transport entry points, not the tool surface. Record that as an explicit decision
where the server is constructed, or someone will "helpfully" upgrade both at once.
