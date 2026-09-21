---
name: api-mocking
description: Stands up a fake backend that behaves like a real API — from a collection or an OpenAPI spec, running locally or pushed to Postman's cloud for a durable URL — plus request-time scenario and status-code overrides for testing failure paths. Use when the user asks to "mock this API," "create a mock server," "fake the backend," "run tests without hitting the real API," or "simulate an error/out-of-stock response." Covers `postman mock`. Depends on bootstrap for the workspace id only once a mock is pushed to the cloud (`-w`, or the workspace linked in `.postman/resources.yaml`) — generating and running a mock locally needs nothing from bootstrap.
---

# API Mocking

## Overview

A mock is two files on disk: `config.yaml` (name, port, scenarios) and
`default.js` — a plain Node HTTP server, and the mock itself, not a wrapper
around one. Generating, inspecting, running, and calling a mock are all local
and work for a logged-out guest. Only sharing it — pushing it to the cloud and
deploying a durable URL — needs `postman login`.

Every mock starts as a local folder; `mock push` promotes it to the cloud
later. Reach for cloud only when something other than you needs to hit this
mock over the network — a teammate, a CI job elsewhere, a webhook sender. A
purely local mock answering a `postman request` on your machine never needs it.

## Process

1. **Generate.** `postman mock generate -n NAME` with no source scaffolds a
   sample shopping-cart mock (`POST /cart/items`, `GET /cart`, `POST /checkout`)
   — the fastest way to a server that already answers, useful whenever the
   point is exercising mock *behavior* rather than a specific API's shape. Pass
   a real source — `postman mock generate SOURCE -n NAME`, where `SOURCE` is a
   collection file/directory or an `openapi.yaml` — when the endpoints need to
   mirror an actual API. Either form writes `config.yaml` + `default.js` into
   `postman/mocks/NAME/` (default port 4500).
   - `SOURCE --update ./postman/mocks/NAME` regenerates the default handler
     from the source in place, keeping the existing name, port, and scenarios.
     `--update` still needs the `SOURCE`; it cannot be combined with `--output`.
   - `-w <workspaceId>` saves the mock to a cloud workspace *instead of* the
     repository — it writes no local files and requires being logged in. Cannot
     be combined with `--output`, `--force`, or `--update`.
2. **Run it.** `postman mock run ./postman/mocks/NAME` starts the server and
   prints the bound URL (`... at http://localhost:PORT`). If the port in
   `config.yaml` is taken and `--port` wasn't passed explicitly, it falls back
   to a free OS-assigned port instead of erroring — read the real port off that
   line rather than assuming the configured one. Naming `--port N` explicitly
   makes a taken port a hard error; `--port auto` always picks a free one.
3. **Call it.** Plain `postman request localhost:PORT/route` returns the
   default scenario's response. Two headers change that per-request, with no
   restart needed: `x-mock-scenario: <name>` selects a different scenario (the
   valid names live in `config.yaml`), and `x-mock-response-code: <code>`
   returns that status instead. A wrong route and a wrong scenario name both
   come back as `Endpoint not defined` — indistinguishable from the message
   alone. There's no hot reload: a `default.js` edit does nothing until you
   Ctrl+C the running server and `mock run` it again.
4. **Push it, if it needs to leave your machine.**
   `postman mock push ./postman/mocks/NAME` is safe to re-run — `Created` the
   first time, `Updated` after — and records the cloud mapping in
   `.postman/resources.yaml`; commit that change. If a mock server is already
   live for this mock, the push updates what it serves.
5. **Deploy it, for a URL that outlives your terminal.**
   `postman mock deploy CLOUD_ID -s SLUG -y` prints
   `https://SLUG.mock.<team-domain>.postman.dev`. Deployed private by default —
   callers need a Postman API key (`x-api-key`) — add `--public` only when the
   mock should be reachable by anyone with the URL. `--auto-deploy` re-publishes
   the live server automatically whenever the mock changes; even without it, a
   later `push` already updates a live server, so you only re-`deploy` for the
   first URL or after taking the server down. (`deploy` and `log` are available
   on Postman Solo, Team, and Enterprise plans.)
6. **See who's calling it.** `postman mock get CLOUD_ID --json` returns
   `.mockServerId`; feed that into `postman mock log MOCK_SERVER_ID` for call
   entries (filter with `--method` / `--status` / `--path` / `--since` /
   `--until` / `--limit`, or `--json`). An empty log means the URL genuinely
   hasn't been hit — a rejected caller still shows up, recorded with its failing
   status code.
7. **Tear down.** `postman mock delete ./path --yes` (local) or
   `postman mock delete CLOUD_ID --yes` (cloud) — both refuse while the mock is
   running locally or deployed, so stop the local run or take the server down
   first. Cloud delete doesn't touch `.postman/resources.yaml`; drop that line
   by hand afterward or the repo keeps claiming a mock that's gone.

To point real request/assertion runs at a mock instead of hand-editing
base-URL variables, see the `api-testing` skill's `--use-mock`/`--mock` flags
on `collection run`.

## The two ids that matter

- **Code-mock id** — the `id` in `config.yaml`, and the id that `push` and
  `generate -w` print (often the same value). Use it for `get`, `deploy`,
  `delete`, and `run` by cloud id.
- **`mockServerId`** — a different value, available only from
  `mock get CLOUD_ID --json`. Use it for `mock log`, and nothing else.

## Critical Rules

1. **Every gated cloud command fails the same way:**
   `Authentication required. Run postman login or provide --api-key`, exit 1,
   nothing half-done. Whether a command is gated is decided by what you pass it,
   not the verb — `mock get`/`mock run` take either a local path (ungated) or a
   cloud ID (gated); `mock list` is gated only when called with no path.
2. **`push` is what moves an existing local mock to the cloud — `-w` at
   `generate` time is optional, not a fork you must choose up front.** A mock
   built as a guest can be pushed and deployed later with no rework.
3. **`--public` on `deploy` is the one action here with real exposure** — it
   stands up a server anyone with the URL can hit, with no API key. The default
   (private) is the safe one; confirm intent before adding it.
4. **To change a mock:** refresh it from a source with
   `mock generate SOURCE --update PATH`, or edit `default.js` by hand and
   restart `mock run`. Never pass a code-mock id to `mock log` — that command
   takes a `mockServerId`.
5. **`-w`/`--workspace` only exists on `generate`, `list`, `push`, and
   `deploy`.** `get`, `run`, `log`, and `delete` already take a path or an id
   that says where the mock is — there's nothing left for `-w` to resolve on
   those.

## Anti-patterns

- Running `generate -w` and expecting a local `postman/mocks/NAME/` folder —
  `-w` writes to the cloud instead of disk, so `mock run ./postman/mocks/NAME`
  then fails. Generate locally first, then `mock push`, if you want both.
- Passing the code-mock id to `mock log` instead of the `mockServerId`.
- Regenerating into a second folder because you assumed there's no in-place
  update — use `generate --update`.
- Adding `deploy --public` when the user didn't ask for a public URL.

## Verification

A mock isn't done because `generate` or `run` exited 0 — hit it with
`postman request` and check the actual status/body, or `mock get CLOUD_ID
--json` for a cloud one, then state whether it ended up local or cloud, and
(if deployed) private or public. For a scenario/status-code check, confirm the
header actually changed the response — a typo'd scenario name returns the same
`Endpoint not defined` as a wrong route, so a passing exit code alone proves
nothing.
