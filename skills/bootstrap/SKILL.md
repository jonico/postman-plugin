---
name: bootstrap
description: Resolves the Postman CLI, authenticates, links the workspace, and records this repo's spec path, collections directory and workspace id. Use when the user asks to "set up Postman here", "connect this repo to Postman", "link this workspace", or "run postman init" — and before the mocking, ci, monitoring or api-documentation skills only when the CLI, the linked workspace or the spec path has not already been confirmed in this session. Those skills stop and point back here if it has not completed; they never re-derive these values themselves.
---

# Bootstrap Postman for This Repo

## Overview

A one-time, idempotent setup every other Postman skill in this plugin depends
on. Authenticate with `postman login`, then run this repo's `postman init`
(see the root README) to fetch `manifest.json` and write the skill bindings —
the values behind `{{POSTMAN_BINDINGS}}`: spec path, collections directory,
workspace id. Nothing downstream re-derives or guesses these — they read
what this skill recorded.

## Ask the CLI what it can do: `-h`

The CLI is self-describing at three levels, and its own top-level output says
so — *"To get available options for a command: `postman <command> -h`"*. Walk
down only as far as the question needs:

```
postman -h                      # the resource list: collection, spec, mock, monitor, workspace, api, flows…
postman <resource> -h           # that resource's actions — e.g. postman spec -h → lint, ai-readiness
postman <resource> <action> -h  # the real flags, defaults and examples — e.g. postman spec lint -h
```

Read the third level before writing any command that carries a flag. It is
the only place the **defaults** are stated, and a wrong default fails
silently rather than loudly. Many `<action> -h` screens end with worked `Eg.`
lines — copy the shape from there rather than composing one.

This replaces guessing, and it often replaces asking the user. Two cases
where it is the whole answer:

- There's no single verb for "confirm the workspace is linked and synced."
  Run `postman workspace -h` and pick from what it prints. On v1.56.3 that is
  `list` (which workspaces exist — start here when the id is unknown),
  `create`, `connect-git`, `pull`, `push`, `prepare`, `lint`, each a different
  direction. The live output is authoritative, not this list.
- A capability the user names may not map to the verb it sounds like. Check
  before reporting it missing, and check before inventing it.

## Resolving the invocation

Work down this ladder and stop at the first rung that answers. Record the
absolute path you settled on so later sessions skip straight to rung 1.

1. `$CLAUDE_PLUGIN_DATA/postman-bin`, if it exists and holds an absolute path —
   verify it with `--version` and use it. This is the normal case after the
   first session.
2. `postman --version`. If a global install answers, record its path with
   `command -v postman > "$CLAUDE_PLUGIN_DATA/postman-bin"` and use it.
3. Install once into the plugin's own data directory, then record it:

   ```
   npm install --prefix "$CLAUDE_PLUGIN_DATA" --no-save postman-cli
   printf '%s\n' "$CLAUDE_PLUGIN_DATA/node_modules/.bin/postman" \
     > "$CLAUDE_PLUGIN_DATA/postman-bin"
   ```

   This happens once per machine, not once per session. Tell the user it is
   happening and that it will not recur.
4. Only if rung 3 fails — no Node, no network, no write access — fall back to
   `npx --yes --package=postman-cli postman` for this session alone, and say
   that every call will re-resolve the package from the registry.
5. If rung 4 also fails, the CLI is genuinely unavailable. Say which rung
   failed and why before considering `postman-mcp-fallback`.

See [reference/cli_installation.md](reference/cli_installation.md) for the
per-platform install/update/uninstall commands behind rungs 2-4 (npm,
curl, PowerShell).

### Is the resolved copy current?

Rungs 1 and 2 establish that *something runs*, not that something *current*
runs — `--version` is a liveness probe there, so a binary installed months ago
wins the ladder indefinitely and rung 3 never fires again. Compare
`postman --version` against `npm view postman-cli version` once per session,
before real work — see
[reference/cli_installation.md](reference/cli_installation.md).

What to do about a mismatch depends on who owns that copy:

- **Rungs 1 and 3 — the plugin's own copy**, under `$CLAUDE_PLUGIN_DATA`. Ours
  to maintain: re-run rung 3's `npm install --prefix …` to refresh it, then say
  you did and which version replaced which.
