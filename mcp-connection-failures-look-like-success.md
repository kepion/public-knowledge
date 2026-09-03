---
name: mcp-connection-failures-look-like-success
description: Connecting an MCP server to a real backend fails in ways that never announce themselves as connection failures — an unread credential var, a hung auth prompt counted as a leak, a launcher shim that breaks signal delivery, and an env loader that overrides its own spawner
type: reference
classification: public
scope: public
repo: public-knowledge
---

Four independent incidents across three MCP servers — a vendor's Azure DevOps server, a
SQL Server bridge, and a planning-platform server — produced the same class of bug: **the
connection was misconfigured, and every visible signal said something else.** None
surfaced as "authentication failed". They surfaced as a memory leak, a hung script, a
corrupted tenant, and a config that looked immaculate and had never once worked.

If you are wiring an MCP server to a real backend, these are the four checks worth
building in before you trust it.

## 1. A credential env var the package never reads fails as a LEAK, not an auth error

A vendor MCP server was registered with three entries, each passing a personal access
token in an env var named for the product. The package's own `auth.js` read two entirely
different names, and no `--authentication` flag was passed — so every launch fell through
to its `default:` branch, interactive OAuth, which opened a browser nobody would ever
complete and then **hung on a pending promise instead of exiting**.

The observable symptom was **216 orphaned processes and ~21 GB of committed memory**,
accumulating in per-session waves of ~36 over three days. The tokens were simultaneously
exposed in plaintext *and* completely unused. No error was ever logged, because from the
process's point of view nothing had failed yet — it was still waiting.

**The generalization worth keeping:** a launcher wrapper (`npx`, `uvx`, `cmd /c`)
explains why an orphan *survives* teardown; it does not explain why a process never
*exits* on its own. When a process count grows in per-session waves rather than by one or
two, look for a **blocking prompt** — interactive auth, a confirmation, a missing tty.
The wrapper is the accomplice, not the culprit.

**Checks:**
- Grep the installed package for the env var you are setting. `grep -o "process\.env\.[A-Z_0-9]*" dist/*.js | sort -u` takes ten seconds and is authoritative. A credential name that appears nowhere is being ignored.
- Read the auth module's `switch` and find out what its `default:` branch does. If the default is interactive, an unset or misnamed credential means *hang*, not *fail*.
- Prefer an auth mode with no secret in config at all. Most first-party servers accept a CLI-credential mode (`--authentication azcli` and equivalents) that reuses an existing SDK login; a config holding no token cannot leak one or misname one.

## 2. Never point config at a package-manager CACHE path

The obvious fix for a leaking `npx -y pkg@tag` entry is to skip the wrapper and invoke
the resolved entrypoint directly. The trap is *which* resolved path you use: an `npx`
cache directory (`_npx/<hash>/node_modules/...`) is keyed to the exact `pkg@tag` spec and
evaporates on `npm cache clean` or the next nightly resolve. It works today and silently
breaks later.

Install to a stable path you own and pin an exact version:
`npm install --prefix ~/.local/<tool> pkg@1.2.3`, then point config at that. Dropping a
floating `@latest`/`@next` tag also removes a registry round-trip from inside the host's
start-up timeout, and gets you off nightly builds you never chose.

## 3. On Windows, a `.bin` shim forces `shell: true`, which breaks signal delivery

A launcher that spawns a dev-runner via its `node_modules/.bin/` entry hits a `.cmd`
batch file, which Node has refused to exec directly since ~v20 (`EINVAL`). The natural
workaround — `shell: true` — inserts a `cmd.exe` between the launcher and the server. That
hop costs a process, emits a deprecation warning about unescaped arguments, and, the part
that actually matters, **breaks signal delivery: killing the launcher does not reliably
reach a grandchild through `cmd.exe`**, so orphaned servers accumulate across restarts.

Spawn the runner's **plain JS entrypoint** with `process.execPath` instead
(`node_modules/<runner>/dist/cli.mjs`), never the `.bin` shim. It is portable, needs no
shell, and keeps the process tree two deep so kills land.

Related: a server run through a transpiling dev-runner costs ~3 processes per instance
(launcher → runner CLI → runner preflight) versus 1 for a compiled entrypoint. That is
overhead, not a leak, but it multiplies by every registered entry — a reason to ship a
build step for a server you register more than once.

