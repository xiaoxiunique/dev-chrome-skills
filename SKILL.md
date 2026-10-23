---
name: dev-chrome-skills
description: |
  DevChromeSkills —— 统一管理本机 Debug Chrome 实例（「全占的工具箱」里的 Dev Profile）。
  任何需要用 Chrome 浏览器完成的任务（打开网页、登录、抓 Cookie、CDP / chrome-devtools MCP / js-reverse MCP
  调试、爬虫逆向、自动化）都必须先用本 skill 查询、创建、启动实例，禁止自己另起 Chrome 或用临时 user-data-dir。
  用户提到「Debug Chrome」「调试浏览器」「9222」「远程调试端口」「CDP」「chrome-devtools MCP」「开个浏览器」时使用。
---

# DevChromeSkills

本机所有调试用 Chrome 都由 macOS App「全占的工具箱」（`/Applications/全占的工具箱.app`）的 **Dev Profile** 统一管理：
每个 Profile = 一个独立的 `--user-data-dir`（登录态互相隔离）+ 一个调试端口（默认共用 `9222`）+ 可选 Charles 代理。
命令行工具 `devchrome`（同目录 `scripts/devchrome`）**独立运行，不需要打开 App**：它直接读写与 App 相同的配置文件
`~/Library/Application Support/Browser Modes/debug-profiles.json`，用与 App 相同的数据目录和启动参数拉起 Chrome，
所以 CLI 和 App 管理的是同一批实例，App 开着时它的 Debug 页面也会显示 CLI 的改动。

## 铁律

1. **不准自己创建浏览器。** 禁止直接运行 `Google Chrome --remote-debugging-port=…`、`open -na "Google Chrome" --args …`、
   puppeteer/playwright `launch()`、chrome-devtools MCP 的 `--isolated` / 自带 `--userDataDir`、或任何临时 profile 目录。
   需要浏览器 → 用 `devchrome` 复用或新建 Dev Profile。
2. **先查再建。** 动手前先 `devchrome list`，同平台同用途已有 Profile 就复用（登录态在里面），不要重复创建。
3. **不碰外部进程。** 端口被非 Dev Profile 的进程占用时，报告给用户（`devchrome who <端口>` 的输出），不要 kill。
4. **不手改配置。** 不要编辑 `debug-profiles.json` 或 Profile 数据目录，一律走 `devchrome`（或 App 的 Debug 页面）。
5. **删除要用户确认。** 删除会把数据目录（含登录态）移到废纸篓。
6. 用完是否停止：用户没说就保持运行（方便复用）；任务明确结束或用户要求时 `devchrome stop`。

## 前置条件

- 不需要 App 在运行。需要 `/Applications/Google Chrome.app`、`jq`、`curl`、`lsof`；`cookies` / `set-cookies` 另需 `node` ≥ 22。
- 环境变量（一般不用设）：`DEVCHROME_HOME` 覆盖配置目录，`DEVCHROME_CHROME_APP` 覆盖 Chrome 路径。
- `devchrome` 已软链到 `~/.local/bin/devchrome`（在 PATH 上），任意目录直接运行 `devchrome …` 即可。
  若 `command -v devchrome` 找不到，用完整路径 `<本 skill 目录>/scripts/devchrome`（Claude Code：`~/.claude/skills/dev-chrome-skills/scripts/devchrome`；
  Codex：`~/.codex/skills/dev-chrome-skills/scripts/devchrome`）。

## 命名规范

- 名称格式：`<平台>-<用途>[-序号]`，全小写英文 / 数字，用 `-` 连接，≤ 48 字符，全局唯一（不区分大小写）。
  `devchrome create` / `rename` 会强制校验。
- 平台用常见缩写：`douyin`、`xhs`（小红书）、`xianyu`、`kuaishou`、`weibo`、`bili`、`wechat`、`grok`、`adobe`、`coingecko`、`google` …
  无具体平台用 `general`。
