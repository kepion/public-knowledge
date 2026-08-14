---
name: aos-db-telemetry-schema
description: How to query the local AOS telemetry db (<AOS_HOME>/data/aos.db) from a headless pass — node:sqlite pattern, real column names, and the SQL gotchas that keep re-costing dreams
type: reference
classification: public
scope: public
repo: public-knowledge
---

Querying `<AOS_HOME>/data/aos.db` (AOS telemetry) from a headless/scheduled pass. Don't
assume a `sqlite3` CLI exists — use Node 24's built-in module:

```js
const {DatabaseSync}=require("node:sqlite");
const db=new DatabaseSync("C:/Users/<you>/.aos/data/aos.db",{readOnly:true});
db.prepare("SELECT ... ").all();
```

Use **forward-slash** paths in the connect string. Backslash paths inside a bash `node -e`
heredoc get mangled (`\.aos\data` → escape errors, "unable to open database file").

Real column names — **always `PRAGMA table_info(x)` first**, these have burned three
overnight (Hermes dream) runs:
- `runs`: `id, created_ts, finished_ts, provider, initiative, board_item, board_plane,
  status, output` (NOT `started_at`).
- `sessions`: `session_id, first_ts, last_ts, input_tokens, output_tokens,
  cache_read_tokens, cache_creation_tokens, initiative` (NOT `started_at`).
- `events`: **has NO `type` column** — a query assuming one errors (`no such column: type`).

SQL gotchas:
- `node:sqlite` reads **double-quoted** strings as identifiers. `WHERE type="table"` fails;
  use single quotes `'table'`. Inside a bash `node -e` heredoc, escape as `\x27table\x27`.
- Timestamps are **ISO-8601 strings** (e.g. `2026-07-28T09:00:05.537Z`), NOT epoch
  milliseconds. They sort/compare lexicographically as-is; `new Date(r.created_ts)` parses
  them directly.

Reading signals: `runs.output` holds each headless run's full final message (fastest way to
see what a prior run found). `runs.board_item`/`board_plane` reveal board→run launches (E2E
failures show as `[failed]`). Only run-launched sessions get `initiative` set; interactive
sessions are `(none)` unless something tags them — the AOS session-start focus check exists
to close that gap.

The dashboard API/express server is **:5676** (`core/dashboard/server/index.mjs`, `AOS_PORT`);
:5675 is the vite client that proxies `/api` → :5676. Curl `http://127.0.0.1:5676/api/overnight`
only if a server is actually listening. The Overnight view reads the operator's `dream-log.md`
directly.
