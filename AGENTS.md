# Agent instructions for this repo

## Keep README.md in sync with the skills that actually exist

`README.md`'s `## Layout` section and its "Adding a skill" / "Removing a
skill" sections describe `skills/` in prose. That prose does not update
itself when a skill is added, renamed, or deleted — it has already drifted
before (a `## Layout` line hardcoded five skill names and kept two of them
long after those skills were replaced).

Whenever a change to this repo adds, renames, or removes a directory under
`skills/`:

1. Re-read `README.md` and update anything that names a specific skill —
   don't leave an enumerated list of skill names in prose if the change
   makes it stale. Prefer phrasing that points at `skills/` itself over
   re-typing the list, so the next rename doesn't create the same problem.
2. Grep the whole repo for the old name before deleting or renaming a
   skill directory — `grep -rn "<old-name>" README.md skills/ intent.md` —
   since other `SKILL.md` files reference each other by name in prose
   (descriptions, Critical Rules, "see `<skill>`" pointers), not just
   through frontmatter or `manifest.json`. Fix every hit; a reference to a
   deleted skill fails silently, it doesn't error.
3. Run `node scripts/build-manifest.js` and commit the regenerated
   `manifest.json` alongside the skill change.

`intent.md` is a historical design record of how the current skill set was
planned, not living documentation — its old skill names are not bugs and
should not be "fixed" to match the current directory list.

## The per-route MCP configs are hand-maintained, and asymmetric on purpose

Four files carry MCP configuration, one per route:

```
mcp.claude-code.json        <- .claude-plugin/plugin.json  "mcpServers": "./mcp.claude-code.json"
mcp.cursor.json             <- .cursor-plugin/plugin.json  "mcpServers": "./mcp.cursor.json"
.kimi-plugin/plugin.json       inline — Kimi documents no path form
```

They look like near-duplicates and they are not. Three differences are load
bearing, and each has cost time before:

1. **`X-Source` must differ per route** (`postman-claude-code-plugin`,
   `postman-cursor-plugin`, `postman-kimi-plugin`). That is the whole reason
   these files are separate: it is the dimension telemetry keys on. Two routes
   sharing a value collapse into one bucket, which reads exactly like an agent
   nobody uses.
2. **Versions are independent per route.** Each ships on its own cadence, so
   differing versions are correct, not drift. Within a route, though, the
   manifest `version` and the two version strings in its headers must agree.
   `validate.yml` enforces the second and deliberately not the first.
3. **Kimi's URL uses `${POSTMAN_MCP_MODE:-minimal}` while the others use `mcp`.**
   That segment selects a different tool surface. Unifying it changes which tools
   Kimi users get — it is a product decision, not a tidy-up.

Copying one file over another to "make them consistent" breaks all three. There
is no generator: an earlier version of this repo had one, and it was removed once
versions became per-route, because a tool whose job is to keep them identical is
wrong under that model.

Don't "consolidate" these into a default-named file either. Claude Code's default
is `.mcp.json` at the plugin root and Cursor auto-discovers `mcp.json` — two
names one letter apart in the same directory. Cursor's docs never mention the
dot-prefixed name but never rule it out, and the live
`Postman-Devrel/cursor-postman-plugin` ships `.mcp.json` and works, so a repo
carrying both defaults could hand a client two configs with different `X-Source`
values and no documented winner. This repo ships neither default name and points
each manifest at its own file. Verified in `pmbox`: with neither default present,
`claude mcp list` still reported `plugin:postman:postman`, so the pointer alone
is enough.

Note the Agent Plugins spec permits neither the pointer nor the inline form —
§7: MCP config "MUST NOT be declared inline in `plugin.json` or loaded from any
alternative core path", only `mcp.json` at the plugin root. Per-agent headers are
therefore only available through client-native manifests, so don't treat spec
conformance and per-agent telemetry as compatible goals without deciding which
one wins.
