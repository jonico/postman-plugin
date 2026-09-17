# Postman CLI Installation

Two kinds of install, and they are refreshed by different commands. Get the
distinction right before running anything here:

- **The plugin's own copy** (the ladder's rungs 1 and 3) — plugin-local, not on
  `PATH`, ours to maintain.
- **A global install** (rung 2) — on `PATH`, owned by the user, installed by one
  of three different tools.

## Install

**Plugin-local, for rung 3 — the default. Not a global install:**

```bash
PM_DATA="${XDG_DATA_HOME:-$HOME/.local/share}/postman-plugin" \
  && mkdir -p "$PM_DATA" \
  && npm install --prefix "$PM_DATA" --no-save postman-cli
```

`--prefix` is an npm-wide config flag, so it is absent from
`npm install --help`; that absence is not evidence the command is wrong. The
binary lands at `$PM_DATA/node_modules/.bin/postman` and is invoked by absolute
path, never by name.

The rest of this section is the **global** install (rung 2), for a user who
wants `postman` on `PATH`.

**npm (all platforms):**

```bash
npm install -g postman-cli
```

**macOS, Linux, and WSL (curl):**

```bash
curl -o- "https://dl-cli.pstmn.io/install/unix.sh" | sh
```

**Windows (PowerShell):**

```powershell
powershell.exe -NoProfile -InputFormat None -ExecutionPolicy AllSigned -Command "[System.Net.ServicePointManager]::SecurityProtocol = 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://dl-cli.pstmn.io/install/win64.ps1'))"
```

## Check for drift

```bash
"$PM" --version                # installed, via the resolved invocation
npm view postman-cli version   # latest published
```

Use the resolved invocation, not a bare `postman` — on every rung but 2 the
binary is deliberately off `PATH`, so `postman --version` reports
"command not found" rather than a version.

## Update

**The plugin's own copy:** re-run the plugin-local install command above. It
overwrites in place; there is nothing to uninstall first.

**A global install:** run the same command that installed it — the npm, curl or
PowerShell line above, whichever put the binary there. Using a different one
leaves two `postman` binaries and a `PATH` question. Never `npm install -g` over
a copy that came from the curl installer or a system package manager.

The CLI has no self-update verb. `postman skills update` is a different
thing: it refreshes a repository's committed `postman/skills/`, not the
binary.

## Uninstall

npm installations:

```bash
npm uninstall -g postman-cli
```

Other install methods: delete the `postman` binary from its install
directory (`%USERPROFILE%\AppData\Local\Microsoft\WindowsApps` on Windows,
`/usr/local/bin` on macOS/Linux/WSL).

Source: https://learning.postman.com/docs/postman-cli/postman-cli-installation/
