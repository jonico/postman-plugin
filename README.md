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

Some Postman CLI commands these skills run report events and results to Postman
by default. Each has its own opt-out flag — they are not spelled the same, so
copy the one for the command you're running:

| Command | Sent by default | Opt out with |
| --- | --- | --- |
| `postman application test` | Run results upload to Postman after each run | `--report-events=false` |
| `postman runner start` | Runner analytics | `--no-report-events` |
| `postman flows run` | Flow run analytics | `--no-report-events` |

Separately, every route configures the hosted Postman MCP server at
`mcp.postman.com`, so MCP tool calls made through any of them reach Postman too.

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
3. Bump the version — see [Releasing](#releasing).
4. Commit all of it. CI runs `--check` and fails if you forget step 2.

Step 2 is not optional — `manifest.json` carries a `sha256` per file, and a
stale manifest silently drifts from what the files actually contain instead
of failing loudly.

## Releasing

`.claude-plugin/plugin.json` declares a `version`, and that string is the only
thing `claude plugin update` compares. An install is cached at a version-keyed
path, so a release that changes files without changing the version reports
"already at the latest version" and delivers nothing. Bump it on every release
that users should receive — this repo has shipped empty updates for exactly
this reason before.

Each route versions independently — they ship on their own cadences, so
differing versions are expected and nothing compares them. Bumping one means the
three strings that route owns: `version` in its manifest, and `X-Plugin-Version`
and `User-Agent` in its MCP config (for Kimi, all three are in the manifest).
`validate.yml` fails if a route's three disagree; it does not compare routes.

`marketplace.json` deliberately declares no version — it would override
`plugin.json` and give that route a second source of truth.

Semantic versioning: a breaking change to a skill's contract is major, a new
skill is minor, and a wording or bug fix is patch.

TODO: what is enforced is only that a route is internally consistent. Nothing
fails a PR that changes `skills/` without bumping any version at all, which is
the case that ships an empty update. Worth adding a PR gate requiring a
semver-greater version on at least the routes whose shipped files changed.
`claude plugin validate .` would also catch manifest schema errors the current
JSON.parse loop cannot.

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
