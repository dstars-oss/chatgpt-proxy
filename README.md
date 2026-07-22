# MyCodex

MyCodex is an MVP launcher for the Microsoft Store ChatGPT app on Windows. The Store package is
still named `OpenAI.Codex`; MyCodex starts its packaged ChatGPT app with Chromium proxy flags and
per-process proxy environment variables.

## Scope

- Launches the installed Store package `app\ChatGPT.exe` directly when its ACL permits it.
- Automatically retries through a package-context helper when newer Store package ACLs reject
  direct execution with `ERROR_ACCESS_DENIED`.
- Adds `--proxy-server=http://127.0.0.1:7897` by default.
- Sets `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, and `NO_PROXY` only on the launched ChatGPT process.
- Waits for the Codex `resources\codex.exe app-server` child process and verifies that it inherited
  the proxy environment variables.
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

Install for the current user:

```powershell
.\target\release\mycodex.exe install
```

This copies the running executable to:

```text
%LOCALAPPDATA%\MyCodex\MyCodex.exe
```

It also creates a current-user Start Menu shortcut:

```text
%APPDATA%\Microsoft\Windows\Start Menu\Programs\MyCodex.lnk
```

The shortcut launches `MyCodex.exe launch` and uses the Microsoft Store `app\ChatGPT.exe` icon.
Running `install` again overwrites the installed executable, refreshes the shortcut, and removes the
legacy `Codex++.lnk` shortcut if present.

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
%LOCALAPPDATA%\MyCodex\MyCodex.exe launch
```

## Current Limitations

- MyCodex discovers the installed package with `Get-AppxPackage -Name OpenAI.Codex` and selects
  `app\ChatGPT.exe`, falling back to the legacy `app\Codex.exe` when needed. Pass `--chatgpt-exe`
  if discovery fails or the package layout changes.
- The package-context fallback depends on the Windows PowerShell `Appx` module and
  `Invoke-CommandInDesktopPackage` remaining available.
- It does not modify `app.asar` or files under `C:\Program Files\WindowsApps`.
- It does not write registry environment values or change the global Windows proxy.
