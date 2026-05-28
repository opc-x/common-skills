# browser — AI 全功能浏览器（三引擎自动路由）

三个浏览器 agent 按任务类型自动路由：
- **Playwright MCP** — 带登录态的 Chrome（Profile 1），实时交互，截图
- **Webwright** — 无状态 Firefox，产出可重用 Python 脚本
- **Libretto** — CDP Chrome，TypeScript 工作流，网络拦截，反爬支持

## 使用方式

```
/browser <url>                        # 截图，自动选引擎
/browser <url> <自然语言操作>          # 执行操作后截图
/browser <描述>                       # 在当前页继续（保持上下文）
```

---

## Step 0 — 路由决策（必须先执行）

```
输入分析
    │
    ├─ 需要已登录账号？快速查看？截图？一次性操作？不确定用哪个？
    │   └→ [A] Playwright MCP   ← 默认 fallback
    │
    ├─ 提取数据？批量？需要脚本？可重复执行？
    │   └→ [B] Webwright
    │
    └─ 生产级自动化？需要 TypeScript 工作流？网络拦截？反爬/CAPTCHA 风险高？
        └→ [C] Libretto
```

**关键路由信号：**

| 信号词 | 路由 |
|--------|------|
| 查一下 / 看看 / 截图 / 已登录网站(X/GitHub/Gmail) / 不确定 | A（默认）|
| 提取 / 爬取 / 数据 / 批量 / 脚本 / 可重复 | B |
| 工作流 / 自动化 / 持久 / TypeScript / 拦截请求 | C |

**当 B 或 C 失败时，降级到 A**（Playwright MCP 是最稳健的 fallback）。

---

## [A] Playwright MCP — 快速交互（Chrome + 你的登录态）

**适用**：需要已登录账号、快速截图、一次性操作

### 动作映射

| 描述关键词 | Playwright 动作 |
|-----------|----------------|
| 打开 / 去 / 导航 | `playwright_navigate` |
| 点击 / 点 / 按 | `playwright_click` |
| 填写 / 输入 / 搜索 | `playwright_fill` |
| 滚动 / 往下 | `playwright_evaluate` (scroll) |
| 等待 / 加载 | `playwright_wait_for_load_state` |
| 读取 / 获取内容 | `playwright_evaluate` (读 DOM) |
| 截图 / 看看 | `playwright_screenshot` |
| 回车 / 提交 | `playwright_press` (Enter) |

### 执行规则

- 关键步骤截图（navigate 后 / 操作完成后）
- 中间步骤（click/fill）不截图，只在最终结果截图
- 元素找不到 → 截图当前状态，描述问题
- 同一对话内保持 session（登录态/Cookie/页面状态持久）

### 无 URL 时自动检测端口

```bash
lsof -i :3000 | grep LISTEN  # 优先
lsof -i :3001 | grep LISTEN
lsof -i :8080 | grep LISTEN
lsof -i :4000 | grep LISTEN
```

---

## [B] Webwright — 脚本化数据提取

**适用**：数据提取、批量操作、需要可重复执行的 Python 脚本

触发方式：通过 **Skill tool** 调用（不是 Bash）：
- 一次性任务 → `Skill("webwright:run", args="<task描述>")`
- 需要复用脚本 → `Skill("webwright:craft", args="<task描述>")`

### 执行模式

- **默认（一次性）**：`webwright:run` — 产出 `final_script.py`
- **CLI 工具（参数化）**：`webwright:craft` — 产出带 argparse 的可重用脚本

### 工作流

```
Plan → Explore(截图探索) → final_script.py → 执行 → 自验证(读截图确认)
```

产出目录：`final_runs/run_<id>/`，含截图、日志、脚本

### session 续接

Webwright 每次执行是无状态的（新 Firefox 进程）。续接上次任务 = 继续编辑同一个 `final_script.py`，不需要重新 explore。

### 使用 context7 增强

遇到 Playwright Python selector / API 问题时主动查文档：
```python
# Step 1: 找 library ID
mcp__plugin_context7_context7__resolve-library-id(libraryName="playwright python")
# Step 2: 按返回的 ID 查具体文档
mcp__plugin_context7_context7__query-docs(context7CompatibleLibraryID="<id>", query="<具体问题>")
```

---

## [C] Libretto — 生产级 TypeScript 工作流

**适用**：需要持久工作流文件、网络请求拦截、anti-bot 场景、TypeScript

触发方式：通过 **Bash tool** 直接运行 `npx libretto` 命令（不是 Skill tool）。

### 核心命令

```bash
npx libretto setup                          # 首次初始化
npx libretto open <url> --headed            # 开启 session（带界面）
npx libretto snapshot --session <name>      # 页面快照（优先用这个观察状态）
npx libretto exec "await page.url()"        # 单步交互/验证选择器
npx libretto run ./workflow.ts              # 执行工作流
npx libretto close --session <name>         # 关闭 session
```

### 默认集成策略（按优先级）

```
1. browser-context fetch()    ← 优先（速度快）
2. page.on('response', ...)   ← 若 fetch() 不安全（被 bot 检测）
3. Playwright UI 自动化        ← 最后手段
```

### CAPTCHA 处理

- 本地 Chrome session：在可见浏览器手动解 CAPTCHA，然后继续
- 云端托管：等待最多 2 分钟自动处理

### session 续接

`npx libretto snapshot / exec` 在同一 session name 下天然保持状态，直接继续操作即可。

### 使用 context7 增强

遇到 Libretto API / TypeScript workflow 问题时主动查文档：
```python
mcp__plugin_context7_context7__resolve-library-id(libraryName="libretto")
mcp__plugin_context7_context7__query-docs(context7CompatibleLibraryID="<id>", query="<具体问题>")
```

---

## 输出格式

```
[截图]
完成：[一句话说明做了什么、结果是什么]

发现问题（如有）：
- [问题1]
```

没有问题不输出"发现问题"。

---

## 原则

- 路由先于行动，路由错误比执行慢更糟糕
- 截图直接内联，不说废话
- 操作失败立刻截图说明，不硬跑
- 同一对话保持 session 上下文，人说继续就继续
