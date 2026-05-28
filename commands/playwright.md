# /playwright — Playwright 全能浏览器

两种模式：
- **默认（Claude 独驾）**：Claude 开一个可见 Chrome，用户旁观
- **CDP 共享（人机同车）**：用户先开 Chrome，Claude 接入同一个窗口，双方都能操作

---

## 启动前：用 ToolSearch 确认工具可用

```
ToolSearch("select:mcp__playwright__browser_navigate,mcp__playwright__browser_take_screenshot")
```

若无结果 → playwright MCP 未加载，提示用户重启 Claude Code，然后停止。

---

## 模式一：默认模式（Claude 独驾，用户可见）

无需任何准备，Claude Code 启动时自动开启。
Chrome 窗口在用户屏幕可见，用户可旁观，Claude 控制操作。

**预登录（继承登录态）：**
```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --user-data-dir=/Users/cuijianchen/.playwright-profile
```
登录完关闭，之后 Claude 自动继承 cookie。

---

## 模式二：CDP 共享模式（人机同一浏览器窗口）

用户先用以下命令开 Chrome：
```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir=/Users/cuijianchen/.playwright-profile
```

然后告诉 Claude：「用 CDP 模式连接」

Claude 改用 CDP 工具 `mcp__playwright__browser_run_code_unsafe` 接入：
```js
async (page) => {
  // 已连接到用户现有的 Chrome 窗口
}
```

或临时切换 MCP config：`--cdp-endpoint http://localhost:9222`

---

## 工具名称（完整 mcp__playwright__ 前缀）

| 操作 | 工具全名 |
|------|---------|
| 导航 / 打开 URL | `mcp__playwright__browser_navigate` |
| 后退 | `mcp__playwright__browser_navigate_back` |
| 截图 | `mcp__playwright__browser_take_screenshot` |
| 读取 DOM / 快照 | `mcp__playwright__browser_snapshot` |
| 点击（元素） | `mcp__playwright__browser_click` |
| 输入文字 | `mcp__playwright__browser_type` |
| 填写多个表单字段 | `mcp__playwright__browser_fill_form` |
| 按键 | `mcp__playwright__browser_press_key` |
| 下拉选择 | `mcp__playwright__browser_select_option` |
| 悬停 | `mcp__playwright__browser_hover` |
| 拖拽（元素间） | `mcp__playwright__browser_drag` |
| 鼠标移动 | `mcp__playwright__browser_mouse_move_xy` |
| 鼠标点击坐标 | `mcp__playwright__browser_mouse_click_xy` |
| 鼠标拖拽坐标 | `mcp__playwright__browser_mouse_drag_xy` |
| 鼠标按下 / 释放 | `mcp__playwright__browser_mouse_down` / `mcp__playwright__browser_mouse_up` |
| 滚轮 | `mcp__playwright__browser_mouse_wheel` |
| 上传文件 | `mcp__playwright__browser_file_upload` |
| 拖放文件到元素 | `mcp__playwright__browser_drop` |
| 执行 JS | `mcp__playwright__browser_evaluate` |
| 执行 Playwright 代码 | `mcp__playwright__browser_run_code_unsafe` |
| 网络请求列表 | `mcp__playwright__browser_network_requests` |
| 网络请求详情 | `mcp__playwright__browser_network_request` |
| 控制台日志 | `mcp__playwright__browser_console_messages` |
| 对话框处理 | `mcp__playwright__browser_handle_dialog` |
| 标签页管理 | `mcp__playwright__browser_tabs` |
| 等待文字/时间 | `mcp__playwright__browser_wait_for` |
| 调整窗口 | `mcp__playwright__browser_resize` |
| 关闭浏览器 | `mcp__playwright__browser_close` |

Vision 模式：可用自然语言描述元素位置，如"点击右上角的登录按钮"。

---

## 执行规则

- 每次操作完成后必须截图（`browser_take_screenshot`），不截图不算完成
- 元素找不到 → 先用 `browser_snapshot` 读 DOM 再说明
- 同一对话 session 全程保持浏览器，不重启
- 截图可存任意路径（已开启 unrestricted file access）

## 输出格式

```
[截图]
完成：[一句话]
```

ARGUMENTS: $ARGUMENTS
