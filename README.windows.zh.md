# Windows 指南 — codex-browser-site-safety-win

[English](README.windows.md) · [中文](README.windows.zh.md) · [返回首页](README.zh.md)

> 本文译自 [README.windows.md](README.windows.md)，**以英文版为准**。命令、路径、版本号保持原样；版本更新时先改英文版，再同步本页。

这是 [bypass-codex-site-safety](README.md) 的 Windows 移植。它打开 Codex / ChatGPT Desktop 的 **Browser Use 本地测试模式**（`BROWSER_USE_SECURITY_MODE=disabled-for-local-testing`），从而对所有 `http://` / `https://` 站点跳过云端 `aura/site_status`（“站点安全策略”）检查。安装同时会修补本地插件缓存，让 `Page.navigationBlocked` 在开关打开时被忽略。

## 本 fork 的增量：运行时 browser-service 副本

上游的 Windows 版只改了**插件缓存**。在 Codex Desktop `26.1013.x` 上，有一部分 `node_repl` 是由插件服务（`node.exe ./server.mjs`、`cua-repl.mjs`）拉起的，它们**不继承** `[mcp_servers.node_repl.env]`，而是加载 **cua_node 运行时**里的 browser service，所以环境变量到不了 site-status 闸门。

本 fork 额外把能找到的每一份安全模式解析函数都改了，让"读不到环境变量"等同于本地测试模式：

```text
function <resolver>(t){let e=Nt(t,<VAR>);return <list>.find(r=>r===e)??<default>}
                                                              ^^^^^^^^^
                     becomes ??"disabled-for-local-testing"
```

覆盖的文件：

- `~/.codex/plugins/cache/openai-bundled/{browser,chrome}/*/scripts/browser-service.mjs`
- `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\<hash>\bin\node_modules\@oai\browser-desktop\scripts\browser-service.mjs`
- `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\<hash>\bin\node_modules\@oai\cua\dist\lib\js\oai_js_browser\dist\skill\scripts\browser-service.mjs`

watcher（`--persist`）和 `CodexBrowserLabRepair` 计划任务会在 Codex 重写配置或替换运行时之后重新打补丁。每个被改写的文件都会备份到 `~/.codex/mcp-wrappers/backups/`；运行时副本旁边还会留一份 `browser-service.mjs.bak-ftmode-<timestamp>`。

## 环境要求

| 组件 | 要求 |
|---|---|
| Windows | 10 或 11（x64） |
| Codex / ChatGPT Desktop | 已安装并至少启动过一次，确保 `%USERPROFILE%\.codex\config.toml` 存在 |
| Python | 3.9+（`python`、`python3` 或 `py` 启动器在 PATH 中） |

实测版本：Codex Desktop `26.1013.4894.0` / plugin `26.1013.40934`（本 fork），以及 `26.915.4065.0` / plugin `26.915.31945`（上游）。

## 版本兼容矩阵

下表只有前两行做过端到端实测。其余版本一律"先跑 `status`"——当某个构建不再包含这个开关时，安装器不会假装成功。

| Codex Desktop（包版本） | Plugin | cua_node runtime | 状态 | 需要什么 |
|---|---|---|---|---|
| `26.1013.4894.0` | `26.1013.40934` | `78fb63cda1a9e2b2` | **本 fork 实测** | env 表 + 插件缓存 + 运行时 resolver 补丁 |
| `26.915.4065.0` | `26.915.31945` | — | 上游实测 | env 表 + 插件缓存 |
| 其他 `26.1013.x` | 任意 | 任意 | 预期可用，需 `status` 验证 | runtime hash 目录按通配匹配 |
| 更旧 / 未知 | 任意 | 任意 | 未验证 | 先跑 `status`，不要盲改 |

查看自己的版本：

```powershell
(Get-AppxPackage OpenAI.Codex).Version
Get-ChildItem "$env:USERPROFILE\.codex\plugins\cache\openai-bundled\browser" -Directory |
    Select-Object -ExpandProperty Name
Get-ChildItem "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node" -Directory |
    Select-Object -ExpandProperty Name
```