- 用途：`login`（保持登录态）、`crawl`（采集）、`reverse`（逆向调试）、`test`（临时测试）、`autologin`、`capture`（配合抓包）…
- 同平台同用途需要多个账号 / 隔离环境时加序号：`douyin-crawl`、`douyin-crawl-2`。
- ID（`profile-<毫秒时间戳>`）和数据目录自动分配，不要自定义；名称才是给人和 agent 看的。
- 数据目录默认：`default` → `~/Library/Application Support/Browser Modes/Chrome Debug`，
  其它 → `~/Library/Application Support/Browser Modes/Chrome Debug Profiles/<id>`。
  只有用户明确要求时才用 `--dir`（必须是空目录或不存在，且不能在正常浏览器数据目录里）。

## 常用流程

```bash
devchrome list                         # 查询已创建的全部实例（名称/ID/端口/状态/代理/数据目录）
devchrome list --json                  # 同上，JSON
devchrome show douyin-crawl            # 单个实例详情 + CDP /json/version

devchrome create douyin-crawl          # 新建（默认端口 9222）
devchrome start douyin-crawl           # 启动；已运行则直接复用。成功后打印 CDP 地址
devchrome stop douyin-crawl
devchrome rename "抖音 debug" douyin-login   # 须先停止
devchrome delete douyin-test --yes     # 须先停止，且须用户确认
devchrome cookies douyin-crawl         # 读运行中实例的 Cookie，按已打开的站点汇总（JSON）
devchrome cookies douyin-crawl douyin.com   # 只输出该站点的 Cookie 串 k=v; k=v
devchrome set-cookies douyin-crawl douyin.com "a=1; b=2"
                                       # 覆盖写入：须先在该实例里打开 douyin.com 的标签页；先清空该站点全部 Cookie，
                                       # 再以 domain=.douyin.com、path=/、secure=(任一标签是 https) 写入，写完刷新这些标签页
                                       # 可带 Cookie: 前缀，; 或换行分隔，同名后者覆盖；输出 {cleared, written, reloadedTabs}
devchrome proxy douyin-crawl on        # 启动时走 127.0.0.1:8888（Charles）；--port N 改代理端口；须先停止
devchrome proxy douyin-crawl off
```

`<名称|ID>` 两者都能用；名称匹配不区分大小写。

## 端口：9222 与冲突处理

**标准端口是 9222。** 所有 Dev Profile 默认共用 9222，所以同一时间只能有一个 Profile 跑在 9222 上；
本机已配好的 chrome-devtools MCP / js-reverse MCP 也都连 `http://127.0.0.1:9222`。

启动前 `devchrome start` 会检查端口：

| 情况 | 处理 |
|---|---|
| 9222 空闲 | 直接在 9222 启动 |
| 目标 Profile 本身已在运行 | 直接复用，不重启 |
| 9222 被**另一个 Dev Profile** 占用 | 默认报错。若那个实例已不需要 → `devchrome stop <它>` 再启动；若两个都要用 → `--port auto` |
| 9222 被**外部进程**占用（不是 Dev Profile） | 默认报错并打印占用进程。报告用户，**不要 kill**；需要立刻并行工作时用 `--port auto` |

```bash
devchrome who 9222                     # 看谁占着 9222
devchrome start douyin-crawl --port auto   # 自动在 9223–9299 选一个空闲且未被其它 Profile 预留的端口，并写回该 Profile
devchrome start douyin-crawl --port 9230   # 指定端口（会写回 Profile 配置）
devchrome free-port                    # 只查一个可用端口
devchrome set-port douyin-crawl 9222   # 用完把端口改回 9222（须先停止）
```

注意：端口改动会持久化到 Profile 配置里。临时改用非 9222 端口的实例，任务结束后建议 `stop` + `set-port … 9222` 还原，
否则下次 MCP 默认连不到它。

## 用 MCP 管理 / 操作浏览器

MCP 只负责**连接已由 devchrome 启动的实例**，绝不让 MCP 自己拉起浏览器。

本机现有配置（Codex `~/.codex/config.toml`）：

```toml
[mcp_servers.chrome-devtools]
command = "npx"
args = ["chrome-devtools-mcp@latest", "--browserUrl", "http://127.0.0.1:9222"]

[mcp_servers.js-reverse]
command = "node"
args = ["/Users/not/projects/devs/逆向/mcp/JSReverser-MCP/build/src/index.js", "--browserUrl", "http://127.0.0.1:9222"]
```