- **Rung 2 — a global install the user owns.** Report the drift, name both
  versions, and let them decide. **Never `npm install -g` over it**: it may
  have come from the curl installer or a system package manager, and upgrading
  it with the wrong tool leaves two `postman` binaries and a `PATH` question.
  Update it with *the same command that installed it* — see
  [reference/cli_installation.md](reference/cli_installation.md).

Drift is not cosmetic: newer surface is simply absent from an older install —
`postman spec ai-readiness`, for one. Keep this separate from the API Builder
deprecation in Critical Rule 7: that one turns on the Postman platform
generation (v11 vs v12), not the CLI version, so upgrading the CLI does not
change it.

## Critical Rules

1. **A missing `postman` binary is never a reason to switch to the MCP
   fallback.** Rung 3 installs the real CLI into the plugin's own data
   directory. Routing to `postman-mcp-fallback` because the binary is not on
   `PATH` defeats the entire point of this plugin. Only no shell, no Node, or a
   hosted session that cannot install qualifies.
2. **Never record an `npx` invocation as the resolved answer.** `npx`
   re-resolves the package from the registry on every call, so persisting it
   makes every later session pay a network fetch. Only an absolute path gets
   recorded.
3. **Never fabricate a workspace id, spec path, or collections directory.**
   If the CLI can't resolve one, report the gap and stop. A guessed value
   here corrupts every skill that trusts it downstream.
4. **Check existing state before setting anything up — and remember that
   "present" is not "current".** Detect what's already true — CLI installed?
   *at which version?* already logged in? workspace already linked? — and skip
   finished steps. Don't assume a blank slate, and don't assume nothing needs
   to happen just because the CLI is present. An installed-but-stale CLI is
   the case the ladder is blindest to, because it satisfies every rung it
   reaches; see [Is the resolved copy current?](#is-the-resolved-copy-current).
5. **Wire up an existing repo only. Never scaffold a new API.** If there's no
   spec or collection yet, that's a design decision for the user to make.
   Report the gap; do not generate a starter spec to fill it.
6. **Never invent a subcommand or a flag. Run `-h` first, and believe it.**
   The CLI's surface is not guessable from the feature name, and a wrong verb
   fails in a way that reads like the feature is missing.

   ```bash
   # WRONG — plausible, and does not exist
   postman api generate-collection openapi.yaml

   # CORRECT — ask, then act
   postman mock generate -h   # …then the command it actually documents
   ```

   If `-h` doesn't list what you need, don't substitute a verb that sounds
   right.

   **Before reporting a feature missing, check the version once.** `-h`
   describes the binary in hand. If the resolved copy is behind the latest
   published, say so and offer to refresh — then re-read `-h`. If it is
   current, `-h` is the answer: the CLI does not do it. Never assert the CLI
   is out of date without having compared the two version strings, and never
   make a refresh a precondition for answering the question that was asked.
7. **Specs belong to Spec Hub. The API Builder is deprecated — never route new
   work to `postman api`.** Postman's docs are explicit: the API Builder *"is
   no longer supported in Postman v12 and later"* and *"Spec Hub has replaced
   the API Builder"*; `postman api lint` is *"supported for API Builder
   objects in Postman v11, but not in v12 and later"*. So `postman spec lint`
   is the command for a specification, and `postman api …` applies only to a
   pre-existing v11 API Builder object. The `api` resource is still listed in
   `postman -h` and prints **no deprecation warning**, so seeing it there is
   not evidence it is current — this rule is. If a repo has API Builder
   artifacts, say they need migrating to Spec Hub rather than quietly
   building on them.

## Verification

Bootstrap is done only when the resolved invocation has answered a real
`--version`, and workspace id, spec path, and collections directory are all
non-empty and stated back to the user. "The CLI is installed" is not the bar —
those three resolved values are. Never report that Postman is "set up" because
a skill loaded; loading a skill configures nothing.

State the resolved version alongside those three values, plus either the
latest published version or the fact that the check could not run (no network
is a normal answer, and does not block bootstrap). Never call the CLI current
without having compared.

## Reference

- [Collection Schema v3](reference/collection_schema_v3.md) — the schema for
  the collection files this skill resolves the directory for.
- [CLI Installation](reference/cli_installation.md) — install/update/
  uninstall commands per platform.
