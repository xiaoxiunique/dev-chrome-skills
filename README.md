# dev-chrome-skills

`devchrome`：在 macOS 上管理多个相互隔离的 Debug Chrome 实例（Dev Profile），并以 Agent Skill 的形式给 Claude Code / Codex 使用。

- 每个 Profile 是一个独立的 `--user-data-dir`，登录态互相隔离，再加一个 CDP 调试端口（默认 9222），可选走 Charles 代理
- 可以读写 Cookie（`cookies` / `set-cookies`）
- 多个实例可以并行操作：`devchrome cdp <实例> <工具>`，每个端口有一个独立的 chrome-devtools daemon，互不影响
- Skill 规定 agent 只能复用或新建 Dev Profile，不能自己另起浏览器或使用临时 profile

## 依赖

- macOS，`/Applications/Google Chrome.app`
- `jq`、`curl`、`lsof`
- `node` ≥ 22（`cookies` / `set-cookies` 需要）
- `chrome-devtools-mcp`（`cdp` 需要）：`npm install -g chrome-devtools-mcp@latest`

## 安装

```bash
# Claude Code
git clone https://github.com/xiaoxiunique/dev-chrome-skills ~/.claude/skills/dev-chrome-skills
# Codex
# git clone https://github.com/xiaoxiunique/dev-chrome-skills ~/.codex/skills/dev-chrome-skills

ln -s ~/.claude/skills/dev-chrome-skills/scripts/devchrome ~/.local/bin/devchrome   # 确保 ~/.local/bin 在 PATH 上
```

## 常用命令

```bash
devchrome list                              # 列出全部实例
devchrome create douyin-crawl               # 新建（名称格式 <平台>-<用途>[-序号]）
devchrome start douyin-crawl                # 启动（默认 9222）
devchrome start xhs-login --port auto       # 9222 被占时自动挑一个空闲端口
devchrome cdp xhs-login list_pages          # 用 chrome-devtools 的工具操作指定实例
devchrome cdp 9223 take_snapshot 1
devchrome cookies douyin-crawl douyin.com   # 导出某站点的 Cookie 串
devchrome stop douyin-crawl
devchrome help                              # 全部命令
```

配置保存在 `~/Library/Application Support/Browser Modes/debug-profiles.json`，不存在时会自动创建。
可以用 `DEVCHROME_HOME` 换配置目录，用 `DEVCHROME_CHROME_APP` 换 Chrome 路径。

Agent 的完整使用规则见 [SKILL.md](SKILL.md)。
