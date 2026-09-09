---
name: localhost-tools-drift-to-all-interfaces
description: A personal dev tool that binds 0.0.0.0 with no auth is remote code execution on the LAN, not a convenience — the drift is invisible because everything works, the UI keeps claiming "local only", and the test suite ends up asserting the wrong behaviour as a requirement
type: reference
classification: public
scope: public
repo: public-knowledge
---

A local dev dashboard defaulted its listen host to `0.0.0.0` behind an env-var override. It
served **dozens of routes with no authentication of any kind** — no token, no CORS policy, no
rate limit — and several of those routes could run code or write files on the host.

That combination is unauthenticated remote code execution for anyone who can reach the port,
running as the operator, with their credentials and client data in reach. It had been that way
for weeks.

## Why nobody notices

Every signal points the wrong way:

- **It works.** Loopback traffic is unaffected by a wider bind, so the tool behaves identically
  for the person who wrote it. There is no error, no warning, no degraded behaviour.
- **The wide bind was once deliberate.** It usually gets added for a real reason — reaching the
  UI from a phone. The reason expires; the default doesn't.
- **The UI kept asserting the opposite.** The footer read `local only`. Interfaces state
  security properties as decoration, and the claim is never re-checked against the listener.
- **The docs were accurate and useless.** The mitigation was documented — "set the host
  variable to limit it to this computer" — as an opt-in for the cautious, so the insecure path
  stayed the default and the safe one required knowing to ask.
- **The project already knew better.** Guidance *inside the same repo* advised binding
  `127.0.0.1` rather than `0.0.0.0`. Written guidance does not enforce itself.

## The test suite had encoded the wrong contract

The most useful detail. When the default was flipped, exactly one test failed — a test that
asserted an endpoint was served *on a LAN interface*. The suite was asserting the vulnerability *as a requirement*. Someone had confirmed phone
access worked and pinned it — reasonably, at the time. A test named for a capability
("serves on a LAN interface") silently becomes a test against the fix.

Fixing it means asserting both halves of the intended contract, not deleting the test:

```js
it('does NOT serve on a LAN interface by default', ...)       // the closed default
it('serves on a LAN interface when the env var opts in', ...) // the opt-in still works
```

Now the security property is enforced, and the escape hatch is proven rather than assumed.

## What to check on any local tool

1. `netstat -ano | grep <port>` — read the actual listener, not the config. `0.0.0.0` means
   every interface.
2. Check the host firewall. A wide bind behind a blocking firewall is latent; a wide bind with
   an **allow** rule for the runtime is live. Verify rather than assume.
3. Enumerate what the unauthenticated routes can *do*. Read-only telemetry is a different risk
   class from process spawning and file writes.
4. Grep the UI for safety claims ("local only", "private", "this device") and confirm each one
   is still true.

## The rule

**Loopback is the default; widening is an explicit, documented opt-in that says what it
costs.** Not "set this variable to be safe" — the safe state must be what you get by doing
nothing, because the wide bind's original justification always outlives its reviewer.

If the tool genuinely needs to be reachable from another device, put authentication in front
of it (an SSH tunnel or a reverse proxy that terminates auth) rather than exposing
unauthenticated routes and relying on network trust.
