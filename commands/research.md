# research — 并行调研，产出 research-issue

输入问题域描述，并行多源调研，结果写入 GitHub research-issue comment，等人确认进入 /prd。

## 使用方式

```
/research "问题域描述"           # 创建 research-issue + 开始调研
/research "描述" <repo>          # 指定仓库
/research "描述" dry             # 只输出调研草稿，不创建 issue
```

---

## 执行协议

### Step 1 — 创建 research-issue

```bash
TODAY=$(date +%Y-%m-%d)
MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  --jq ".[] | select(.title==\"$TODAY\") | .number")
[ -z "$MILESTONE_NUM" ] && MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  -X POST -f title="$TODAY" --jq ".number")
```

调用 `mcp__github__create_issue`：
- title: `🔵 research: [问题域描述（≤20字）]`
- body: `## 问题域\n\n[描述]\n\n## 调研进行中...\n`
- labels: `["research-issue"]`
- milestone: 今天日期

输出：`🔵 research-issue #N 已创建，开始并行调研...`

---

### Step 2 — 并行调研（多源）

同时启动以下调研，结果合并后一次性 POST：

**源 A — /browser 公开信息**
- 搜索问题域相关技术文档、最佳实践、竞品方案
- 提取核心结论，不复制大段原文

**源 B — /gh 历史 issues/PR**
```bash
gh issue list --repo $REPO --state all --search "[关键词]" --json number,title,body
gh pr list --repo $REPO --state all --search "[关键词]" --json number,title,body
```
- 提取历史上相关的决策和结论

**源 C — 当前代码库上下文**
```bash
find . -type f -name "*.ts" -o -name "*.tsx" | head -20
grep -r "[关键词]" --include="*.ts" -l
```
- 提取相关现有实现

**源 D — 用户补充输入（如有）**
- 若用户在描述中包含额外信息，结构化提取

---

### Step 3 — POST 调研报告到 issue comment

```markdown
## 调研报告

**调研时间：** [timestamp]
**信息源：** /browser · GitHub历史 · 代码库

---

### 关键发现

[按重要性排列，每条一句话结论]

### 现有方案/竞品

[对比表格]

### 代码库现状

[相关现有实现，文件路径]

### 风险/注意点

[潜在问题]

---

**建议下一步：** `/prd #N` 基于以上调研生成 PRD
```

POST 命令：
```bash
gh api repos/$REPO/issues/$N/comments -X POST -f body="$REPORT"
```

---

### Step 4 — 人工确认检查点 🔔

输出：
```
🔔 research-issue #N 调研完成

建议：确认调研内容后执行 → /prd #N
若需补充调研：继续描述，我继续写入 comment
```

等待用户确认，不自动进入 /prd。

---

## 自测验收

| 检查项 | 预期结果 |
|--------|---------|
| research-issue 创建 | label=research-issue，milestone=今天 |
| 调研报告 comment | 包含关键发现、对比、代码库现状三节 |
| 🔔 确认点 | 明确告知下一步是 /prd #N |
| dry 模式 | 只输出草稿，不创建 issue |
