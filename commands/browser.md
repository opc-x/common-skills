# browser — AI 全功能浏览器

Playwright MCP 驱动的真实 Chromium。导航、点击、填表、截图、读内容——anything。
不用离开 Claude Code，不用手动开浏览器。

## 使用方式

```
/browser                              # 截图当前 localhost（自动检测端口）
/browser <url>                        # 打开任意 URL 截图
/browser <url> <自然语言描述操作>      # 执行多步操作后截图
/browser <描述>                       # 在当前页继续操作（保持上下文）
```

示例：

```
/browser github.com/HcPittBrad/issues
/browser localhost:3000 点击登录，填邮箱 test@x.com 密码 123，提交，截图结果
/browser 往下滚动找到第一个 issue，点进去截图
/browser https://x.com 搜索 claude code，截图前三条结果
```

---

## 执行协议

### Step 0 — 解析输入

```
有 URL？
  YES → 先 navigate 到该 URL
  NO  → 在当前页继续（保持浏览器上下文）

有操作描述？
  YES → 解析为有序动作列表（见 Step 1）
  NO  → 直接截图
```

### Step 1 — 解析自然语言 → 动作序列

把描述拆成原子动作，按顺序执行：

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

每步执行后：
- 关键步骤截图（navigate / 表单提交 / 操作完成后）
- 中间步骤（click/fill）默认不截图，只在最终结果截图
- 用户明确说"每步截图"→ 每步都截

### Step 2 — 执行

按动作列表逐步执行。遇到：
- 元素找不到 → 截图当前状态，描述问题，询问下一步
- 页面跳转 → 自动等待加载后继续
- 报错弹窗 → 截图，描述错误内容

### Step 3 — 输出

```
[截图]
完成：[用一句话说做了什么，结果是什么]

发现问题（如有）：
- [问题1]
- [问题2]
```

没有问题就不输出"发现问题"这行。

---

## 上下文保持

同一对话内，Playwright MCP 保持浏览器会话：
- 登录态保留
- 页面状态保留
- Cookie 保留

直接描述下一步操作即可继续，不需要重新导航。

---

## 端口自动检测

无 URL 时，依次检测：

```bash
lsof -i :3000 | grep LISTEN
lsof -i :3001 | grep LISTEN
lsof -i :8080 | grep LISTEN
lsof -i :4000 | grep LISTEN
```

取第一个在监听的端口。

---

## 原则

- 截图直接内联展示，不说废话
- 操作失败立刻截图说明，不继续硬跑
- 有登录态就用，没有就截图当前状态
- 人说继续就继续，不需要重新描述上下文