Claude Code 对应的写法（`~/.claude.json` 的 `mcpServers`，或 `claude mcp add`）：

```bash
claude mcp add chrome-devtools -- npx chrome-devtools-mcp@latest --browserUrl http://127.0.0.1:9222
```

使用步骤：

1. `devchrome list` → 选定或 `create` 一个 Profile。
2. `devchrome start <名称>`，确认输出「CDP 就绪：http://127.0.0.1:9222」。
3. 再调用 MCP 工具（`list_pages`、`navigate_page`、`take_snapshot`、js-reverse 的 hook 等）。
   MCP 报连不上时先 `devchrome show <名称>` 看 `reachable`，不要改用 MCP 的自启动模式。
4. 实例不在 9222 上（`--port auto` 之后）：已配置的 MCP 连不到它。chrome-devtools 的操作直接用 `devchrome cdp <名称> …`（见下节）；一定要用 MCP 时二选一：
   - 首选：停掉占用 9222 的 Dev Profile（外部进程需用户同意），把目标 Profile `set-port … 9222` 后再启动；
   - 或临时给 MCP 换地址：`chrome-devtools-mcp --browserUrl http://127.0.0.1:<端口>`（需要用户同意改 MCP 配置并重启会话）。
5. 先确认 MCP 连的是哪个实例：`devchrome who 9222` 应显示目标 Profile；若显示外部进程，你操作的不是 Dev Profile，先报告。

MCP 配置里禁止出现：`--isolated`、`--userDataDir`、`--headless` 自启动、`--autoConnect`（会连到日常浏览器）。

## 多实例并行：`devchrome cdp`（不受 9222 限制）

要同时操作多个实例（如 9222 / 9223 / 9224），或者目标不在 9222 上时，不要改 MCP 配置，用 `devchrome cdp`。
它包装了官方 `chrome-devtools` CLI（随 `chrome-devtools-mcp` 安装），**工具名和参数与 chrome-devtools MCP 完全相同**：

```bash
devchrome cdp xhs-login list_pages                     # 名称 / ID / 端口号都行
devchrome cdp 9223 take_snapshot 1                     # 位置参数：pageId
devchrome cdp douyin-crawl click 1 <uid>
devchrome cdp douyin-crawl navigate_page 1 --url https://www.douyin.com
devchrome cdp 9224 evaluate_script "() => location.href" --pageId 1
devchrome cdp 9223 take_screenshot 1 --filePath /tmp/x.png
devchrome cdp 9223 <工具> --help                        # 查某个工具的参数
```

- 每个端口一个独立的 daemon（session id = 端口，带 `--browserUrl http://127.0.0.1:<端口>`），实例之间互不影响，可以并行调用。
  daemon 第一次调用时自动启动，版本不一致时自动重启；`devchrome stop` 会顺带停掉它。
- 目标实例必须已经 `devchrome start` 且可达，否则直接报错。**禁止直接运行裸的 `chrome-devtools <工具>`**：
  它自动起的 daemon 不带 `--browserUrl`，会自己拉起一个 Chrome。
- 输出默认是 Markdown，要 JSON 加 `--output-format json`。
- js-reverse 没有 CLI，hook / 断点仍然只能通过 MCP（连 9222）用：逆向任务要把目标 Profile 跑在 9222 上。

## 启动参数（与 App 一致，供排查）

`--user-data-dir=<数据目录> --remote-debugging-address=127.0.0.1 --remote-debugging-port=<端口> --no-first-run
--no-default-browser-check --disable-extensions --restore-last-session`，开启代理时加 `--proxy-server=http://127.0.0.1:<代理端口>`（Charles 默认 8888）。
运行状态按「命令行以 Chrome 可执行文件开头、带 `--user-data-dir=<该 Profile 数据目录>` 的主进程」判断；
`running(cdp-down)` 表示进程在但调试端口不是它在监听（刚启动或端口被别人抢了）。