`status` 是权威判断：`flag support ok` 表示当前 `browser-service.mjs` 里仍然同时存在 `check-url-site-status` 和 `disabled-for-local-testing`。如果输出 `no longer implements`，请停下——机制变了，盲改没有意义。

## 安装

```powershell
git clone https://github.com/h5htrs4t95-alt/codex-browser-site-safety-win.git
cd codex-browser-site-safety-win
powershell -ExecutionPolicy Bypass -File .\install.ps1 install --persist
```

`install --persist` 是最小安装，不会改 origin 白名单。完全不带参数时，`install.ps1` 执行的是 `install --preset shopping-cn --persist`，会额外把淘宝 / 天猫 / 1688 / alicdn / 支付宝加进白名单。

然后**新开一个 Codex 会话**（原因见下）。

## Windows 特有的两个障碍

Codex 桌面版在 Windows 上有两点和 macOS 不同，它们决定了整个设计。两点都在 `26.915.4065.0`（plugin `26.915.31945`）上验证过：

**1. Codex 每次启动都会重写自己管理的配置段。**
`[mcp_servers.node_repl]`（`args`、`command`、`env_vars`、`startup_timeout_sec`）、它的 `[mcp_servers.node_repl.env]` 表以及插件的 `.mcp.json` 都会被 Codex 用自己的模板重写。往里加的任何键——包括 `env_vars` 里的——下次启动就会被删掉。Codex 不管理的表（例如 `[browser_use.origins.*]`）不会被动。

**2. Codex 自己合成 `node_repl` 的环境，不继承父进程。**
环境变量会沿 `explorer.exe → ChatGPT.exe → codex.exe` 传递，但 `node_repl.exe` 子进程拿到的是合成环境：标准系统变量 + `env` 表 + `env_vars` 白名单。所以把 `BROWSER_USE_SECURITY_MODE` 设成 Windows 用户环境变量（`setx`、`HKCU\Environment`）**没有效果**——它到不了 `node_repl`。

