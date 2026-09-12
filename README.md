# Postman for Agents

Postman's skills for coding agents.

The skill files in this repository are the single source of truth for the
Claude Code plugin:

| Route | How it gets the files | Lands at |
| --- | --- | --- |
| Claude Code plugin | `/plugin marketplace add postmanlabs/skills` clones this repo | Claude's plugin dir |

The Postman CLI also has its own path for installing these skills, but it's
still being redesigned — don't treat it as settled or document it here until
it lands.

## Layout

```
.claude-plugin/marketplace.json   the marketplace Claude Code adds
.claude-plugin/plugin.json        the plugin manifest
skills/<name>/SKILL.md            one skill per directory (bootstrap, mocking, ci, monitoring, api-documentation, ...)
manifest.json                     generated index of the skill files
scripts/build-manifest.js         regenerates it
```

## Installing

```
/plugin marketplace add postmanlabs/skills
/plugin install postman@postman
```

## Changing a skill

1. Edit the file under `skills/<skill>/`.
2. Run `node scripts/build-manifest.js`.
3. Commit both. CI runs `--check` and fails if you forget step 2.

Step 2 is not optional — `manifest.json` carries a `sha256` per file, and a
stale manifest silently drifts from what the files actually contain instead
of failing loudly.

## Adding a skill

Create `skills/<name>/SKILL.md` with `name` and `description`
frontmatter, where `name` matches the directory. Run the manifest script.

## The bindings placeholder

`SKILL.md` may contain `{{POSTMAN_BINDINGS}}`. `postman init` replaces it with a
table of that repository's spec path, collections directory, CLI version, and
workspace id. Anything that consumes a skill without substituting it should leave
the marker alone rather than guess.
