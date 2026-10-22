# codex-browser-site-safety-win

Windows fork of [bypass-codex-site-safety](https://github.com/leixyou/bypass-codex-site-safety).

[English](README.md) · [中文](README.zh.md) · [Windows guide](README.windows.md) · [Computer Use](README.computer-use.md)

> **Windows fork — extra runtime patch.** Upstream's Windows port only patched the plugin cache. On Codex Desktop `26.1013.x`, some `node_repl` processes are spawned by plugin helpers (`node.exe ./server.mjs`, `cua-repl.mjs`) and **never inherit** `[mcp_servers.node_repl.env]`; they load `browser-service.mjs` from the `cua_node` runtime, so the env flag never reaches the site-status gate. This fork patches those runtime copies too. Details: [README.windows.md → Fork additions](README.windows.md#fork-additions-runtime-browser-service-copies).
>
> Tested on Codex Desktop `26.1013.4894.0` (Windows 11), runtime `78fb63cda1a9e2b2`, plugin `26.1013.40934`.

### Windows quick start

```powershell
git clone https://github.com/h5htrs4t95-alt/codex-browser-site-safety-win.git
cd codex-browser-site-safety-win
powershell -ExecutionPolicy Bypass -File .\install.ps1 install --persist
```

Fully quit Codex (including the tray icon), reopen it, and use a **new thread**
(old threads keep their already-spawned `node_repl`).

Bypass Codex / ChatGPT Desktop **site-safety / URL policy** on your own machine. There are two separate gates:

| Gate | Typical error | What this repo does |
|---|---|---|
| **Browser Use** (`browser-service.mjs`) | `site-safety policy` / `Browser use is not permitted on https://…` | Turn on the official plugin flag `BROWSER_USE_SECURITY_MODE=disabled-for-local-testing` (`./install.sh`) |
| **Computer Use** (`SkyComputerUseService`) | `blockedURL` (`-10015`) / “Computer Use is not allowed on the current browser URL. Stop your work…” | **Binary patch** of the local helper (`./install-computer-use.sh`). There is no env-flag equivalent |

When Codex opens Taobao, 1688, Pixiv, and similar sites, Browser Use often fails immediately with:

```text
Browser Use rejected this action due to browser security policy.
Reason: The site-safety policy blocks this action; no user permission prompt or Auto-review was attempted.
Browser use is not permitted on https://www.taobao.com.
```

Allowing the site in Settings or in `~/.codex/browser/config.toml` does not override this. The decision is a **cloud** `site_status` check, not a Chrome-extension permission. `./install.sh` turns on a flag the official plugin already implements, which skips `check-url-site-status`. ChatGPT updates rewrite `~/.codex/config.toml`, so the default install also adds a watcher that re-applies that patch.

**Computer Use is a different binary.** `BROWSER_USE_SECURITY_MODE` never reaches `SkyComputerUseService`. That helper calls Aura URL policy / a URL blocklist, returns Sky error `blockedURL` (`-10015`), and `SkyComputerUseClient` injects the “stop your work” instruction (the Chinese UI line “当前 URL 不允许 Computer Use” is the model paraphrasing that English kill string). The crack is therefore:

- patch `isForbiddenComputerUseTarget` / `allowsForbiddenComputerUseTargets` in `~/.codex/computer-use/Codex Computer Use.app`
- skip `throwMappedServerError` for `blockedURL` (`-10015`) so a blocked Chrome URL is not turned into a session-killing RPC error
- rewrite the kill string in the service, client, and lock-screen guardian
- set `defaults` `ComputerUseAllowForbiddenTargets=true`
- overlay JS so `-10015` is not treated as a fatal RPC error
- ad-hoc re-sign the local helper with unrestricted entitlements only (stock SIP; AMFI `-424` if team entitlements are kept)
- inject `cua-appgroup.dylib` so the native pipe still works without OpenAI team entitlements
- install LaunchAgent `com.codex-computer-use-lab.repair` — ChatGPT recopies the official helper and rewrites `.mcp.json` on launch

**Stock SIP is the supported path. Disabling SIP is not required.** Apple Silicon needs **Developer Mode** on (System Settings → Privacy & Security → Developer Mode, then restart). `./install-computer-use.sh` and `status` print SIP + Developer Mode; install exits `2` if Developer Mode is off.

Verified working on ChatGPT.app **26.924.22138** after `./install.sh` **and** `./install-computer-use.sh`, then a full quit (Cmd+Q) and a **new** Computer Use turn.

The GitHub [release](https://github.com/leixyou/bypass-codex-site-safety/releases/tag/computer-use-v3) ships **the patcher**, not a redistributed ChatGPT.app. Details: [README.computer-use.md](README.computer-use.md).

Not affiliated with OpenAI. Does not MITM `chatgpt.com`. Browser Use also edits the local plugin cache so `Page.navigationBlocked` is ignored while the flag is on; app updates overwrite that file and `--persist` puts the guard back.

## Tested on

Verified locally against:

| Component | Version |
|---|---|
| ChatGPT.app | **26.924.22138** |
| Browser / Chrome plugin (`openai-bundled`) | **26.924.22138** |
| `BROWSER_USE_CODEX_APP_VERSION` | **26.924.22138** |
| Codex Computer Use.app | **26.923.1001242** (Computer Use URL kill bypass **verified**) |

On this build, `browser-service.mjs` still maps `BROWSER_USE_SECURITY_MODE=disabled-for-local-testing` to skipping `check-url-site-status`. ChatGPT updates often bump this version and rewrite `node_repl`; re-run `./install.sh status` or rely on `--persist`.

Windows (Codex Desktop **26.915.4065.0** / plugin **26.915.31945**) is documented in [README.windows.md](README.windows.md). Native Computer Use URL policy is a separate helper; see [README.computer-use.md](README.computer-use.md).

---

## Do I have to add every site?

**No.** Once local-testing mode is on, the cloud site-safety check is skipped for **all** `http://` and `https://` hosts. You do not run `--allow` per site to bypass *site-safety policy*.

`--preset` / `--allow` only write **local origin allowlists**. In this same security mode, origin-access is also skipped, so those lists are belt-and-suspenders, not required for the bypass.

| Layer | After `./install.sh` |
|---|---|
| Cloud `aura/site_status` (the “site-safety policy” error) | Skipped for every site |
| Local origin allowlist | Optional; `shopping-cn` is applied by default |
| Computer Use “this Chrome URL is not allowed” | Separate native helper — run `./install-computer-use.sh` |
| Model refusing after an old `site_status` error | Use a **new** Codex thread |

`localhost` / `127.0.0.1` were already exempt from `site_status`.

### Still blocked on WeChat mini-program admin (or similar)

If **open** works but **clicking a feature** fails with “browser security policy / not permitted on the current URL”, that is often **not** `site_status`. After a click, Chrome CDP can emit `Page.navigationBlocked`; the plugin then throws `browser_navigation_blocked` with the same “not permitted” wording. Local-testing mode did not skip that path.

`./install.sh` now also patches cached `browser-service.mjs` so that event is ignored while the flag is on. Then start a **new** thread.

Still separate:

- **Computer Use on Chrome** while a blocked host is the front tab — native `blockedURL` (`-10015`). Run `./install-computer-use.sh` for that helper.
- WeChat admin detecting Chrome’s debugger and showing its own security page.
- Model confirmation policy (publish / pay / change permissions). Say explicitly that you authorize that action.

Prefer `@Chrome` / Browser Use on a tab you already logged in when you only need the page. Use Computer Use after `./install-computer-use.sh` when you need the desktop helper.

---

## One-shot install (macOS Computer Use)

Native Computer Use kills the session with `blockedURL` (`-10015`) when Chrome
is on a blocked URL. That path is **not** `BROWSER_USE_SECURITY_MODE`.

```bash
chmod +x install-computer-use.sh
./install-computer-use.sh
./install-computer-use.sh status
```

Apple Silicon: turn on Developer Mode first, then restart. The installer
does **not** ask you to disable SIP. Exit code `2` means the helper is
patched but will not launch until Developer Mode is on.

Patches `~/.codex/computer-use/Codex Computer Use.app` (including `blockedURL`
`-10015`) and a JS overlay, then installs a LaunchAgent so ChatGPT cannot keep
restoring the official helper. Restart ChatGPT.app, then start a new Computer
Use turn. Details: [README.computer-use.md](README.computer-use.md).

---

## One-shot install (macOS)

Needs: macOS, ChatGPT.app or Codex.app (tested **26.917.62051**), Python 3.9+.

```bash
git clone https://github.com/leixyou/bypass-codex-site-safety.git
cd bypass-codex-site-safety
chmod +x install.sh
./install.sh
```

With no arguments that is:

```bash
python3 codex_browser_lab.py install --preset shopping-cn --persist
```

Then start a **new Codex thread** (or restart ChatGPT.app). Already-running `node_repl` processes keep the old environment.

The `shopping-cn` preset allowlists:

`taobao.com` · `tmall.com` · `1688.com` · `alicdn.com` · `alipay.com` (apex + subdomains)

That preset is convenience only. Security mode is what bypasses site-safety globally.

---

## One-shot install (Windows)

Needs: Windows 10/11, Codex / ChatGPT Desktop installed (run it once so
`%USERPROFILE%\.codex\config.toml` exists), Python 3.9+.

```powershell
git clone https://github.com/leixyou/bypass-codex-site-safety.git
cd bypass-codex-site-safety
powershell -ExecutionPolicy Bypass -File .\install.ps1
```

With no arguments that is:

```powershell
python codex_browser_lab.py install --preset shopping-cn --persist
```

Then **restart Codex / ChatGPT Desktop** (or start a new Codex thread).
Already-running `node_repl.exe` processes keep the old environment.

On Windows there is no wrapper: the flag goes into the
`[mcp_servers.node_repl.env]` table and `command` keeps pointing at the official
binary. Note that Codex regenerates that whole block on **every launch**, and it
builds `node_repl`'s environment itself (so a `setx` variable never reaches it) —
`--persist` therefore installs a small watcher that re-applies the patch within
about two seconds, after which a **new Codex thread** picks it up. Full details:
[README.windows.md](README.windows.md).

---

## This is not a Chrome extension

The check runs in Codex’s Node plugin `browser-service.mjs`. Every navigation hits:

```http
GET https://chatgpt.com/backend-api/aura/site_status
    ?site_url=<target URL>
    &url_request_source=codex_browser_use
```

If `feature_status.agent === true`, the plugin throws `site_status_blocked`. The official Chrome extension is only the CDP / native-pipe backend; it never sees this request. Wrapping official Browser Use as MCP still fails on Taobao if it still loads that `browser-service.mjs`.

Related issues:

- [openai/codex#29343](https://github.com/openai/codex/issues/29343) — 1688 / Taobao
- [openai/codex#42932](https://github.com/openai/codex/issues/42932) — Pixiv; user allowlist ignored

---

## Commands

```bash
# is it active?
./install.sh status

# skip site_status only; do not touch origin allowlists
python3 codex_browser_lab.py install --persist

# extra local allowlist entries (optional)
python3 codex_browser_lab.py install --persist --preset shopping-cn --allow pixiv.net

# preview, no writes
python3 codex_browser_lab.py install --preset shopping-cn --dry-run

# uninstall (keeps origin allowlists)
python3 codex_browser_lab.py uninstall
```

`--persist` installs a watcher that re-applies the patch:

| Platform | Mechanism |
|---|---|
| macOS | `~/Library/LaunchAgents/com.codex-browser-lab.repair.plist` |
| Windows | an HKCU `Run` watcher that re-applies the patch within ~2 s, plus the `CodexBrowserLabRepair` Scheduled Task (every 5 min; `--interval MIN`) as a safety net |

It watches `~/.codex/config.toml` and the plugin cache. After a ChatGPT update reverts the change, `repair` runs again.

---

## Files it changes

| Path | Role |
|---|---|
| `~/.codex/mcp-wrappers/node-repl-security-mode.sh` | Wrapper: export the flag, then `exec` the signed `node_repl` |
| `~/.codex/config.toml` `[mcp_servers.node_repl]` | Point `command` at the wrapper; set `BROWSER_USE_SECURITY_MODE` |
| `~/.codex/config.toml` `[browser_use.origins.*]` | Optional origin allow tables |
| `~/.codex/browser/config.toml` | Optional `allowed` arrays |
| `~/.codex/plugins/cache/openai-bundled/unified-computer-use/*/.mcp.json` | CUA env + `CUA_REPL_NODE_REPL_PATH` |
| `~/.codex/plugins/cache/.../{chrome,browser}/*/scripts/browser-service.mjs` | Ignore CDP `Page.navigationBlocked` while local-testing mode is on |

The wrapper uses `exec`, so the running image is still official `node_repl` and the native-pipe code-signing identity does not change.

On **Windows** no wrapper is created: `command` stays the official `node_repl.exe`,
the flag goes in `[mcp_servers.node_repl.env]`, and the `Page.navigationBlocked`
cache edit still applies. Codex rewrites the `node_repl` block on every launch, so
a watcher re-applies the patch — see [README.windows.md](README.windows.md).

`./install.sh` does **not**:

- Intercept `chatgpt.com/backend-api/aura/site_status`
- Patch the ChatGPT.app bundle (Browser Use only edits the local plugin cache, re-applied after updates)
- Patch Computer Use by itself — that is `./install-computer-use.sh` (local `~/.codex/computer-use` helper, ad-hoc re-signed)

---

## Verify

```bash
python3 codex_browser_lab.py status
```

You want:

```text
config command      .../mcp-wrappers/node-repl-security-mode.sh
config security     disabled-for-local-testing
config ok           True
live pid=...        mode=disabled-for-local-testing
```

`config ok true` but live processes still `<unset>`: the file is patched, the current session is an old process. Open a new thread.

After a ChatGPT update, run `status` again. If `config ok false` and you installed `--persist`, wait a few seconds for the watcher. Without persist, re-run `./install.sh` or `.\install.ps1`.

---

## Limits

- **Model refusal:** if an old thread already saw a `site_status` error, the model may refuse to continue (“no workarounds”). Use a new thread.
- **Computer Use:** `./install.sh` does not patch it. Use `./install-computer-use.sh` for the native helper (`blockedURL` / `-10015`).
- **Updates:** plugin cache directory names change with the app version. That is why `--persist` exists.
- Supported on macOS (LaunchAgent) and Windows (HKCU Run watcher + Scheduled Task).

---

## Uninstall

```bash
python3 codex_browser_lab.py uninstall
```

Restores the official `node_repl` path, removes the env flag, and removes the LaunchAgent / Windows watcher. Origin allowlists are left in place unless you delete them from `config.toml` yourself.

## License

MIT
