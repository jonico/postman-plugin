---
name: api-code-generation
description: Finds a request in Postman, explores it via MCP context tools, and generates faithful client code for it into the local repo — then tracks, updates, and removes what it installed. Use when the user asks to "install this API," "generate a client for this endpoint," "find me a good email API and wire it in," "what Postman-sourced code do we have installed," "are our integrations up to date," or wants to remove a previously generated request. Covers the MCP `code` toolset's `get*Context` family and `getCodeGenerationInstructions` — a different job from `api-discovery` (finding whether something exists, via the CLI) and from generic MCP CRUD tools (editing Postman's own data, not the local repo).
---

# API Code Generation

## Overview

This skill turns a request that already exists in Postman into real client
code living in the repo — "install a request" means fetching its full
context from Postman and generating a file that faithfully represents that
endpoint. It runs entirely over Postman's MCP server (see
`postman-mcp-fallback` for connecting it and picking a toolset — `code` is
the minimum this skill needs, `full` also works). It has nothing to do with
the CLI-driven skills in this plugin, and nothing to do with editing data in
Postman itself.

`api-discovery`'s `search`/`context-graph` answer "does something like this
exist"; this skill answers "put the thing that already exists into my
code." A user question about existence routes to `api-discovery`; a request
to actually generate or maintain code from a found request routes here.

## Which tools to use

Prefer the `*Context` tools over their generic CRUD equivalents whenever
the goal is understanding or generating code — the context tools return
markdown shaped for that purpose, not raw entity JSON:

| Purpose | Use (context tool) | Not (generic CRUD tool) |
| --- | --- | --- |
| Collection structure | `getCollectionContext` | `getCollection` |
| Request details | `getRequestContext` | `getCollectionRequest` |
| Full code-gen context | `getRequestCodeContext` | *(no equivalent)* |
| Folder details | `getFolderContext` | `getCollectionFolder` |
| Response example | `getResponseContext` | `getCollectionResponse` |
| Workspace details | `getWorkspaceContext` | `getWorkspace` |
| List workspaces | `getWorkspacesContext` | `getWorkspaces` |
| Environment | `getEnvironmentContext` | `getEnvironment` |
| Workspace environments | `getWorkspaceEnvironmentsContext` | `getEnvironments` |

Confirm these are still exposed with `getEnabledTools` if a name here comes
back unavailable — toolset contents are the part of this surface most
likely to have changed since this was written (see `postman-mcp-fallback`).

## Workflow

1. **Find.** For a public/third-party API, `searchPostmanElements` with
   `ownership: external`. For an internal API, `getWorkspacesContext` (filter
   to personal workspaces if the user said "my APIs") then
   `getWorkspaceContext`. When the user states a need rather than a name
   ("I need an email API"), explore a few candidates and compare them on
   real structure — auth approach, folder layout, endpoints — not on
   general knowledge of the vendor.
2. **Explore.** Drill into the collection/folder/request with the context
   tools above. Fetch only what's relevant — don't dump an entire large
   collection to answer a narrow question.
3. **Confirm before installing.** Present the candidate folders/requests and
   ask which ones to install. Never install all of them on the strength of
   "explore this API" alone — exploring and installing are different asks.
4. **Install.** For each confirmed request, call `getRequestCodeContext` to
   get the full code-gen document (collection metadata, request detail,
   parent folder docs, response examples, environment variables) — this is
   the one document actually built for generating code, and no code
   generation should proceed without it. Then follow
   `reference/code_generation_rules.md` in full before writing the file.
5. **Maintain.** For listing, staleness checks, unused-request cleanup, or
   removal of already-installed requests, follow
   `reference/maintenance.md`.

## Critical Rules

1. **Never generate code without the request-level confirmation in step 3.**
   Exploring a collection is not consent to install everything in it.
2. **`getRequestCodeContext` is mandatory before generating code for a
   request — `getCollectionContext`/`getRequestContext` alone are for
   exploring, not for code generation.** They're missing pieces (parent
   folder docs, resolved environment variables) that code generation needs.
3. **Every generated file gets the provenance header from
   `reference/code_generation_rules.md`, no exceptions.** It's the only way
   a later "what's installed" or "check for updates" can find the file
   again and know which Postman request it came from.
4. **Match the target project's own conventions before reaching for a
   language default.** See `reference/code_generation_rules.md` — an axios
   project gets axios, not fetch, unless told otherwise.
5. **Don't add validation or business logic the API definition doesn't
   have.** The generated file is a faithful representation of the request;
   logic beyond that belongs in the caller.

## Verification

State which collection/request was installed, the file path it landed at,
and confirm the provenance header (collection UID, request UID, modified-at
timestamp) is present in the generated file. For a maintenance pass, state
how many installed requests were found and their status (current/outdated)
rather than "everything's fine."

## Reference

- `reference/code_generation_rules.md` — file placement, the provenance
  header format, variables file, auth, response handling, shared-code
  extraction. Read before generating any file.
- `reference/maintenance.md` — listing installed requests, checking for
  upstream changes, finding unused requests, removing them.
- `postman-mcp-fallback` skill — connecting the MCP server and choosing a
  toolset; this skill assumes that's already done.
- `api-discovery` skill — the CLI-side answer to "does this exist," a
  different dataset and a different surface than this skill's MCP tools.
