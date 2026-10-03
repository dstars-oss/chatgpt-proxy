# AGENTS.md

## Project

chatgpt-proxy is a standalone Windows launcher for the Microsoft Store ChatGPT app (package identity
`OpenAI.Codex`). It is not a Codex plugin package.

The current verified goal is to start Store ChatGPT with a local proxy without enabling the global
Windows proxy and without modifying files under `C:\Program Files\WindowsApps`.

## Launch Method

The launcher starts the Store app executable with its package identity:

```text
<InstallLocation>\app\ChatGPT.exe --proxy-server=http://127.0.0.1:7897
```

Store package `26.924.2738.0` allows direct process creation but fails during application bootstrap
without a package identity. Do not use `ERROR_ACCESS_DENIED` as the condition for package-context
launch. The launcher uses `Invoke-CommandInDesktopPackage` to start a hidden, transient PowerShell
helper with the package identity. The helper sets the process-only proxy environment and starts the
same executable with the same arguments; it does not use AUMID activation. Explicit executable paths
inside an installed `OpenAI.Codex` package use the same route; external executables use
`std::process::Command` directly.

The direct process or package-context helper sets these environment variables only on the launched
process tree:

```text
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
ALL_PROXY=http://127.0.0.1:7897
NO_PROXY=localhost,127.0.0.1,::1
```

Windows child processes inherit this environment by default, so `resources\codex.exe app-server`
and tools it starts, such as `git.exe`, inherit the proxy env unless Codex explicitly overrides the
environment for that child.

Do not reintroduce the discarded approaches unless explicitly requested:

- `IApplicationActivationManager` / AUMID activation.
- Node `--require` preload injection.
- Registry or user environment writes.
- Local API relay.
- `app.asar` or `WindowsApps` file modification.

## Discovering ChatGPT.exe

When `--chatgpt-exe` is not provided, the launcher discovers the Store package install directory by
running PowerShell:

```powershell
Get-AppxPackage -Name OpenAI.Codex |
  Sort-Object Version -Descending |
  Select-Object -First 1
```

It reads the package `InstallLocation`, then appends:

```text
app\ChatGPT.exe
```

For compatibility with older Store packages, it falls back to `app\Codex.exe` only when
`app\ChatGPT.exe` is absent.

For the current tested package this resolves to:

```text
C:\Program Files\WindowsApps\OpenAI.Codex_26.924.2738.0_x64__2p2nqsd0c76g0\app\ChatGPT.exe
```

If discovery fails or the Store package layout changes, pass an explicit executable path:

```powershell
cargo run -- launch --chatgpt-exe "C:\Program Files\WindowsApps\...\app\ChatGPT.exe"
```

## Verification

Use the narrow checks below after code changes:

```powershell
cargo test
cargo clippy -- -D warnings
cargo build
cargo run -- launch --dry-run
cargo run -- install --dry-run
```

For real launch verification, close existing `ChatGPT.exe` windows first, then run:

```powershell
cargo run -- launch
```

The launcher waits for the same Codex child process to remain running for at least one second and
verifies that it inherited the expected proxy environment variables. This filters transient bootstrap
processes but does not prove UI or network readiness. For real launch verification, also check that
the main window opens and the latest application log has no bootstrap failure. New Store versions
may run a copied `codex.exe` from local app data instead of `resources\codex.exe`.

## Install Command

`install` copies the running executable into the current user's local app data directory:

```text
%LOCALAPPDATA%\chatgpt-proxy\chatgpt-proxy.exe
```

It then creates or refreshes this current-user Start Menu shortcut:

```text
%APPDATA%\Microsoft\Windows\Start Menu\Programs\chatgpt-proxy.lnk
```

The shortcut target is the installed launcher, its arguments are:

```text
launch
```

The shortcut icon must come from the Microsoft Store ChatGPT executable discovered by the normal
Store app path lookup:

```text
<InstallLocation>\app\ChatGPT.exe,0
```

The install command supports overwrite installation. If the launcher is already running from
`%LOCALAPPDATA%\chatgpt-proxy\chatgpt-proxy.exe`, skip self-copy and still refresh the shortcut.
Old installations are not migrated or removed automatically. Users manually remove the old
`%LOCALAPPDATA%\MyCodex` directory and `MyCodex.lnk` / `Codex++.lnk` Start Menu shortcuts.
