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

### The data directory: compute it, never inherit it

Rungs 1-3 all read and write one plugin-owned directory:

```
${XDG_DATA_HOME:-$HOME/.local/share}/postman-plugin
```

**Write that expression out in full in every command that needs it.** Each Bash
call is a fresh shell: *"Environment variables don't persist. An `export` in one
command won't be available in the next."* A `PM_DATA=…` assignment in one call
is gone by the next, so assign it only to use it **within that same call**, as
rungs 2 and 3 do.

**Never substitute a host-provided plugin variable here — neither the bare
`$CLAUDE_PLUGIN_DATA` nor its braced form.** Two measured reasons and one
structural one:

- **It expands to nothing in a shell.** Claude Code substitutes that variable
  into skill *text*, and only when written braced; it does not export it to the
  Bash tool's environment. The bare `$CLAUDE_PLUGIN_DATA` above is therefore
  neither substituted nor set — which is why this file writes it bare, and why
  you are reading a name rather than a path. The resulting failure is loud but
  misdirecting: `mkdir -p ""` exits 1 with `cannot create directory ''`, which
  reads as a filesystem problem rather than an unexpanded variable, and the
  `&&` chain short-circuits, so the session never gets a data directory and
  falls through to rung 4.
- **Cursor and Kimi Code load these same files and substitute nothing.** All
  three manifests point at this one `skills/` directory rather than copying it,
  so a host-specific token ships verbatim to two hosts that will never resolve
  it — the Agent Plugins spec requires a client to leave unrecognized
  placeholders literal.
- **One path means one install**, shared by all three hosts and outliving plugin
  updates. Anything under the plugin's *install* directory does not survive:
  uninstall+install deletes and rebuilds it.

POSIX shell: `$HOME` resolves under Git Bash and WSL. A native PowerShell
session has no rung-3 story and falls to a global install (rung 2) via
[reference/cli_installation.md](reference/cli_installation.md).

### The ladder

Work down it and stop at the first rung that answers. Record the absolute path
you settled on so later sessions skip straight to rung 1.

1. The recorded path, if the record is non-empty and the binary it names still
   answers. Read and verify in one call; a missing record is a quiet miss, not
   an error worth reporting.

   ```bash
   PM_BIN=$(cat "${XDG_DATA_HOME:-$HOME/.local/share}/postman-plugin/postman-bin" 2>/dev/null) \
     && [ -n "$PM_BIN" ] && "$PM_BIN" --version
   ```

   If the record exists but the binary no longer answers, the record is stale —
   delete it and continue to rung 3, **not** rung 2. A global install found by
   rung 2 can sit on a version-manager path (`.nvm/versions/node/<v>/bin`) that
   dies on the next `nvm use`, so re-recording one is how a dead record comes
   back. Rung 3's copy does not move.
2. `postman --version`. If a global install answers, record its path and use it.
   Resolve first, write only on success — a bare redirect truncates the record
   before `command -v` has answered, leaving an empty file that rung 1 would
   later have to reject:

   ```bash
   PM_DATA="${XDG_DATA_HOME:-$HOME/.local/share}/postman-plugin"
   PM_BIN=$(command -v postman) && mkdir -p "$PM_DATA" \
     && printf '%s\n' "$PM_BIN" > "$PM_DATA/postman-bin"
   ```
3. Install once into the plugin's own data directory, then record it. One call,
   so the variable survives to the lines that use it, and `&&`-chained
   throughout — an unchained `printf` would record a path that `npm install`
   never created, which rung 1 then reports as a success next session and rung 4
   never gets the chance to catch:

   ```bash
   PM_DATA="${XDG_DATA_HOME:-$HOME/.local/share}/postman-plugin" \
     && mkdir -p "$PM_DATA" \
     && npm install --prefix "$PM_DATA" --no-save postman-cli \
     && printf '%s\n' "$PM_DATA/node_modules/.bin/postman" > "$PM_DATA/postman-bin"
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

### Invoking what the ladder resolved

**Rungs 1, 3 and 4 all leave the binary off `PATH` by design, so a bare
`postman …` returns "command not found" on every rung except 2.** That is the
same signal Critical Rule 2 warns misroutes a session into rung 4 — and it is
reached by following a `postman …` command rather than by any decision. Bind
the resolved invocation once per call and use it:

```bash
PM="$(cat "${XDG_DATA_HOME:-$HOME/.local/share}/postman-plugin/postman-bin")" \
  && "$PM" --version
```

**Read every `postman …` in this file and in the sibling skills as `"$PM" …`** —
including the `-h` commands above and the drift check below. On rung 4 alone
`$PM` is not a path: substitute `npx --yes --package=postman-cli postman` for it
and expect a registry fetch per call.

### Is the resolved copy current?

Rungs 1 and 2 establish that *something runs*, not that something *current*
runs — `--version` is a liveness probe there, so a binary installed months ago
wins the ladder indefinitely and rung 3 never fires again. Compare the two once
per session, before real work — see
[reference/cli_installation.md](reference/cli_installation.md):

```bash
"$PM" --version                # installed, via the resolved invocation
npm view postman-cli version   # latest published
```

Skip this immediately after rung 3 fired: you just installed `latest`, so there
is nothing to compare.

What to do about a mismatch depends on who owns that copy:

- **Rungs 1 and 3 — the plugin's own copy**, in the data directory. Ours to
  maintain: re-run **rung 3's block in full** to refresh it, then say you did
  and which version replaced which. Reconstructing the `npm install` line alone
  gives `--prefix ""`, which installs into the current directory and drops a
  `node_modules/` into the user's repo.
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
   fallback.** Rung 3 installs the real CLI under
   `${XDG_DATA_HOME:-$HOME/.local/share}/postman-plugin`, off `PATH` by design.
   Routing to `postman-mcp-fallback` because the binary is not on
   `PATH` defeats the entire point of this plugin. Only no shell, no Node, or a
   hosted session that cannot install qualifies.
2. **Never record an `npx` invocation as the resolved answer.** `npx`
   re-resolves the package from the registry on every call, so persisting it
   makes every later session pay a network fetch. Only an absolute path gets
   recorded.

   The observed way this goes wrong is not a deliberate choice: an unresolved
   data directory makes rungs 1-3 all fail on a path that expanded to nothing,
   and the session falls through to rung 4 and stays there for every
   subsequent command. If you are about to reach for `npx`, echo the data
   directory path first and confirm it is non-empty — that is the actual fault
   far more often than a missing Node.
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
8. **Write no host-specific path into a command.** One `skills/` directory is
   loaded by Claude Code, Cursor and Kimi Code, so a command that only resolves
   on one host is broken on the other two — and, as rung 1's note records, a
   host variable that is substituted into text but absent from the shell
   environment is broken on that host too. Derive paths in the shell from what
   a fresh shell always has — `$HOME`, `$XDG_DATA_HOME` — and write the whole
   expression in each command rather than carrying it in a variable between
   calls. If a host genuinely needs its own handling, branch on something
   observable at runtime, never on a variable the host is assumed to have set.
   This applies to every skill in the plugin; bootstrap is just where the paths
   are.

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
