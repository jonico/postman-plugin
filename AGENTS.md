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
