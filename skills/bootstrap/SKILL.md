---
name: bootstrap
description: This skill should be used before running the mocking, ci, monitoring, or api-documentation skills in this plugin, whenever a repository has no confirmed Postman CLI login, linked workspace, or resolved spec/collection path yet. Also use when the user asks to "set up Postman here", "connect this repo to Postman", "link this workspace", or "run postman init". It is a hard prerequisite — the other four skills stop and point back here if it hasn't completed.
---

# Bootstrap Postman for This Repo

## Overview

A one-time, idempotent setup every other Postman skill in this plugin depends
on. Authenticate with `postman login`, then run this repo's `postman init`
(see the root README) to fetch `manifest.json` and write the skill bindings —
the values behind `{{POSTMAN_BINDINGS}}`: spec path, collections directory,
workspace id. Nothing downstream re-derives or guesses these — they read
what this skill recorded.

There's no single CLI verb for "confirm the workspace is linked and synced."
Check `postman workspace -h` rather than assume one — `create`,
`connect-git`, `pull`, `push`, `prepare`, and `lint` are the real
subcommands, each a different direction (new workspace, bind an existing
one, cloud→local, local→cloud, validate before push, validate in place).

## Critical Rules

1. **Never fabricate a workspace id, spec path, or collections directory.**
   If the CLI can't resolve one, report the gap and stop. A guessed value
   here corrupts every skill that trusts it downstream.
2. **Check existing state before setting anything up.** Detect what's already
   true — CLI installed? already logged in? workspace already linked? — and
   skip finished steps. Don't assume a blank slate, and don't assume nothing
   needs to happen just because the CLI is present.
3. **Wire up an existing repo only. Never scaffold a new API.** If there's no
   spec or collection yet, that's a design decision for the user to make.
   Report the gap; do not generate a starter spec to fill it.

## Verification

Bootstrap is done only when workspace id, spec path, and collections
directory are all non-empty and stated back to the user. "The CLI is
installed" is not the bar — those three resolved values are.
