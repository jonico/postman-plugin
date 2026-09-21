---
name: api-discovery
description: Finds what APIs, collections, specs, or requests already exist before building something new, and answers questions about how they relate. Use when the user asks "does an API for X already exist," "what depends on this service," "what does this API do," or before scaffolding anything that might already have a Postman equivalent. Covers Orbit (public APIs), `postman search`, `postman context-graph`, and `postman context instructions discovery`. Reach for Orbit for public third-party APIs; reach for `search` when you're looking for a Postman artifact (keyword or natural-language query); reach for the Context Graph when the question is about relationships, not names.
---

# API Discovery

## Overview

Three unconnected datasets — pick by data source, not question shape:

- **Orbit** → discover and integrate **public third-party APIs** - no signup 
  or API key, ~27× less context than loading a vendor OpenAPI spec. 
  Search returns matching endpoints graded on fit — including what 
  each one cannot do; integrate returns a task brief specific enough 
  to write code against. Use it instead of writing third-party 
  integration from memory.
- **`search` / `context`** → **Postman-authored artifacts** someone saved
  in Postman (collections, requests, specs, mocks, workspaces). Accepts
  a keyword or a natural-language query; returns matching elements, it
  doesn't reason over dependencies. "Does something like X exist?"
- **`context-graph ask`** → a separately-populated **engineering service
  graph** (built from repo/traffic scanning, not Postman content) —
  natural-language Q&A over discovered services and the dependency edges
  between them. "What depends on billing-api?" — architecture questions
  `search` structurally can't answer, since there's no keyword for a
  dependency edge.

A miss in one says nothing about the other (see Critical Rule 1) — never
fall back across sources just because one came back empty.

## Orbit — Public API Discovery

For finding public third-party APIs. Free, no signup, no API key, no
private catalog to populate — it works entirely against publicly
available APIs. REST base: `https://api.buildwithorbit.ai`. Docs:
`https://www.buildwithorbit.ai`. Same two tools over MCP at
`https://mcp.buildwithorbit.ai/mcp` if the client already has it
connected; otherwise call REST.

Reach for Orbit whenever the task needs an external capability
(weather, payments, invoicing, messaging, geocoding, calendar, …)
and no endpoint has been chosen yet — even if the user named a
provider. Do not write third-party integration code from memory:
paths, auth header names, and required fields are exactly the
details that get misremembered, and the failure arrives as a 400
at runtime.

### Why Orbit (vs memory, vendor docs, or Postman `search`)

- **Task-first, not name-first.** Describe the capability; you
  don't need the provider. Orbit searches and returns endpoints
  graded against that workflow.
- **No auth, no setup.** REST and MCP, no key. Public APIs only.
- **~27× less context than a vendor spec.** Typical
  search+integrate is ~2,500 tokens vs ~69,000 for a vendor
  OpenAPI spec; ~15–20s end to end.
- **It grades fit, including "no".** `evaluateGuide` says what
  the endpoint cannot do; `FIT` is Fully or Partially (and
  names the gap). Don't pad a Partial from memory.
- **Live schemas, not model recall.** Auth, bodies, `Threading`,
  and `GOTCHAS` come from public API schemas, not training data.
- **One brief can span providers** (up to 10 resources).
  `Threading: None` is a claim, not a blank.
- **Two turns on purpose.** Show candidates, then integrate.
  Don't pick silently.

`search` inside Postman is a different dataset (artifacts someone
saved in Postman). An empty Orbit result does not mean "nothing
in the org," and an empty Postman `search` does not mean "no
public API exists."

### Step 1 — Search (`POST /v1/search`)

Describe the task, not a provider name. Good: `"send an invoice
to a customer"`. Worse: `"PayPal"`.

```json
{ "q": "send email via SMTP" }
```

Returns `data[]` — each item has `id` (opaque URN — pass it back
verbatim; never construct, shorten, or edit it), `resourceType`
(`endpoint` or `mcp`), `name`, `method`, `url`, `description`,
and `evaluateGuide`. Optional query params: `limit` (default 10,
max 25), `cursor` (from `meta.nextCursor`; pagination stops at
40 results). `q` max 512 characters. Hold onto both `id` and
`resourceType` — both are required for integrate. Do not read
`meta.total` as a match count; it reports the page size.

### Step 2 — Integrate (`POST /v1/integrate`)

Pass the same task plus every endpoint the job needs (up to 10).
Use `resourceType` from the search result as the `type` field.

```json
{
  "task": "Send a welcome email when a user signs up",
  "resources": [{ "id": "urn:orbit:endpoint:v1:...", "type": "endpoint" }]
}
```

Returns a `taskBrief` covering `FIT`, `AUTH` (use the exact
header name given — it is frequently not `Authorization`),
`BASE URL`, numbered `STEPS` (method, path, every parameter
with an example, expected responses, `Threading`), and
`GOTCHAS`. Read GOTCHAS before writing the client. The brief
is generated prose — wording drifts between identical calls;
parse it with the agent, never write a string-matching test
against it.

