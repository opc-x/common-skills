# research — 并行调研，产出 research.md

输入问题域描述，并行多源调研，结果写入 `.opc/tmp/[主题]/research.md`，你 QC 后可选开 research-issue。

## 使用方式

```
/research "问题域描述"           # 调研，结果写本地文件
/research "描述" --issue         # 调研完后额外开 research-issue
/research "描述" <repo>          # 指定仓库（配合 --issue 用）
/research "描述" dry             # 只输出草稿到终端，不写文件
```

---

## 执行协议

### Step 1 — 初始化工作目录

根据问题域描述提取主题名（kebab-case，≤4个词），创建：

```
.opc/tmp/[主题]/research.md
```

若 `.opc/tmp/[主题]/` 已存在（上次调研残留），继续追加，不覆盖。

---

### Step 2 — 并行调研（多源）

同时启动以下调研，结果合并后写入文件：

**源 A — 公开信息**
- 搜索问题域相关技术文档、最佳实践、竞品方案
- 提取核心结论，不复制大段原文

**源 B — GitHub 历史 issues/PR**
```bash
gh issue list --repo $REPO --state all --search "[关键词]" --json number,title,body
gh pr list --repo $REPO --state all --search "[关键词]" --json number,title,body
```
- 提取历史上相关的决策和结论

**源 C — 当前代码库**
```bash
grep -r "[关键词]" --include="*.ts" --include="*.tsx" -l
```
- 提取相关现有实现，记录文件路径+行号

**源 D — 用户补充输入（如有）**
- 结构化提取用户描述中的额外信息

---

### Step 3 — 写入 research.md

```markdown
# [主题] 调研报告

> 🤖 由 /research 生成 · [YYYY-MM-DD]
> 下一步：QC 完成后 → `/prd --from .opc/tmp/[主题]/`

---

## 问题域

[一句话描述要解决的问题]

---

## 关键发现

[按重要性排列，每条一句话结论]

---

## 现有方案/竞品对比

| 方案 | 优点 | 缺点 | 是否适用 |
|------|------|------|---------|
| [方案A] | ... | ... | ✅/❌ |

---

## 代码库现状

| 文件 | 行号 | 说明 |
|------|------|------|
| [path/file.ts] | L42 | [现有实现描述] |

---

## 风险/注意点

- [风险1]
- [风险2]

---

## 建议方向

[基于调研给出 1-2 个可行方向，供 /prd 决策用]
```

---

### Step 4 — 人工确认检查点 🔔

输出：
```
🔔 调研完成 → .opc/tmp/[主题]/research.md

QC 完后执行：/prd --from .opc/tmp/[主题]/
需要补充调研：继续描述，我继续追加到文件
需要开 issue 存档：/research "描述" --issue
```

等待用户确认，不自动进入 /prd。

---

### Step 5 — 可选：开 research-issue（仅 --issue 时）

用户明确要求时才执行：

调用 `mcp__github__create_issue`：
- title: `🔵 research: [主题（≤20字）]`
- body: research.md 全文
- labels: `["research-issue"]`
- milestone: 今天日期

```
✅ research-issue #N 已创建
→ 文件已存档到 GitHub，本地 .opc/tmp/[主题]/research.md 可保留或删除
```

---

## 自测验收

| 检查项 | 预期结果 |
|--------|---------|
| 文件位置 | `.opc/tmp/[主题]/research.md` |
| 四源调研 | 关键发现/对比/代码库现状/风险四节齐全 |
| 署名 | 文件头部含 `🤖 由 /research 生成` |
| 🔔 确认点 | 告知下一步是 `/prd --from` |
| --issue 模式 | 才开 research-issue，默认不开 |
| dry 模式 | 只输出到终端，不写文件不开 issue |