因此，对于 `codex.exe` 直接拉起的 `node_repl`，唯一有效的通道是 `[mcp_servers.node_repl.env]`，在每次 Codex 启动**之后**重新写入。插件服务拉起的进程根本看不到这张表——它们由[本 fork 的运行时 resolver 补丁](#本-fork-的增量运行时-browser-service-副本)覆盖。Windows 上的 `--persist` 就是做这件事：

| 机制 | 触发方式 | 作用 |
|---|---|---|
| **Watcher**（`watch`） | HKCU `...\Run` 自启；每 2 秒轮询 | 在 Codex 重写后约 2 秒内重新打补丁 |
| **计划任务** `CodexBrowserLabRepair` | 每 5 分钟（`--interval MIN`） | watcher 没在跑时的兜底 |

安装的是当前用户级任务，不需要管理员权限。Codex 是**按会话懒加载** `node_repl` 的，不跟应用启动绑定。所以顺序是：

1. Codex 启动，抹掉补丁。
2. watcher 在约 2 秒内补回。
3. 你**新开一个 Codex 会话** → 它的 `node_repl` 在读取已经补好的配置后启动，带上开关。

在 watcher 运行之前就预热好的 `node_repl` 会保留旧环境——这正是 `status` 里显示 `mode=<unset>` 的那些进程。

## macOS 与 Windows 的差异

| | macOS | Windows |
|---|---|---|
| 注入点 | bash wrapper `exec` 签名过的 `node_repl`；`command` 被重定向 | 只用 `[mcp_servers.node_repl.env]`——`command` 保持官方二进制 |
| Wrapper 文件 | `~/.codex/mcp-wrappers/node-repl-security-mode.sh` | 无 |
| 持久化 | 带 `WatchPaths` 的 LaunchAgent | HKCU `Run` watcher + 计划任务 |
| `node_repl` 发现 | `/Applications/{ChatGPT,Codex}.app/...` | `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\*\bin\node_repl.exe`，否则用 `config.toml` 里已有的 `command` |
| 实时检查 | `pgrep` + `ps eww` | toolhelp32 + 读取每个进程的 PEB 环境块（`ReadProcessMemory`，x64） |
| `Page.navigationBlocked` | 改缓存里的 `browser-service.mjs` | 相同 |

## 常用命令

```powershell
# 是否已生效？
.\install.ps1 status

# 只跳过 site_status，不动 origin 白名单
.\install.ps1 install --persist

# 额外加本地白名单（可选）
.\install.ps1 install --persist --preset shopping-cn --allow pixiv.net

# 把兜底周期调慢（分钟）
.\install.ps1 install --persist --interval 15

# 预览，不写文件
.\install.ps1 install --preset shopping-cn --dry-run

# 前台运行 watcher（调试）
.\install.ps1 watch --interval 2

# 卸载（保留 origin 白名单）
.\install.ps1 uninstall
```

## 会修改哪些文件

| 路径 | 作用 |
|---|---|
| `~/.codex/config.toml` 的 `[mcp_servers.node_repl.env]` | 加入 `BROWSER_USE_SECURITY_MODE`（真正的开关） |
| `~/.codex/config.toml` 的 `[browser_use.origins.*]` | 可选的 origin 白名单（`--preset` / `--allow`） |
| `~/.codex/browser/config.toml` | 可选的 `allowed` 数组 |
| `~/.codex/plugins/cache/openai-bundled/unified-computer-use/*/.mcp.json` | CUA 的 env 开关 |
| `~/.codex/plugins/cache/.../{chrome,browser}/*/scripts/browser-service.mjs` | 开关打开时忽略 CDP `Page.navigationBlocked` |
| `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\*/bin/node_modules\@oai\.../browser-service.mjs` | 安全模式默认值（覆盖插件服务拉起的 `node_repl`） |
| `~/.codex/mcp-wrappers/` | 工具安装副本、`repair` shim、watcher 日志、所有被改文件的 `backups/` |
| `HKCU\...\CurrentVersion\Run\CodexBrowserLabWatch` | watcher 自启 |
| 计划任务 `CodexBrowserLabRepair` | 周期修复 |

`backups/` 会在写入前保留每个文件的带时间戳副本；命名基于相对 `CODEX_HOME` 的路径，所以 `config.toml` 和 `browser/config.toml` 的备份不会互相覆盖。

## 报错对照表

模型会转述工具报错。请按**底层 reason** 匹配，不要按模型的说法匹配。

| 底层 reason / 文案 | 闸门 | Windows 是否覆盖 | 处理方式 |
|---|---|---|---|
| `site_status_blocked` — “The site-safety policy blocks this action; no user permission prompt or Auto-review was attempted.” | `browser-service.mjs` 里对云端 `aura/site_status` 的检查 | **覆盖**——env 开关 + resolver 默认值 | 重启 Codex，开**新**会话；`status` 必须显示 `config ok True` |
| `site_status_unavailable` — “The site-safety check was unavailable.” | 同一闸门，网络 / 5xx 失败 | 不适用 | 临时性问题；重试并检查代理 |
| `browser_navigation_blocked` — “Browser Use is not permitted on `<url>`.” | CDP `Page.navigationBlocked` | **覆盖**——已在各副本里插入守卫 | 重跑 `.\install.ps1 install --persist`；新会话 |
| `navigation_url_policy_blocked` — “The browser URL policy blocks this action.” | 本地 URL 策略 | 部分覆盖——开关也会跳过 `check-navigation-url-policy` | 确认该进程拿到了开关（见排查） |
| `blockedURL`（`-10015`）/ “Computer Use is not allowed on the current browser URL” | Computer Use 原生 helper | **不覆盖**——仅 macOS | 本工具不管；见 `README.computer-use.md` |
| `windows_runtime_framework_identity_unavailable … failureReason=no-package-identity` | 直接从 `WindowsApps\...\app\ChatGPT.exe` 启动应用 | 不适用 | 从开始菜单 / `shell:AppsFolder\OpenAI.Codex_2p2nqsd0c76g0!App` 启动，不要直接跑 exe |
| “no workarounds” 之类的拒绝 | 模型上下文，不是工具报错 | 不适用 | 该会话已经见过拒绝；换新会话 |
| 重启电脑后：显示登录页 + 英文界面 | 系统代理（`127.0.0.1:<port>`）没运行，token 刷新超时 | 不适用 | 启动代理客户端；不是本补丁导致的 |

中文界面下，同一个 `site_status_blocked` 常被转述成「浏览器安全策略阻止读取 …」或「站点安全策略阻止了此操作」。底层 reason 仍然是 `site_status_blocked`。

## 自动修复机制

Codex 每次启动都会重写自己管理的配置，升级还会替换插件和运行时目录。因此工具盯着三层：

| 层 | 文件 | 何时重打 |
|---|---|---|
| 1. MCP env | `~/.codex/config.toml` → `[mcp_servers.node_repl.env]` | 键消失时 |
| 2. 插件缓存 | `~/.codex/plugins/cache/openai-bundled/{browser,chrome}/*/scripts/browser-service.mjs` | 文件指纹变化时 |
| 3. 运行时副本 | `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\*\bin\node_modules\@oai\...\browser-service.mjs` | 文件指纹变化时 |

触发方式：

- **watcher** — `HKCU\...\Run\CodexBrowserLabWatch`，每 2 秒轮询；
- **计划任务** — `CodexBrowserLabRepair`，每 5 分钟兜底；
- 两者都走同一条幂等的 `install --persist` 路径，重复执行无害。

Codex 启动或升级后的时序：

```text
Codex 启动 / 升级
      │
      ├─ 重写 config.toml 和/或替换运行时目录
      │
      └─ 约 2 秒内 watcher 检测到变化并重打：
         config env + 插件缓存守卫 + 运行时 resolver 默认值
      │
      └─ 你新开一个会话
            └─ node_repl 带着补好的 env / 代码启动
```

日志：

```powershell
Get-Content ~\.codex\mcp-wrappers\codex-browser-lab.log -Tail 20
```

自动修复**做不到**的事：

- 改变已经在运行的 `node_repl` 进程的环境——必须新开会话（或完全重启应用）；
- 给未来某个代码结构已变的 `browser-service.mjs` 打补丁——watcher 只会写 `WARNING … no longer implements`，不会硬猜；
- 覆盖 Computer Use 或 macOS 的二进制补丁。

## 排查是哪一层失效

1. **配置层** — `.\install.ps1 status` 必须显示 `config security disabled-for-local-testing` 和 `config ok True`。
2. **运行中的进程** — 每条 `live pid` 行显示该进程实际携带的环境。`mode=<unset>` 表示它是在补丁之前启动的；`mode=disabled-for-local-testing` 才是健康状态。
3. **判断进程走的是哪条启动路径**（config env 还是运行时副本）：

```powershell
Get-CimInstance Win32_Process -Filter "Name='node_repl.exe'" |
  ForEach-Object {
    $p = Get-CimInstance Win32_Process -Filter "ProcessId=$($_.ParentProcessId)"
    [pscustomobject]@{ Pid = $_.ProcessId; Parent = $p.Name; Cmd = $p.CommandLine }
  } | Format-Table -Wrap
```

   `Parent=codex.exe` → 走 config env 路径。`Parent=node.exe`（`server.mjs` / `cua-repl.mjs`）→ 走运行时副本路径。

4. **确认每份副本里的 resolver 都在**：

```powershell
Select-String -Path `
  "$env:USERPROFILE\.codex\plugins\cache\openai-bundled\browser\*\scripts\browser-service.mjs", `
  "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node\*\bin\node_modules\@oai\browser-desktop\scripts\browser-service.mjs", `
  "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node\*\bin\node_modules\@oai\cua\dist\lib\js\oai_js_browser\dist\skill\scripts\browser-service.mjs" `
  -Pattern 'disabled-for-local-testing"' | Select-Object Path
```

   所有存在的副本都应该被列出。少一个就重跑 `.\install.ps1 install --persist`。

## 升级操作手册（Codex 升级后）

1. 完全退出 Codex（含托盘图标）。
2. `.\install.ps1 status` — 记下新的 plugin / runtime 版本。
3. `.\install.ps1 install --persist`。
4. 启动 Codex，使用**新**会话。
5. `.\install.ps1 status` — 期望看到 `flag support ok`、`config ok True`；第一次浏览器操作后，应有 `live pid` 显示 `mode=disabled-for-local-testing`。

如果第 3 步报 `flag support … no longer implements`，停下，带上兼容矩阵里的版本号开 issue。

## 验证

```powershell
.\install.ps1 status
```

健康输出：

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
live pid=1234       mode=disabled-for-local-testing      <- 新开会话后出现
```

当运行时副本是最新文件时，`flag support` 可能显示 `browser-desktop` 而不是插件版本号；只要状态是 `ok` 就正常。

退出码：`0` 已生效，`2` 配置未打补丁，`3` 配置已打但运行中的进程仍是旧环境（新开会话）。

观察 watcher：

```powershell
Get-Content ~\.codex\mcp-wrappers\codex-browser-lab.log -Tail 20
```

`mode=<unreadable>` 表示 `ReadProcessMemory` 被拒绝，或进程不是 x64。无论哪种，以配置检查为准。

## 卸载

```powershell
.\install.ps1 uninstall
```

会停止 watcher、删除 HKCU 自启项、删除 `CodexBrowserLabRepair` 和 `.cmd` shim，并从 `config.toml` 和 CUA `.mcp.json` 里移除 `BROWSER_USE_SECURITY_MODE`。origin 白名单会保留，除非你自己从 `config.toml` 里删。

它**不会**还原 `browser-service.mjs`。要撤销这些改动，从 `~/.codex/mcp-wrappers/backups/` 拷回，或使用运行时文件旁边保存的 `browser-service.mjs.bak-ftmode-<timestamp>`。

## 限制

- **必须新开会话：** 补丁只对之后启动的 `node_repl` 生效。应用启动时就打开的会话会保留旧环境。
- **模型拒绝：** 如果老会话已经见过 `site_status` 报错，模型可能拒绝继续（“no workarounds”）。换新会话。
- **Computer Use：** Chrome 停在被拦 URL 上仍可能中断 Computer Use 会话。本工具只覆盖 Browser Use / Chrome 插件。
- **升级：** Codex/ChatGPT 升级会重写 `config.toml`。只要 watcher 在跑就能覆盖；没装 `--persist` 的话，重跑 `.\install.ps1`。版本相关路径会自动跟随：插件版本目录（`unified-computer-use/*/`、`browser/*/`）、运行时 hash 目录（`runtimes/cua_node/*/`）和 `@oai/...` 副本都是通配匹配，`node_repl` 从 Codex 自己写入的 `command` 解析，所以 runtime hash 变化无需手动处理。升级唯一可能拿走的是开关本身——`status` 会报告 `flag support`，如果新的 `browser-service.mjs` 不再实现 `disabled-for-local-testing`，watcher 会在日志里写 `WARNING`。
- **实时环境检查仅限 x64：** ARM64 / 32 位 Windows 仍会打配置补丁，但 `status` 可能显示 `mode=<unreadable>`。
- **副作用：** `disabled-for-local-testing` 跳过的不止 `check-url-site-status`——它还会关闭 `browser-history-read`、`browser-origin-access`、`check-navigation-url-policy`、`file-download`、`file-upload`、`full-cdp`、`page-asset-*`、`raw-cdp-destination-url` 的用户确认提示。浏览器代理会在你的登录态 profile 里获得更大自主权。卸载可恢复这些提示。

## 免责声明

非官方项目，与 OpenAI 无关。`disabled-for-local-testing` 跳过的不仅是站点安全检查：它同时关闭浏览历史读取、origin 访问、文件下载/上传、完整 CDP 和页面资源的用户确认提示，浏览器代理会在你的登录态 profile 里获得更大自主权。建议使用独立的浏览器 profile，自行确认厂商条款，风险自负。

## 许可证

MIT（与上游相同）。见 [LICENSE](LICENSE)。
