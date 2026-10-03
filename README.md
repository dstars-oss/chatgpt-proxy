# chatgpt-proxy

chatgpt-proxy is an MVP launcher for the Microsoft Store ChatGPT app on Windows. The Store package is
still named `OpenAI.Codex`; chatgpt-proxy starts its packaged ChatGPT app with Chromium proxy flags and
per-process proxy environment variables.

## Scope

- Launches the installed Store package `app\ChatGPT.exe` through a package-context helper so
  application initialization has the required package identity, even when direct execution is allowed.
- Explicit `--chatgpt-exe` paths inside the installed Store package use the same helper;
  executables outside the package are launched directly.
- Adds `--proxy-server=http://127.0.0.1:7897` by default.
- Sets `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, and `NO_PROXY` only on the launched ChatGPT process.
- Waits for a Codex child process to remain running for at least one second and verifies that it
  inherited the proxy environment variables, avoiding transient bootstrap processes.
- Refuses to launch over an already running `ChatGPT.exe` by default, because startup flags only
  reliably apply to a fresh process.

## Usage

Build:

```powershell
cargo build
```

Build a release executable:

```powershell
cargo build --release
```

Preview installation without copying files or creating a shortcut:

```powershell
.\target\release\chatgpt-proxy.exe install --dry-run
```

Install for the current user:

```powershell
.\target\release\chatgpt-proxy.exe install
```

This copies the running executable to:

```text
%LOCALAPPDATA%\chatgpt-proxy\chatgpt-proxy.exe
```

It also creates a current-user Start Menu shortcut:

```text
%APPDATA%\Microsoft\Windows\Start Menu\Programs\chatgpt-proxy.lnk
```

The shortcut launches `chatgpt-proxy.exe launch` and uses the Microsoft Store `app\ChatGPT.exe` icon.
Running `install` again overwrites the installed executable and refreshes the shortcut. When run
from the installed executable, it skips self-copy and still refreshes the shortcut.

Old installations are not migrated or removed automatically. If present, manually remove
`%LOCALAPPDATA%\MyCodex` and the `MyCodex.lnk` / `Codex++.lnk` shortcuts under
`%APPDATA%\Microsoft\Windows\Start Menu\Programs` when no longer needed.

Preview the launch target and arguments without launching ChatGPT:

```powershell
cargo run -- launch --dry-run
```

Launch with the default local proxy:

```powershell
cargo run -- launch
```

Launch with a custom proxy:

```powershell
cargo run -- launch --proxy http://127.0.0.1:7890
```

Launch a specific ChatGPT executable:

```powershell
cargo run -- launch --chatgpt-exe "C:\Program Files\WindowsApps\OpenAI.Codex_...\app\ChatGPT.exe"
```

The legacy option name `--codex-exe` remains available as an alias.

Launch without proxy environment variables, keeping only Chromium `--proxy-server`:

```powershell
cargo run -- launch --no-env
```

This also removes inherited `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY`, and their
lowercase variants from the launched process.

Launch without checking whether `app-server` inherited proxy environment variables:

```powershell
cargo run -- launch --no-env-check
```

Enable remote debugging for inspection:

```powershell
cargo run -- launch --remote-debugging-port 9229 --remote-allow-origins http://127.0.0.1:9229
```

Pass extra ChatGPT/Electron arguments after `--`:

```powershell
cargo run -- launch -- --some-electron-flag=value
```

If ChatGPT is already running and you intentionally want to skip the fresh-process guard:

```powershell
cargo run -- launch --allow-existing-instance
```

After installation, the installed launcher can be run directly:

```powershell
& "$env:LOCALAPPDATA\chatgpt-proxy\chatgpt-proxy.exe" launch
```

## Current Limitations

- chatgpt-proxy discovers the installed package with `Get-AppxPackage -Name OpenAI.Codex` and selects
  `app\ChatGPT.exe`, falling back to the legacy `app\Codex.exe` when needed. Pass `--chatgpt-exe`
  if discovery fails or the package layout changes.
- Store package launch depends on the Windows PowerShell `Appx` module and
  `Invoke-CommandInDesktopPackage` remaining available.
- The child-process stability and environment check is not a full UI or network readiness check.
  Timeouts shorter than one second cannot complete this check; `--no-env-check` disables it.
- It does not modify `app.asar` or files under `C:\Program Files\WindowsApps`.
- It does not write registry environment values or change the global Windows proxy.
