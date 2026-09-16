# Postman CLI Installation

## Install

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
postman --version              # installed
npm view postman-cli version   # latest published
```

## Update

To update an existing installation to the latest version, run the same
command used to install it — the npm, curl or PowerShell line above,
whichever put the binary there. Using a different one leaves two `postman`
binaries and a `PATH` question.

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
