# Postman for Agents

Postman's skills for coding agents.

The skill files in this repository are the single source of truth for every
plugin route below — each tool's manifest points back at the same `skills/`
directory rather than copying files into itself:

| Route | How it gets the files | MCP config it reads | Reports itself as |
| --- | --- | --- | --- |
| Claude Code plugin | `/plugin marketplace add postmanlabs/postman-plugin` clones this repo | `mcp.claude-code.json` | `postman-claude-code-plugin` |
| Cursor plugin | `.cursor-plugin/plugin.json` points at this repo's `skills/` dir | `mcp.cursor.json` | `postman-cursor-plugin` |
| Kimi Code plugin | `.kimi-plugin/plugin.json` points at the same `skills/` dir | `mcpServers` in `.kimi-plugin/plugin.json` | `postman-kimi-plugin` |

The Postman CLI also has its own path for installing these skills, but it's
still being redesigned — don't treat it as settled or document it here until
it lands.

## Layout

```
.claude-plugin/marketplace.json   the marketplace Claude Code adds
.claude-plugin/plugin.json        the Claude Code plugin manifest
.cursor-plugin/plugin.json        the Cursor plugin manifest
.kimi-plugin/plugin.json          the Kimi Code plugin manifest — carries its MCP block inline
mcp.claude-code.json              Claude Code's MCP config
mcp.cursor.json                   Cursor's MCP config
skills/<name>/SKILL.md            one skill per directory — see skills/ for the current list
manifest.json                     generated index of the skill files
scripts/build-manifest.js         regenerates it
```

## Installing

Claude Code:

```
/plugin marketplace add postmanlabs/postman-plugin
/plugin install postman@postman
```

Cursor or Kimi Code:

```
npx plugins add postmanlabs/postman-plugin
```

## Data sent to Postman

Postman CLI commands these skills run report to Postman by default. There are
two separate paths, and only one of them can be turned off.

### Declinable: `--no-report-events`

Seven commands send analytics — and in `application test`'s case the run
results too — unless you opt out:

| Command | Sent by default |
| --- | --- |
| `postman collection run` | Run analytics (see the note below on run history) |
| `postman application test` | Run results and analytics |
| `postman spec lint` | Lint analytics (violation counts, pass/fail) |
| `postman workspace push` | Push analytics |
| `postman runner start` | Runner analytics |
| `postman flows run` | Flow run analytics |
| `postman request` | Request analytics |

**One spelling covers all seven.** `--no-report-events`,
`--report-events false` and `--report-events=false` are equivalent —
`bin/postman.js` rewrites the latter two into the first before the arguments
are parsed, for every command. The positive `--report-events` no longer
enables anything on these commands; its help text says it is accepted for
compatibility only.

**`collection run` uploads its run history either way.** The opt-out covers
analytics only — the upload is gated on a separate internal flag that
`--no-report-events` does not touch. What the flag does affect is git-native v3
collections specifically: it selects the execution engine that lets *their*
results upload, which is why the command's `--report-events` help text reads
"Upload results for git-native v3 collection runs. Analytics are sent by
default." Contrast `application test`, whose opt-out does cover both.

The exception is `postman init`, which the `bootstrap` skill runs. There
`--report-events` is opt-*in* (it gates one richer analytics row and needs a
login), and `init` declares no negated form — so `--report-events=false` on
`init` is rewritten to an option it does not have, and the command exits
non-zero with `unknown option`. Don't copy the opt-out onto `init`.

### Not declinable: client-events

Independently of any flag, the CLI emits a one-line "this command ran" event to
Postman's unauthenticated client-events collector. It does not depend on
`--report-events` and does not depend on being logged in, so
`--no-report-events` does not stop it. `postman collection run`,
`spec lint`, `workspace push` and `init` emit it in addition to the table
above, as do the `postman mock` subcommands and `postman performance run`.

The one thing that does suppress it: the collector is only wired for the US
region, and emission no-ops in other regions (EU included).

### MCP

Separately, **every route** configures the hosted Postman MCP server at
`mcp.postman.com`, so MCP tool calls made through any of them reach Postman
too — see [The MCP server config](#the-mcp-server-config). None of the CLI
flags above apply to that traffic; declining it means not installing the MCP
server.

One caveat on the mode segment: all three URLs are written as
`https://mcp.postman.com/${POSTMAN_MCP_MODE:-...}`, and Claude Code does not
expand a placeholder inside a URL — `claude plugin list --json` reports the
registered URL with the literal `${POSTMAN_MCP_MODE:-mcp}` still in the path.
The Agent Plugins spec agrees: only `${PLUGIN_ROOT}`/`${PLUGIN_DATA}` expand,
and never in a URL. So the per-route mode choice below is deliberate but does
not currently take effect on the Claude Code route; resolve it at build time
or behind a stdio wrapper. Unverified for Cursor and Kimi.

## The MCP server config

Each route has its own config file, so each can report itself in `X-Source` and
traffic can be attributed to the agent it came from:

```
mcp.claude-code.json        <- .claude-plugin/plugin.json  "mcpServers": "./mcp.claude-code.json"
mcp.cursor.json             <- .cursor-plugin/plugin.json  "mcpServers": "./mcp.cursor.json"
.kimi-plugin/plugin.json       inline — Kimi documents no path form
```

Maintained by hand, and they are not interchangeable copies — `X-Source`, the
version, and the URL's `mcp` vs `minimal` mode all differ per route on purpose.
`AGENTS.md` has the reasons and the constraints before you change one.

## Changing a skill

1. Edit the file under `skills/<skill>/`.
2. Run `node scripts/build-manifest.js`.
3. Bump the version on every route that ships the change. Routes version
   independently — differing versions across routes are correct, not drift —
   so a bump means the three strings that one route owns: `version` in its
   manifest, plus `X-Plugin-Version` and `User-Agent` in its MCP config (for
   Kimi all three live in the manifest). `validate.yml` fails if a route's
   three disagree, and deliberately does not compare routes to each other.
   Don't skip this: `claude plugin update` compares only that string against a
   version-keyed cache, so a release that changes files without bumping it
   reports "already at the latest version" and delivers nothing. Semver here is
   major for a breaking change to a skill's contract, minor for a new skill,
   patch for wording or a bug fix.
4. Commit all of it. CI runs `--check` and fails if you forget step 2.

`marketplace.json` deliberately declares no version — it would override
`plugin.json` and give that route a second source of truth.

Step 2 is not optional — `manifest.json` carries a `sha256` per file, and a
stale manifest silently drifts from what the files actually contain instead
of failing loudly.

## Adding a skill

Create `skills/<name>/SKILL.md` with `name` and `description`
frontmatter, where `name` matches the directory. Run the manifest script.

## Removing a skill

Delete `skills/<name>/`, then grep the rest of the repo for that name —
`grep -rn "<name>" README.md skills/ intent.md` — since other `SKILL.md`
files and this README can reference a skill by name in prose, not just in
frontmatter, and nothing catches a stale reference automatically. Fix or
remove what turns up, then run the manifest script.

## The bindings placeholder

`SKILL.md` may contain `{{POSTMAN_BINDINGS}}`. `postman init` replaces it with a
table of that repository's spec path, collections directory, CLI version, and
workspace id. Anything that consumes a skill without substituting it should leave
the marker alone rather than guess.

## License

Apache-2.0 — see [LICENSE](LICENSE).
