# preview — 浏览器预览 & 页面审查

用 Playwright MCP 打开页面、截图、读 DOM，不用离开 Claude Code。

## 使用方式

```
/preview                        # 截图当前项目默认端口（自动检测 3000/3001/8080）
/preview <url>                  # 截图指定 URL
/preview <url> check            # 截图 + 分析页面问题（布局/错误/空状态）
/preview <url> <selector>       # 截图指定元素
/preview <url> interact <desc>  # 描述交互操作（点击/填表/滚动）
```

示例：
```
/preview                                    # 截图 localhost:3000
/preview localhost:3000/protected/review    # 截图 review 页面
/preview localhost:3000 check               # 截图 + 审查页面问题
/preview localhost:3000 .hero-section       # 截图 hero 区域
```

---

## 执行流程

### 1. 截图（默认）

1. 用 `playwright_navigate` 打开 URL
2. 等待页面加载（`playwright_wait_for_load_state`）
3. 用 `playwright_screenshot` 截全屏或指定元素
4. 输出截图 + 简短描述页面现状

### 2. check 模式

截图后额外分析：
- 有没有明显报错（红色文字、error boundary）
- 是否有空状态/loading 卡住
- 布局是否明显异常（内容溢出、元素重叠）
- 控制台报错（`playwright_evaluate` 读 console.error）

### 3. interact 模式

按描述执行交互后截图：
- 点击：`playwright_click`
- 填写表单：`playwright_fill`
- 滚动：`playwright_evaluate` scroll
- 交互后截图对比前后状态

---

## 端口自动检测

无参数时，依次尝试：
```bash
lsof -i :3000 | grep LISTEN
lsof -i :3001 | grep LISTEN
lsof -i :8080 | grep LISTEN
```
找到第一个在监听的端口，打开 `http://localhost:<port>`。

---

## 输出原则

- 截图直接内联展示
- 问题用中文列出，每条一行
- 不啰嗦，有问题说问题，没问题说没问题
