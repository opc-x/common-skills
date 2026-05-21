# gh — GitHub Issue & PR 管理

用 GitHub MCP 管理 HcPittBrad 下的所有仓库 issue 和 PR，无需打开浏览器。

## 使用方式

```
/gh                          # 列出 talkflow 所有 open issue（默认）
/gh <repo>                   # 列出指定仓库的 open issue
/gh <repo> <number>          # 查看某个 issue 的完整内容
/gh <repo> pr                # 列出该仓库的 open PR
/gh <repo> close <number>    # 关闭某个 issue（需确认）
/gh <repo> comment <number>  # 给某个 issue 加 comment
```

示例：
```
/gh                          # talkflow open issues
/gh talkflow 13              # 查看 issue #13 详情
/gh talkflow pr              # talkflow open PRs
/gh app-ai-input             # app-ai-input open issues
```

---

## 固定配置

- **owner**: `HcPittBrad`
- **默认 repo**: `talkflow`

---

## 执行流程

### 1. 列出 open issues（默认或指定 repo）

调用 `mcp__github__list_issues`：
- owner: `HcPittBrad`
- repo: 参数指定，默认 `talkflow`
- state: `open`
- per_page: 20

输出格式（简洁表格）：
```
# HcPittBrad/talkflow — open issues (N)

#13  feat: 复习复盘模块 — SRS 每日复习   [enhancement, design]  2026-05-19
#12  feat: 订阅计费体系 — 课程解锁        [enhancement]          2026-05-19
#10  feat: Onboarding 重做               [enhancement, design]  2026-05-18
```

### 2. 查看 issue 详情

调用 `mcp__github__get_issue`：
- owner: `HcPittBrad`
- repo: 参数指定
- issue_number: 参数指定

完整输出 title + body + labels + 状态，保留原始 markdown 格式方便阅读。

### 3. 列出 open PR

调用 `mcp__github__list_pull_requests`：
- owner: `HcPittBrad`
- repo: 参数指定
- state: `open`

### 4. 关闭 issue

1. 先调用 `mcp__github__get_issue` 确认标题
2. 向用户确认：「确认关闭 #N: <title>？」
3. 用户确认后调用 `mcp__github__update_issue`（state: closed）

### 5. 加 comment

调用 `mcp__github__add_issue_comment`：
- 先向用户确认 comment 内容
- 确认后提交

---

## 输出原则

- 列表视图：简洁，编号 + 标题 + labels + 日期，一行一条
- 详情视图：完整 body，不截断，保留 markdown
- 所有操作默认中文交互
