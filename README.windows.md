# Windows guide — codex-browser-site-safety-win

[English](README.windows.md) · [中文](README.windows.zh.md)

Windows port of [bypass-codex-site-safety](README.md). It turns on Codex / ChatGPT
Desktop **Browser Use local-testing mode** (`BROWSER_USE_SECURITY_MODE=disabled-for-local-testing`),
so the cloud `aura/site_status` ("site-safety policy") check is skipped for every
`http://` / `https://` site. The same install also patches the local plugin cache so
`Page.navigationBlocked` is ignored while that flag is on.

## Fork additions: runtime browser-service copies

> 中文读者：完整中文版见 [README.windows.zh.md](README.windows.zh.md)。

The upstream Windows port only patched the **plugin cache**. On Codex Desktop
26.1013.x some `node_repl` processes are spawned by plugin helpers
(`node.exe ./server.mjs`, `cua-repl.mjs`) and never inherit
`[mcp_servers.node_repl.env]`; those processes load the browser service from the
**cua_node runtime** instead, so the env flag never reaches the site-status gate.

This fork additionally patches the security-mode resolver in every copy it can
find, so an unset environment is treated as local-testing mode:

```text
function <resolver>(t){let e=Nt(t,<VAR>);return <list>.find(r=>r===e)??<default>}
                                                              ^^^^^^^^^
                     becomes ??"disabled-for-local-testing"
```

Files covered:

- `~/.codex/plugins/cache/openai-bundled/{browser,chrome}/*/scripts/browser-service.mjs`
- `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\<hash>\bin\node_modules\@oai\browser-desktop\scripts\browser-service.mjs`
- `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\<hash>\bin\node_modules\@oai\cua\dist\lib\js\oai_js_browser\dist\skill\scripts\browser-service.mjs`

The watcher (`--persist`) and the `CodexBrowserLabRepair` task re-apply the
patch after Codex rewrites its config or replaces the runtime. Every rewritten
file is backed up to `~/.codex/mcp-wrappers/backups/`; runtime copies also keep
a `browser-service.mjs.bak-ftmode-<timestamp>` next to the original.

## Requirements

| Component | Needed |
|---|---|
| Windows | 10 or 11 (x64) |
| Codex / ChatGPT Desktop | installed and run once, so `%USERPROFILE%\.codex\config.toml` exists |
| Python | 3.9+ (`python`, `python3` or the `py` launcher on PATH) |

Verified on Codex Desktop `26.1013.4894.0` / plugin `26.1013.40934` (this fork) and
`26.915.4065.0` / plugin `26.915.31945` (upstream).

## Compatibility matrix

Only the two builds below have been exercised end-to-end. Everything else is
"run `status` first" — the installer refuses to pretend when a build no longer
contains the flag.

| Codex Desktop (package) | Plugin | cua_node runtime | Status | What it needs |
|---|---|---|---|---|
| `26.1013.4894.0` | `26.1013.40934` | `78fb63cda1a9e2b2` | **Tested (this fork)** | env table + plugin cache + runtime resolver patch |
| `26.915.4065.0` | `26.915.31945` | — | Tested by upstream | env table + plugin cache |
| other `26.1013.x` builds | any | any | Expected; verify with `status` | runtime hash directory is globbed |
| older / unknown | any | any | Unverified | run `status`; do not patch blindly |

Get your own versions:

```powershell
(Get-AppxPackage OpenAI.Codex).Version
Get-ChildItem "$env:USERPROFILE\.codex\plugins\cache\openai-bundled\browser" -Directory |
    Select-Object -ExpandProperty Name
Get-ChildItem "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node" -Directory |
    Select-Object -ExpandProperty Name
```

`status` is authoritative: `flag support ok` means the installed
`browser-service.mjs` still ships both `check-url-site-status` and
`disabled-for-local-testing`. If it prints `no longer implements`, stop — the
mechanism changed and a blind patch will do nothing.

## Install

```powershell
git clone https://github.com/h5htrs4t95-alt/codex-browser-site-safety-win.git
cd codex-browser-site-safety-win
powershell -ExecutionPolicy Bypass -File .\install.ps1 install --persist
```

`install --persist` is the minimal install: it does not touch origin allowlists.
With no arguments at all, `install.ps1` runs
`install --preset shopping-cn --persist`, which additionally allowlists
taobao / tmall / 1688 / alicdn / alipay.

Then **start a new Codex thread** (see below for why).

## The Windows-specific obstacle

Codex desktop on Windows behaves differently from macOS in two ways that decide
the whole design. Both were verified on build `26.915.4065.0`
(plugin `26.915.31945`):

