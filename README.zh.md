# codex-browser-site-safety-win

[bypass-codex-site-safety](https://github.com/leixyou/bypass-codex-site-safety) 的 Windows fork。

[English](README.md) · [中文](README.zh.md) · [Windows 指南](README.windows.md) · [Computer Use](README.computer-use.md)

> **Windows fork 额外补丁。** 上游的 Windows 版只改了插件缓存。Codex Desktop `26.1013.x` 上，有一部分 `node_repl` 由插件服务（`node.exe ./server.mjs`、`cua-repl.mjs`）拉起，**不继承** `[mcp_servers.node_repl.env]`；它们加载的是 `cua_node` 运行时里的 `browser-service.mjs`，环境变量到不了 site-status 闸门。本 fork 把这些运行时副本也一起改了。详见 [README.windows.md → Fork additions](README.windows.md#fork-additions-runtime-browser-service-copies)。
>
> 实测环境：Codex Desktop `26.1013.4894.0`（Windows 11），runtime `78fb63cda1a9e2b2`，plugin `26.1013.40934`。

### Windows 快速开始

```powershell
git clone https://github.com/h5htrs4t95-alt/codex-browser-site-safety-win.git
cd codex-browser-site-safety-win
powershell -ExecutionPolicy Bypass -File .\install.ps1 install --persist
```

装完**完全退出 Codex（含托盘图标）再重开**，然后新开会话；老会话里已经拉起的 `node_repl` 仍是旧环境。

Windows 指南按"参考资料"来写：

- [版本兼容矩阵](README.windows.md#compatibility-matrix) — 实测过的版本、怎么读 `status`
- [报错对照表](README.windows.md#error-map) — 每条底层原因的归属，以及本工具是否覆盖
- [自动修复机制](README.windows.md#how-auto-repair-works) — 三层补丁和触发条件
- [排查是哪一层失效](README.windows.md#diagnose-which-layer-failed)
- [升级操作手册](README.windows.md#upgrade-playbook-codex-update)

在本机绕过 Codex / ChatGPT Desktop 的 **站点安全 / URL 策略**。拦法有两套，破解方式也不同：

| 闸门 | 典型报错 | 本仓库怎么过 |
|---|---|---|
| **Browser Use**（`browser-service.mjs`） | `site-safety policy` / `Browser use is not permitted on https://…` | 打开官方插件里已有的 `BROWSER_USE_SECURITY_MODE=disabled-for-local-testing`（`./install.sh`） |
| **Computer Use**（`SkyComputerUseService`） | `blockedURL`（`-10015`）/ 「Computer Use is not allowed on the current browser URL. Stop your work…」 | **破解本机 helper 二进制**（`./install-computer-use.sh`）。没有等价环境变量 |

Codex 打开淘宝、1688、Pixiv 等站点时，Browser Use 经常直接报：

```text
Browser Use rejected this action due to browser security policy.
Reason: The site-safety policy blocks this action; no user permission prompt or Auto-review was attempted.
Browser use is not permitted on https://www.taobao.com.
```

Settings 里 Allow browsing、`~/.codex/browser/config.toml` 白名单都压不过这层。原因是 **云端** `site_status`，不是 Chrome 扩展权限。`./install.sh` 打开客户端里已经存在的 local-testing 开关，跳过 `check-url-site-status`。ChatGPT 一更新会把 `~/.codex/config.toml` 改回去，所以默认安装带自动回补。

**Computer Use 是另一套原生程序。** `BROWSER_USE_SECURITY_MODE` 进不去 `SkyComputerUseService`。它走 Aura URL 策略 / URL 黑名单，返回 Sky 错误 `blockedURL`（`-10015`），再由 `SkyComputerUseClient` 塞进「停下手头工作」的杀会话文案。界面里「当前 URL 不允许 Computer Use」是模型把这句英文意译出来的。所以破解是：

- 补丁 `~/.codex/computer-use/Codex Computer Use.app` 里的 `isForbiddenComputerUseTarget` / `allowsForbiddenComputerUseTargets`
- `throwMappedServerError` 对 `blockedURL`（`-10015`）直接返回，不再把被拦 Chrome URL 变成杀会话 RPC
- 改写 service / client / lock-screen guardian 里的杀会话字符串
- 写入 `defaults` `ComputerUseAllowForbiddenTargets=true`
- JS overlay 把 `-10015` 不当致命 RPC 错误
- 对本机 helper 做 ad-hoc 重签，只保留 unrestricted entitlements（库存 SIP 路径；保留 team entitlements 会 AMFI `-424`）
- 注入 `cua-appgroup.dylib`，没有 OpenAI team entitlements 时 native pipe 仍可用
- 安装 LaunchAgent `com.codex-computer-use-lab.repair`——ChatGPT 启动会把官方 helper 拷回来并改写 `.mcp.json`

**支持库存 SIP，不需要关闭 SIP。** Apple Silicon 要打开 **Developer Mode**（设置 → 隐私与安全性 → 开发者模式，然后重启）。`./install-computer-use.sh` 和 `status` 会打印 SIP / Developer Mode；Developer Mode 没开时安装退出码为 `2`。

在 ChatGPT.app **26.924.22138** 上，连续执行 `./install.sh` 和 `./install-computer-use.sh`，Cmd+Q 退出后再**新开** Computer Use，已验证可用。

GitHub [Release](https://github.com/leixyou/bypass-codex-site-safety/releases/tag/computer-use-v3) 发的是 **补丁安装器**，不是破解好的 ChatGPT.app。详见 [README.computer-use.md](README.computer-use.md)。

与 OpenAI 无关。不 MITM `chatgpt.com`。Browser Use 还会改本地插件缓存，让 local-testing 开启时忽略 `Page.navigationBlocked`；应用更新会覆盖该文件，`--persist` 会再打回去。

## 实测版本

本机验证过：

| 组件 | 版本 |
|---|---|
| ChatGPT.app | **26.924.22138** |
| Browser / Chrome 插件（`openai-bundled`） | **26.924.22138** |
| `BROWSER_USE_CODEX_APP_VERSION` | **26.924.22138** |
| Codex Computer Use.app | **26.923.1001242**（Computer Use URL 拦截 **已验证可过**） |

该版本的 `browser-service.mjs` 仍会把 `BROWSER_USE_SECURITY_MODE=disabled-for-local-testing` 映射为跳过 `check-url-site-status`。ChatGPT 更新常会升这个版本并改写 `node_repl`；再跑 `./install.sh status`，或依赖 `--persist`。

Windows（Codex Desktop **26.915.4065.0** / 插件 **26.915.31945**）见 [README.windows.md](README.windows.md)。Computer Use 原生 URL 策略是另一套 helper，见 [README.computer-use.md](README.computer-use.md)。

---

## 其他站点要不要手动加？

**不用。** local-testing 模式一旦生效，云端 site-safety 检查对 **所有** `http://` / `https://` 站点都跳过。不是每加一个网站就跑一次 `--allow`。

`--preset` / `--allow` 只写 **本地 origin 白名单**。同一安全模式下 origin-access 也会被跳过，所以白名单是双保险，不是绕过 site-safety 的必要条件。

| 层 | `./install.sh` 之后 |
|---|---|
| 云端 `aura/site_status`（就是 “site-safety policy” 那条报错） | 对所有站点跳过 |
| 本地 origin 白名单 | 可选；默认会写上 `shopping-cn` |
| Computer Use「当前 Chrome URL 不允许」 | 另一套原生 helper — 跑 `./install-computer-use.sh` |
| 旧会话里模型看到 `site_status` 后拒绕过 | **新开** Codex thread |

`localhost` / `127.0.0.1` 本来就不做 `site_status`。

### 微信小程序后台点某个功能仍被拦

能打开后台，但一点具体功能就报「浏览器安全策略禁止操作当前网址」——这经常 **不是** `site_status`。点击后 Chrome CDP 会发 `Page.navigationBlocked`，插件再抛 `browser_navigation_blocked`，文案同样是 “not permitted”。local-testing 原先不跳过这条。

`./install.sh` 现在会顺手改缓存里的 `browser-service.mjs`，在开关打开时忽略该事件。然后 **新开 thread**。

仍然分开的：

- **Computer Use 操作 Chrome**，前台是被拦域名——原生 `blockedURL`（`-10015`）。这层用 `./install-computer-use.sh`。
- 微信后台检测到 Chrome debugger，自己弹出安全验证页。
- 模型确认策略（发布 / 支付 / 改权限）。需要你在对话里明确授权该操作。

只需要网页时用 `@Chrome` / Browser Use 认领已登录标签。需要桌面 Computer Use 时，先跑 `./install-computer-use.sh`。

---

## 一键安装（macOS Computer Use）

Computer Use 在 Chrome 停在被拦 URL 上时会用原生 `blockedURL`（`-10015`）杀掉会话。这层**不受** `BROWSER_USE_SECURITY_MODE` 控制。

```bash
chmod +x install-computer-use.sh
./install-computer-use.sh
./install-computer-use.sh status
```

Apple Silicon 先打开 Developer Mode 再重启。安装器**不会**要求关闭 SIP。退出码 `2` 表示补丁已打上，但 Developer Mode 没开，helper 起不来。

改的是 `~/.codex/computer-use/Codex Computer Use.app`（含 `blockedURL` `-10015`）和 JS overlay，并装 LaunchAgent，避免 ChatGPT 启动时把官方 helper 盖回来。然后重启 ChatGPT.app，**新开** Computer Use。详见 [README.computer-use.md](README.computer-use.md)。

---

## 一键安装（macOS）

需要：macOS、已安装 ChatGPT.app 或 Codex.app（实测 **26.917.62051**）、Python 3.9+。

```bash
git clone https://github.com/leixyou/bypass-codex-site-safety.git
cd bypass-codex-site-safety
chmod +x install.sh
./install.sh
```

无参数时等于：

```bash
python3 codex_browser_lab.py install --preset shopping-cn --persist
```

然后 **新开一条 Codex thread**（或重启 ChatGPT.app）。已经在跑的 `node_repl` 不会继承新环境变量。

`shopping-cn` 预设会放行：

`taobao.com` · `tmall.com` · `1688.com` · `alicdn.com` · `alipay.com`（主域 + 子域）

这只是方便项。真正全局跳过 site-safety 的是 security mode。

---

## 一键安装（Windows）

需要：Windows 10/11、已安装 Codex / ChatGPT Desktop（先运行一次，让
`%USERPROFILE%\.codex\config.toml` 生成）、Python 3.9+。

```powershell
git clone https://github.com/leixyou/bypass-codex-site-safety.git
cd bypass-codex-site-safety
powershell -ExecutionPolicy Bypass -File .\install.ps1
```

无参数时等于：

```powershell
python codex_browser_lab.py install --preset shopping-cn --persist
```

然后 **重启 Codex / ChatGPT Desktop**（或新开一条 Codex thread）。已经在跑的
`node_repl.exe` 不会继承新环境变量。

Windows 上不需要 wrapper：开关直接写进 `[mcp_servers.node_repl.env]` 表，`command` 仍指向
官方二进制。注意 Codex **每次启动都会重写整个 `node_repl` 块**，而且 `node_repl` 的环境由
Codex 自己拼装（所以 `setx` 的系统变量传不进去）——`--persist` 因此装了一个小守护进程，
在约两秒内把补丁补回去，之后 **新开一条 Codex thread** 即可生效。详见
[README.windows.md](README.windows.md)。

---

## 这不是浏览器插件

检查发生在 Codex 的 Node 插件 `browser-service.mjs`，每次导航都会请求：

```http
GET https://chatgpt.com/backend-api/aura/site_status
    ?site_url=<目标 URL>
    &url_request_source=codex_browser_use
```

`feature_status.agent === true` 就抛 `site_status_blocked`。官方 Chrome 扩展只是 CDP / native pipe 后端，拦不住这条请求。把官方 Browser Use 再封装成 MCP，只要还走这份 `browser-service.mjs`，淘宝照样挂。

相关 issue：

- [openai/codex#29343](https://github.com/openai/codex/issues/29343) — 1688 / Taobao
- [openai/codex#42932](https://github.com/openai/codex/issues/42932) — Pixiv，用户白名单无效

---

## 命令

```bash
# 是否生效
./install.sh status

# 只跳过 site_status，不动 origin 白名单
python3 codex_browser_lab.py install --persist

# 额外写本地白名单（可选）
python3 codex_browser_lab.py install --persist --preset shopping-cn --allow pixiv.net

# 预览，不写盘
python3 codex_browser_lab.py install --preset shopping-cn --dry-run

# 卸载（保留 origin 白名单）
python3 codex_browser_lab.py uninstall
```

`--persist` 会安装一个自动回补的守护：

| 平台 | 机制 |
|---|---|
| macOS | `~/Library/LaunchAgents/com.codex-browser-lab.repair.plist` |
| Windows | HKCU `Run` 守护进程，约 2 秒内自动补回补丁；另加 `CodexBrowserLabRepair` 计划任务（每 5 分钟，`--interval MIN` 可调）兜底 |

监控 `~/.codex/config.toml` 和插件缓存；ChatGPT 更新改回去之后自动 `repair`。

---

## 改了哪些文件

| 路径 | 作用 |
|---|---|
| `~/.codex/mcp-wrappers/node-repl-security-mode.sh` | wrapper：export 开关，再 `exec` 原版签名过的 `node_repl` |
| `~/.codex/config.toml` `[mcp_servers.node_repl]` | `command` 指向 wrapper；写入 `BROWSER_USE_SECURITY_MODE` |
| `~/.codex/config.toml` `[browser_use.origins.*]` | 可选 origin 放行表 |
| `~/.codex/browser/config.toml` | 可选 `allowed` 数组 |
| `~/.codex/plugins/cache/openai-bundled/unified-computer-use/*/.mcp.json` | CUA 环境变量 + `CUA_REPL_NODE_REPL_PATH` |
| `~/.codex/plugins/cache/.../{chrome,browser}/*/scripts/browser-service.mjs` | local-testing 开启时忽略 CDP `Page.navigationBlocked` |

wrapper 用 `exec`，运行中的进程镜像仍是官方 `node_repl`，native pipe 的代码签名身份不变。

**Windows** 上不生成 wrapper：`command` 保持指向官方 `node_repl.exe`，开关写进
`[mcp_servers.node_repl.env]`，`Page.navigationBlocked` 的缓存补丁仍然会打。Codex
每次启动都会重写 `node_repl` 块，所以靠守护进程回补。见 [README.windows.md](README.windows.md)。

`./install.sh` **不会做的事：**

- 不劫持 `chatgpt.com/backend-api/aura/site_status`
- 不改 ChatGPT.app 应用包（Browser Use 只动本地插件缓存，更新后由 `--persist` 再打）
- 不单独破解 Computer Use — 那是 `./install-computer-use.sh`（改本机 `~/.codex/computer-use` helper，并 ad-hoc 重签）

---

## 怎么确认生效

```bash
python3 codex_browser_lab.py status
```

期望：

```text
config command      .../mcp-wrappers/node-repl-security-mode.sh
config security     disabled-for-local-testing
config ok           True
live pid=...        mode=disabled-for-local-testing
```

`config ok true` 但 live process 仍是 `<unset>`：配置已写上，当前会话还是旧进程。新开 thread。

更新 ChatGPT 后再跑一次 `status`。若 `config ok false` 且装了 `--persist`，等几秒让守护回补；没有 persist 就再执行 `./install.sh` 或 `.\install.ps1`。

---

## 限制

- **模型拒绕过**：旧会话里已经吃过 `site_status` 错误时，模型可能按「禁止 workaround」拒绝继续。换新 thread。
- **Computer Use**：`./install.sh` 不管这层。原生 helper 用 `./install-computer-use.sh`（`blockedURL` / `-10015`）。
- **更新覆盖**：插件缓存目录名随版本变。`--persist` 就是为这个准备的。
- 已支持 macOS（LaunchAgent）与 Windows（HKCU Run 守护进程 + 计划任务）。

---

## 卸载

```bash
python3 codex_browser_lab.py uninstall
```

恢复官方 `node_repl` 路径、删掉 env 开关，以及 LaunchAgent / Windows 守护。origin 白名单默认保留，要删自己改 `config.toml`。

## License

MIT
