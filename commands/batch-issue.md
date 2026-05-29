# batch-issue — PRD → 批量 child-issues

读取 prd-issue 的 PRD，拆解为原子 child-issues，每个含 AC + DoD + T1/T2/T3。
**所有字段由 agent 自动生成，人只确认列表。**
等人确认后执行 /cursor。

## 使用方式

```
/batch-issue #N          # 从 prd-issue #N 批量生成 child-issues
/batch-issue #N <repo>   # 指定仓库
/batch-issue #N dry      # 只输出拆解草稿，不创建 issue
```

---

## 执行协议

### Step 1 — 读取 prd-issue PRD

调用 `mcp__github__get_issue` 读取 prd-issue body。

提取：
- 目标（一句话）
- 方案边界（做/不做）
- AC（验收标准）
- DoD（完成定义）

---

### Step 2 — 拆解为原子任务列表

拆解原则：
- 每个 child-issue 单一职责，独立可执行
- 不依赖其他 child-issue 才能开始（或明确标注依赖顺序）
- 粒度：一个 Cursor agent 在一个 worktree 能独立完成
- C4 L3 Component 粒度（有明确接口边界）

输出草稿（先给人看，确认后才创建）：

```
拟创建 N 个 child-issues（来自 prd-issue #M）：

#1  [emoji] [type]: [描述]
    AC: [功能验收条件]
    DoD: tsc + lint + [其他技术条件]

#2  [emoji] [type]: [描述]
    AC: [功能验收条件]
    DoD: tsc + lint + [其他技术条件]
...
```

---

### Step 3 — 人工确认检查点 🔔

```
🔔 拟创建以上 N 个 child-issues

确认创建 → 回复"确认"
需要调整 → 直接说修改意见
```

等待用户确认，收到确认后执行 Step 4。

---

### Step 4 — 批量创建 child-issues

为每个任务调用 `mcp__github__create_issue`：

```bash
TODAY=$(date +%Y-%m-%d)
MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  --jq ".[] | select(.title==\"$TODAY\") | .number")
[ -z "$MILESTONE_NUM" ] && MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  -X POST -f title="$TODAY" --jq ".number")
```

每个 child-issue body 格式（T1/T2/T3 全部 agent 自动推导）：

```markdown
## 目标

[一句话，极限精确]

prd-issue: #[N]

---

## 上下文（Cursor 必读，拿到即可跑）

> 技术栈：[从 CLAUDE.md 提取，如 Next.js 15 + Supabase + Hono :4000]
> 关键约束：[硬限制，如"不改 RLS"、"只动 backend/ 目录"、"不破坏现有接口"]
> 相关文件：[必须参考的路径+行号，如 lib/learn.ts:42]
> 依赖：[前置 issue #N 已完成后才能跑，无则写"无"]
> 不做：[明确排除项] — 原因：[为什么]

---

## AC（验收标准）

- [ ] [功能条件1]
- [ ] [功能条件2]

## DoD（完成定义）

- [ ] tsc --noEmit → 0 errors
- [ ] npm run lint → 0 warnings
- [ ] [其他技术条件]

---

## T1（机器验证）

- `[具体命令]` → 期望：`[精确值]`

## T2（AI推理验证）

- [判断标准] → 置信度 ≥ [X]%

## T3（人工验证）

- 第1步：[精确操作]
- 预期：[标准]

---

## 执行步骤

[具体步骤，含文件路径和操作位置]

## 边界

不做：[out of scope] — 原因：[为什么]

---
> 🤖 由 /batch-issue 生成 · [YYYY-MM-DD]
```

- labels: `["child-issue"]`
- milestone: 今天日期

---

### Step 5 — 输出汇总

在 prd-issue 写入 comment：
```markdown
## Child Issues 已创建

| # | Issue | 状态 |
|---|-------|------|
| 1 | #[N1] [title] | 待执行 |
| 2 | #[N2] [title] | 待执行 |

**并发分组（依赖列为空=可并发，有依赖=串行）：**

| 分组 | Issues | 派发方式 |
|------|--------|---------|
| 可并发 | #N1, #N2, #N3（无相互依赖） | `/cursor #N1 #N2 #N3` |
| 串行 | #N4 依赖 #N1 | 等 #N1 验收后再 `/cursor #N4` |

**下一步：**
- 并发跑：`/cursor #N1 #N2 #N3`
- 执行完验收：`/issule-verify #N1 #N2 #N3`

---
> 🤖 由 /batch-issue 生成 · [YYYY-MM-DD]
```

终端输出：
```
✅ 已创建 N 个 child-issues

#N1 [title]
#N2 [title]

🔔 确认开始执行？→ /cursor #N1 #N2 ...
```

---

## 自测验收

| 检查项 | 预期结果 |
|--------|---------|
| 先输出草稿 | 创建前展示完整列表，等人确认 |
| child-issue 结构 | 含 AC + DoD + T1/T2/T3 + prd-issue 引用 |
| T1 覆盖率 | 每条 AC 至少对应一条 T1 命令 |
| T3 节制 | 能写 T1 的不写 T3 |
| label | child-issue |
| milestone | 今天日期 |
| prd-issue comment | 写入 child 清单 |
| dry 模式 | 只输出草稿，不创建任何 issue |