**1. Codex regenerates its managed config block on every launch.**
`[mcp_servers.node_repl]` (`args`, `command`, `env_vars`, `startup_timeout_sec`),
its `[mcp_servers.node_repl.env]` table and the plugin's `.mcp.json` are rewritten
from Codex's own template. Any key added there — including in `env_vars` — is
removed the next time the app starts. Tables Codex does not own, such as
`[browser_use.origins.*]`, are left alone.

**2. Codex builds `node_repl`'s environment itself; it does not inherit it.**
The variable propagates `explorer.exe → ChatGPT.exe → codex.exe`, but the
`node_repl.exe` children are spawned with a synthesised environment: the standard
system variables plus the `env` table plus the `env_vars` allowlist. Setting
`BROWSER_USE_SECURITY_MODE` as a Windows user environment variable (`setx`,
`HKCU\Environment`) therefore has **no effect** — it never reaches `node_repl`.

So for `node_repl` spawned directly by `codex.exe`, the only working channel is
`[mcp_servers.node_repl.env]`, re-applied *after* each Codex launch.
Processes spawned by plugin helpers never see that table — they are covered by
the runtime resolver patch described in
[Fork additions](#fork-additions-runtime-browser-service-copies) instead.
That is what `--persist` does on Windows:

| Mechanism | Trigger | Purpose |
|---|---|
| **Watcher** (`watch`) | HKCU `...\Run` autostart; polls every 2 s | re-applies the patch within ~2 s of Codex rewriting it |
| **Scheduled Task** `CodexBrowserLabRepair` | every 5 min (`--interval MIN`) | safety net when the watcher is not running |

No logon/elevated Scheduled Task is installed. Codex spawns `node_repl` lazily,
per session — it is not tied to app start. So the sequence is:

1. Codex starts and wipes the patch.
2. The watcher re-applies it within ~2 seconds.
3. You open a **new Codex thread** → its `node_repl` is spawned while reading the
   re-patched config, and carries the flag.

`node_repl` processes warmed up at app start, before the watcher ran, keep the old
environment — that is exactly what `status` reports as `mode=<unset>`.

## macOS vs Windows

| | macOS | Windows |
|---|---|---|
| Injection point | bash wrapper `exec`ing the signed `node_repl`; `command` is repointed | `[mcp_servers.node_repl.env]` only — `command` stays the official binary |
| Wrapper file | `~/.codex/mcp-wrappers/node-repl-security-mode.sh` | none |
| Persistence | LaunchAgent with `WatchPaths` | HKCU `Run` watcher + Scheduled Task |
| `node_repl` discovery | `/Applications/{ChatGPT,Codex}.app/...` | `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\*\bin\node_repl.exe`, else the `command` already in `config.toml` |
| Live check | `pgrep` + `ps eww` | toolhelp32 + reading each process's PEB environment block (`ReadProcessMemory`, x64) |
| `Page.navigationBlocked` | patched in cached `browser-service.mjs` | same |

## Commands

```powershell
# is it active?
.\install.ps1 status

# skip site_status only; do not touch origin allowlists
.\install.ps1 install --persist

# extra local allowlist entries (optional)
.\install.ps1 install --persist --preset shopping-cn --allow pixiv.net

# slower safety net (minutes)
.\install.ps1 install --persist --interval 15

# preview, no writes
.\install.ps1 install --preset shopping-cn --dry-run

# run the watcher in the foreground (debugging)
.\install.ps1 watch --interval 2

# uninstall (keeps origin allowlists)
.\install.ps1 uninstall
```

## Files it changes on Windows

| Path | Role |
|---|---|
| `~/.codex/config.toml` `[mcp_servers.node_repl.env]` | adds `BROWSER_USE_SECURITY_MODE` (the actual bypass) |
| `~/.codex/config.toml` `[browser_use.origins.*]` | optional origin allow tables (`--preset` / `--allow`) |
| `~/.codex/browser/config.toml` | optional `allowed` arrays |
| `~/.codex/plugins/cache/openai-bundled/unified-computer-use/*/.mcp.json` | CUA env flag |
| `~/.codex/plugins/cache/.../{chrome,browser}/*/scripts/browser-service.mjs` | ignore CDP `Page.navigationBlocked` while the flag is on |
| `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\*/bin/node_modules\@oai\.../browser-service.mjs` | security-mode default (covers `node_repl` spawned by plugin helpers) |
| `~/.codex/mcp-wrappers/` | installed tool copy, `repair` shim, watcher log, `backups/` of every file touched |
| `HKCU\...\CurrentVersion\Run\CodexBrowserLabWatch` | autostarts the watcher |
| Scheduled Task `CodexBrowserLabRepair` | periodic repair |

`backups/` keeps a timestamped copy of each file before it is written; names are
derived from the path relative to `CODEX_HOME`, so `config.toml` and
`browser/config.toml` never overwrite each other's backup.

## Error map

The model paraphrases tool errors. Match on the **underlying reason**, not on
the model's wording.

| Underlying reason / message | Gate | Covered on Windows? | What to do |
|---|---|---|---|
| `site_status_blocked` — “The site-safety policy blocks this action; no user permission prompt or Auto-review was attempted.” | cloud `aura/site_status` check in `browser-service.mjs` | **Yes** — env flag + resolver default | restart Codex, open a **new** thread; `status` must show `config ok True` |
| `site_status_unavailable` — “The site-safety check was unavailable.” | same gate, network/5xx | n/a | transient; retry and check your proxy |
| `browser_navigation_blocked` — “Browser Use is not permitted on `<url>`.” | CDP `Page.navigationBlocked` | **Yes** — guard inserted into the service copies | re-run `.\install.ps1 install --persist`; new thread |
| `navigation_url_policy_blocked` — “The browser URL policy blocks this action.” | local URL policy | Partly — the flag also bypasses `check-navigation-url-policy` | confirm the flag reached that process (see Diagnostics) |
| `blockedURL` (`-10015`) / “Computer Use is not allowed on the current browser URL” | Computer Use native helper | **No** — macOS only | not covered; see `README.computer-use.md` |
| `windows_runtime_framework_identity_unavailable … failureReason=no-package-identity` | app launched directly from `WindowsApps\...\app\ChatGPT.exe` | n/a | launch through the Start menu / `shell:AppsFolder\OpenAI.Codex_2p2nqsd0c76g0!App`, not the exe |
| “no workarounds” refusal | model context, not a tool error | n/a | that thread already saw a denial; use a new thread |
| After a reboot: sign-in screen + English UI | a system proxy (`127.0.0.1:<port>`) is not running, so token refresh times out | n/a | start the proxy client; not caused by the patch |

Chinese UI: the same `site_status_blocked` is often paraphrased as
「浏览器安全策略阻止读取 …」 or 「站点安全策略阻止了此操作」. The underlying
reason is still `site_status_blocked`.

## How auto-repair works

Codex rewrites its managed config on every launch, and a Codex update replaces
the plugin and runtime directories. The tool therefore watches three layers:

| Layer | File | Re-applied when |
|---|---|---|
| 1. MCP env | `~/.codex/config.toml` → `[mcp_servers.node_repl.env]` | the key disappears |
| 2. Plugin cache | `~/.codex/plugins/cache/openai-bundled/{browser,chrome}/*/scripts/browser-service.mjs` | the file fingerprint changes |
| 3. Runtime copies | `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\*\bin\node_modules\@oai\...\browser-service.mjs` | the file fingerprint changes |

Triggers:

- **watcher** — `HKCU\...\Run\CodexBrowserLabWatch`, polls every 2 s;
- **scheduled task** — `CodexBrowserLabRepair`, every 5 min as a safety net;
- both call the same idempotent `install --persist` path, so running them twice
  is harmless.

Sequence after a Codex launch or update:

```text
Codex starts / updates
      │
      ├─ rewrites config.toml and/or replaces the runtime directory
      │
      └─ within ~2 s the watcher detects the change and re-patches:
         config env + plugin cache guard + runtime resolver default
      │
      └─ you open a NEW thread
            └─ node_repl is spawned with the patched env / patched code
```

Log:

```powershell
Get-Content ~\.codex\mcp-wrappers\codex-browser-lab.log -Tail 20
```

What auto-repair **cannot** do:

- change the environment of `node_repl` processes that are already running —
  a new thread (or a full app restart) is required;
- patch a future `browser-service.mjs` whose code shape changed — the watcher
  logs `WARNING … no longer implements` instead of guessing;
- cover Computer Use or the macOS binary patch.

## Diagnose which layer failed

1. **Config layer** — `.\install.ps1 status` must show
   `config security disabled-for-local-testing` and `config ok True`.
2. **Running processes** — every `live pid` line shows the environment that
   process actually carries. `mode=<unset>` means it was spawned before the
   patch; `mode=disabled-for-local-testing` is healthy.
3. **Which spawn path owns a process** (config env vs runtime copy):

```powershell
Get-CimInstance Win32_Process -Filter "Name='node_repl.exe'" |
  ForEach-Object {
    $p = Get-CimInstance Win32_Process -Filter "ProcessId=$($_.ParentProcessId)"
    [pscustomobject]@{ Pid = $_.ProcessId; Parent = $p.Name; Cmd = $p.CommandLine }
  } | Format-Table -Wrap
```

   `Parent=codex.exe` → config env path. `Parent=node.exe`
   (`server.mjs` / `cua-repl.mjs`) → runtime copy path.

4. **Resolver actually present in each copy**:

```powershell
Select-String -Path `
  "$env:USERPROFILE\.codex\plugins\cache\openai-bundled\browser\*\scripts\browser-service.mjs", `
  "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node\*\bin\node_modules\@oai\browser-desktop\scripts\browser-service.mjs", `
  "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node\*\bin\node_modules\@oai\cua\dist\lib\js\oai_js_browser\dist\skill\scripts\browser-service.mjs" `
  -Pattern 'disabled-for-local-testing"' | Select-Object Path
```

   Every existing copy should be listed. If one is missing, re-run
   `.\install.ps1 install --persist`.

## Upgrade playbook (Codex update)

1. Fully quit Codex (tray icon too).
2. `.\install.ps1 status` — note the new plugin / runtime version.
3. `.\install.ps1 install --persist`.
4. Start Codex and use a **new** thread.
5. `.\install.ps1 status` — expect `flag support ok`, `config ok True`, and after
   the first browser action a live PID with
   `mode=disabled-for-local-testing`.

If step 3 reports `flag support … no longer implements`, stop and open an issue
with the versions from the compatibility matrix.

## Verify

```powershell
.\install.ps1 status
```

Healthy output:

```text
platform            win32
CODEX_HOME          C:\Users\<you>\.codex
flag support        ok      (browser-service.mjs 26.1013.40934 implements it)
official node_repl  ...\runtimes\cua_node\<hash>\bin\node_repl.exe
watcher             running=True autostart=True
persist             CodexBrowserLabRepair=installed
repair shim         ...\mcp-wrappers\codex-browser-lab-repair.cmd exists=True
config command      ...\runtimes\cua_node\<hash>\bin\node_repl.exe
config security     disabled-for-local-testing
config ok           True
live pid=1234       mode=disabled-for-local-testing      <- after opening a NEW thread
```

`flag support` may name `browser-desktop` instead of a plugin version when the
runtime copy is the newest file; either is healthy as long as it says `ok`.

Exit codes: `0` active, `2` config not patched, `3` config patched but the running
processes still carry the old environment (start a new Codex thread).

Watch the watcher work:

```powershell
Get-Content ~\.codex\mcp-wrappers\codex-browser-lab.log -Tail 20
```

`mode=<unreadable>` for a live PID means `ReadProcessMemory` was refused, or the
process is not x64. The config check is authoritative regardless.

## Uninstall

```powershell
.\install.ps1 uninstall
```

Stops the watcher, removes the HKCU autostart value, deletes `CodexBrowserLabRepair`
and the `.cmd` shim, and strips `BROWSER_USE_SECURITY_MODE` from `config.toml` and
the CUA `.mcp.json`. Origin allowlists are left in place unless you delete them
from `config.toml` yourself.

It does **not** restore `browser-service.mjs`. To undo those edits, copy the
files back from `~/.codex/mcp-wrappers/backups/`, or use the
`browser-service.mjs.bak-ftmode-<timestamp>` copies stored next to the runtime
files.

## Limits

- **New thread required:** the patch only reaches `node_repl` processes spawned
  after it was re-applied. Threads opened at app start keep the old environment.
- **Model refusal:** if an old thread already saw a `site_status` error, the model
  may refuse to continue ("no workarounds"). Use a new thread.
- **Computer Use:** Chrome sitting on a blocked URL can still abort a Computer Use
  session. This tool only covers Browser Use / the Chrome plugin.
- **Updates:** a Codex/ChatGPT update regenerates `config.toml`. The watcher covers
  this as long as it is running; without `--persist`, re-run `.\install.ps1`.
  Version-dependent paths are followed automatically: the plugin version directory
  (`unified-computer-use/*/`, `browser/*/`), the runtime hash directory
  (`runtimes/cua_node/*/`) and the `@oai/...` copies are all globbed, and
  `node_repl` is resolved from the `command` Codex itself writes, so a new
  runtime hash needs no action.
  The one thing an update *can* take away is the flag itself — `status` reports
  `flag support`, and the watcher writes a `WARNING` line to its log if a new
  `browser-service.mjs` no longer implements `disabled-for-local-testing`.
- **Live env check is x64:** ARM64 / 32-bit Windows still patch config; `status`
  may print `mode=<unreadable>` for live processes.
- **Side effects:** `disabled-for-local-testing` bypasses more than
  `check-url-site-status` — it also removes user-consent prompts for
  `browser-history-read`, `browser-origin-access`, `check-navigation-url-policy`,
  `file-download`, `file-upload`, `full-cdp`, `page-asset-*` and
  `raw-cdp-destination-url`. The browser agent becomes more autonomous inside your
  own browser profile. Uninstall to restore the prompts.

## Disclaimer

Unofficial; not affiliated with OpenAI. `disabled-for-local-testing` bypasses
more than the site-status check: it also removes user-consent prompts for
browser-history reads, origin access, file downloads/uploads, full CDP and page
assets, so the agent becomes more autonomous inside your logged-in browser
profile. Use a dedicated browser profile, review the vendor's terms, and use at
your own risk.

## License

MIT (same as upstream). See [LICENSE](LICENSE).