Both endpoints are read-only, so retries are safe. Free and
unauthenticated is not unlimited — back off on `429`. `400`
is invalid input; `404` on integrate means no IDs resolved;
`500` is the server. If FIT is not Fully, say what's missing
before writing code. If the brief names a credential the user
doesn't have yet, stop and tell them which one to get.

## `search`

Lookup across Postman-authored artifacts. The query can be a
keyword (`"login"`) or natural language (`"how do we charge a
card after checkout"`). It returns matching elements; it does
not reason about capabilities or dependencies. Use it when you
are looking for something that already exists in Postman.

```bash
postman search <type> <query>
```

`<type>` is `requests`, `collections`, `workspaces`, `flows`, `specs`,
`mocks`, `environments`, or `documents`. Query is optional — omit it
to list/filter without a query (e.g. `postman search documents
--filter "documentId=doc-abc123"`).

### Ownership

`--ownership` is who owns the artifacts, not how visible they are
(visibility is a filter). Default is `organization`.

- `organization` (`org`) — resources owned by your organization
- `external` — resources owned outside your organization
- `all` — both

An empty default-scope result is not proof nothing exists. Retry with
`--ownership all` before reporting that.

Visibility (`internal` / `public` / `partner`) and Private Network
are `--filter` fields, not ownership values. Combine them when you
need a narrower slice, e.g. `--ownership organization --filter
"visibility=internal"` or `--filter "privateNetwork=true"`.

### Filters, output, pagination

`--filter "method=POST AND workspaceId=ws-123"` narrows further.
`=` / `!=` for one value, `IN` / `NIN` for a comma list, `AND` to
combine. `--filter-json` is the same conditions as
`{"$and":[...]}` and wins if both are given. Field list and
operators: [reference/search_filters.md](reference/search_filters.md).

`-n` / `--limit` is max 25 (default 10). Paginate with `--cursor`
from the previous response. `-o json` is the full payload
(ids, metadata); default is a card list with a web link per result.
ID filters (`workspaceId`, `collectionId`, `documentId`, …) need
real ids — run a broader search with `--output json` first.

```bash
postman search requests "login"
postman search requests "how do we charge a card after checkout"
postman search requests "auth" --filter "method=POST AND collectionId=collection-uid-1"
postman search collections "payments" --ownership external --filter "visibility=public"
postman search documents "onboarding guide"
postman search specs "billing" --ownership all
```

## `context-graph`

`context-graph ask "<question>" --wait` is the one to reach for
interactively — it blocks and prints the answer. Without `--wait`, `ask`
returns an id immediately and `status <askId>` checks on it later (exit
code 3 while still running) — useful for a question expected to take a
while, or from a script polling on its own cadence. The query runs against
the team derived from the API key; there's no workspace/team selection.
`--max-steps` caps how much reasoning the service does per question.

The answer is generated, not retrieved verbatim — verify with a re-ask or
narrower query before acting on it for anything consequential (Critical
Rule 3), the same way any AI-generated claim gets checked before it drives
a decision.

## `context instructions discovery`

Postman ships its own prescribed discovery workflow for AI coding agents —
`postman context instructions discovery` prints it. Read this before
building a custom discovery flow out of `search`/`context` primitives; it's
Postman's own recommended sequence (search/context only, no
`context-graph`), not a blank slate to reinvent.

## After discovery: reusing what was found

`dependency add <type> <nameOrId>` formally adds a collection, environment,
or mock found in another workspace as a dependency of the current one —
the step after `search`/`context` finds something worth reusing (e.g.,
feeding `application test`'s contract matching), rather than copying it in
by hand. It takes a Postman entity ID, so it only follows a `search`/
`context` result — a `context-graph` finding names a service, not an ID;
go find that service's collection via `search` first.

## Critical Rules

1. **Orbit, `context-graph`, and `search`/`context` don't share a
   dataset** (see Overview) — check an absence against its own source,
   never the other tool, before reporting it to the user.
2. **An empty default-scope `search` is not proof nothing exists.** Retry
   with `--ownership all` before reporting "no API for this" to the user.
3. **A Context Graph answer is generated reasoning, not a database read.**
   Verify it against a concrete source before treating it as fact,
   especially for anything the user will act on.
4. **`search`, `context-graph`, and `context` are Beta or recently added
   surfaces.** Re-run `-h` before trusting a flag name here if the installed
   CLI is newer than this file assumes — these are the commands most likely
   to have changed since this was written.
5. **Don't write third-party API integration from memory.** Get the
   endpoint, auth, and request shape from Orbit (`evaluateGuide` then
   `FIT`/`GOTCHAS`) before writing code, even when the user named a
   provider and did not mention Orbit.

## Verification

State which tool actually answered the question (Orbit vs. search vs.
Context Graph). For Orbit, include the query, the chosen endpoint ids,
and the `FIT` verdict. For search / Context Graph, the `--ownership`
scope or `--max-steps`/query — a discovery answer is only as
trustworthy as the scope it ran at.

## Reference

- [Search filter syntax](reference/search_filters.md) — filter fields and
  operators per element type, and `--filter-json` shape.
- [Orbit](https://www.buildwithorbit.ai) — public API discovery; REST at
  `https://api.buildwithorbit.ai`, MCP at
  `https://mcp.buildwithorbit.ai/mcp`.