## 4. An env loader with `override: true` beats the spawner that was protecting you

A server called `dotenv.config({ override: true })` at startup. That is backwards for any
process a test harness or orchestrator spawns: the caller sets explicit environment
variables to *control* the child, and `override: true` lets a developer's on-disk `.env`
win over them.

The consequence was not subtle. An offline test spec spawned the server with unreachable
credentials so destructive tool calls could not touch anything real; the override
replaced them with the developer's `.env`, which pointed at live environments — and a
whole-row settings write broke sign-in repeatedly before anyone connected the two.

**Two fixes, and take both:**
- Default to `override: false` (dotenv's own default) so explicit spawner environment always wins.
- Add an explicit **sandbox flag** the caller can set — one that ignores the on-disk env file entirely *and* scrubs any numbered/extra connection variables that could reintroduce a real backend:

```js
if ((process.env.MYSERVER_SANDBOX ?? '').trim().toLowerCase() === 'true') {
  for (const key of Object.keys(process.env)) {
    if (/^MYSERVER_(URL|TOKEN)_\d+$/.test(key)) delete process.env[key];
  }
} else {
  dotenv.config({ path: resolve(__dirname, '../.env') });
}
```

Then have offline specs set it with an unreachable URL, and add a canary test that fails
if a supposedly-offline run can reach anything. Scrubbing the *numbered* variables matters
as much as ignoring the file: multi-backend servers often accept `_1`, `_2` suffixed
connections, and one survivor is enough to reach production.

## Multi-backend addressing: check whether it is per-call or per-process

Before you register N entries for N backends, find out which model the server uses — the
answer is in its argument parsing, not its docs:

- **Per-call routing.** The backend is an ordinary tool argument, so **one entry serves every backend**. See [[mcp-stateless-migration-pattern]] for how to retrofit this centrally without touching handlers.
- **Per-process, positional.** A `usage("$0 <organization> [options]")` with a required positional means the backend is fixed at process start and **one entry per backend is mandatory** — there is no consolidation to find, and looking for one wastes an afternoon. A *sub*-scope (project, database) is often still per-call, so one entry per org can still cover all its projects.
- **Per-process, but one package.** A launcher that accepts `--env <path>` lets one installed package serve several registered entries, each with its own credentials and target. This is the cleanest way to expose "same server, different environment" without duplicating an install.

**Identity is per-tenant, and a wrong identity does not look like a wrong identity.** A
cross-tenant backend (a partner's organization, a customer's directory) needs its own
credential session even when the config is perfect. A "user X is not authorized to access
this resource" style error, returned while the startup log shows the correct target and
tenant, means *wrong identity* — not wrong config. Enumerate the sessions your credential
chain can actually see (`az account list --all` and equivalents) before editing anything.
And vendor docs disagree with themselves here: one README's "prefer the hosted remote
server" recommendation coexisted with a docs page stating that the hosted server's auth
flow does not work from several major MCP clients at all. **Believe the page that matches
observed behavior**, and record which one that was.

## The test that would have caught all four

Every one of these passed a config review and failed in production. A config that *looks*
right proves nothing — three of the four had never successfully authenticated even once.
Drive a real handshake against the real backend and assert on process state, not on log
text:

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"<a_read_only_tool>","arguments":{}}}' \
  | timeout 75 node /stable/path/to/server.js <args> ; echo "exit=$?"
```

Four assertions, each catching a different failure above:

1. **Real data came back** — not just a successful `initialize`. A server can complete the handshake and be unable to authenticate a single call.
2. **It exited on its own** before the timeout. A server that needs to be killed is the hung-prompt bug (#1).
3. **Process count returns to baseline** afterwards. Count matching processes before and after; a non-zero delta is the orphan bug (#1/#3).
4. **The config holds no secret.** Grep it. If a credential is required at all, it belongs in a gitignored env file the launcher reads, never inline in a config that gets backed up, synced, and pasted into chat.

Assert on exit codes and process counts, never on output text — a success marker echoed
back inside an error message will match your grep. See
[[success-probes-must-not-match-text]].

**One more, learned the expensive way:** if you inspect a config that turns out to hold
plaintext credentials, those credentials are now wherever your inspection output went.
Treat them as compromised and rotate them, rather than concluding that removing them from
the file was sufficient. Deleting an entry does not revoke the token it contained.
