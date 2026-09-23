---
name: planning-app-model-vs-application-rules
description: Sanitized pattern — a rule that must appear under a specific model has to be saved through the model-scoped save call; the application-scoped one silently writes a different object type with no model binding, and the API read-back hides it
type: reference
classification: public
scope: public
repo: public-knowledge
---

Planning applications (Kepion and similar) commonly expose **two different rule stores** behind
two different RPCs. Choosing the wrong one fails silently and plausibly: the save returns a real
rule id, a read-back returns exactly what you submitted, and the rule simply never appears where
the user is looking.

| Save call | Writes | Type | Model binding | Appears under Models > *model* > Rules |
|---|---|---|---|---|
| application-scoped (`SaveDataRules`) | action-rule object | `ACTION` | always **0** | **no** |
| model-scoped (`SaveModelRules`, takes `modelId`) | SQL / MDX rule object | `SQL` / `MDX` | the model | **yes** |

The two objects are not variations of one thing. A model-attached SQL rule is roughly 1 KB of
XML carrying a `<Definition>` plus parameters. An application-level action rule is a component
pipeline — data source → join transform → insert action → publish-to-SP — running to 30 KB of
serialized DataContract XML. The second is not something to hand-author; wrappers around these
APIs typically say so explicitly.

## The failure mode worth remembering

Passing a SQL rule kind to the **application-scoped** save does not produce a SQL rule. The
server accepts it and writes an action rule whose `<Definition>` happens to contain the T-SQL,
with an empty component list and no model binding. Then the matching read call echoes the
submitted definition back **from cache**, so the read-back "confirms" a rule that the stored XML
shows is an empty shell.

**Verify a metadata write against the store, not against the API read.** Check the root element,
the type, and the model id in the underlying table. Corroborating signals that the write went to
the wrong store: parameter and post-processing collections come back empty, and the object's own
internal `<ID>` stays `0` while the table row carries a real id.

## The wrapper is likelier to be wrong than the server

The blocking restriction here turned out to live in the MCP wrapper, not the product: its
model-scoped tool declared `rules: z.array(MdxRuleSchema)` and hardcoded the MDX serializer,
so it refused SQL rules outright. The underlying RPC took an arbitrary rules array, and a SQL
serializer already existed in the same file for the application-scoped tool. Widening the schema
to a union and dispatching on `kind` was about 20 lines, and the server accepted it first try.

The tell was in the data: an application whose rules had been built through the product's own UI
already contained SQL-type rules with a non-zero model id. **When a wrapper refuses a shape the
product's own UI clearly produces, suspect the wrapper before the server** — and look for
existing artifacts that prove the capability exists.

Related: [[mcp-stateless-migration-pattern]]
