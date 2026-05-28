# issue — 单个 child-issue 创建

基于 master-issue 上下文创建单个 child-issue，结构为 AC + DoD + T1/T2/T3。
**所有字段由 agent 自动生成，人只确认标题。**
solo 场景（无 master-issue）也可用。

## 使用方式

```
/issue "描述" #master-N     # 基于 master-issue 创建 child
/issue "描述"               # solo 场景，不挂载 master
/issue "描述" <repo>        # 指定仓库
/issue "描述" dry           # 只输出草稿
```

---

## 执行协议

### Step 1 — 读取上下文

有 master-issue 编号时：
- 调用 `mcp__github__get_issue` 读取 master body
- 继承：目标边界、DoD 基础要求、milestone

无 master-issue 时（solo）：
- label 使用 `solo`
- 独立设定边界和 DoD

---

### Step 2 — 锁标题

格式：`{emoji} {类型}: {业务描述（≤20字）}`

```
emoji: 🔴 主流程阻断 · 🟡 核心缺失 · 🟢 体验提升 · 🔵 维护/重构 · 🟣 探索
```

输出：
```
🎯 标题提案：[emoji] [type]: [描述]
确认即创建；要改直接说。
```

等用户确认标题。

---

### Step 2.5 — 人工确认标题 🔔

```
🎯 标题提案：[emoji] [type]: [描述]
确认即写正文；要改直接说。
```

等用户确认后执行 Step 3。

### Step 3 — 生成 issue body

**Agent 基于 AC 自动推导 T1/T2/T3，不等人写。**

生成规则：
- T1：每条 AC → 对应一条可执行命令 + 期望值（能机器跑的都写 T1）
- T2：无法命令行验证的功能语义 → AI推理，给置信度阈值
- T3：真正需要人眼/人手操作才能验证的 → 给精确步骤（能 T1/T2 的不写 T3）

```markdown
## 目标

[一句话：让什么从「不存在/不正确」变成「存在/正确」]

[有 master-issue 时] master-issue: #N

---

## AC（验收标准）

> 功能层面，完成长什么样

- [ ] [可验证的功能条件]
- [ ] [可验证的功能条件]

## DoD（完成定义）

> 技术层面，必须满足

- [ ] tsc --noEmit → 0 errors
- [ ] npm run lint → 0 warnings
- [ ] [其他技术条件]

---

## T1（机器验证）

> Agent 自动执行，命令 + 期望值

- `[具体命令]` → 期望：`[精确值/0 errors/status 200 等]`
- `[具体命令]` → 期望：`[精确值]`

## T2（AI推理验证）

> Agent 自判，抽样 + 置信度

- [判断标准，具体不模糊] → 置信度 ≥ [X]%

## T3（人工验证）

> 仅 T1/T2 无法覆盖时才写，给精确步骤不给判断题

- 第1步：[精确操作]
- 第2步：[看什么/找什么]
- 预期：[符合则通过]

---

## 执行步骤

> 每步含文件路径 + 操作位置

**Step 1：**
**Step 2：**

## 边界

不做：[out of scope] — 原因：[为什么]
```

---

### Step 4 — 创建 issue

```bash
TODAY=$(date +%Y-%m-%d)
MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  --jq ".[] | select(.title==\"$TODAY\") | .number")
[ -z "$MILESTONE_NUM" ] && MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  -X POST -f title="$TODAY" --jq ".number")
```

调用 `mcp__github__create_issue`：
- title: Step 2 锁定标题
- body: Step 3 内容
- labels: `["child-issue"]`（solo 场景用 `["solo"]`）
- milestone: 今天日期

输出：
```
✅ Issue #N 已创建

执行入口：
──────────────────────────────
#N [issue 标题]
https://github.com/[owner]/[repo]/issues/N
──────────────────────────────
→ /cursor #N
```

---

## 自测验收

| 检查项 | 预期结果 |
|--------|---------|
| 有 master 时 | 继承边界 + DoD，body 含 master 引用 |
| 无 master（solo）| label=solo，独立边界 |
| AC + DoD + T1/T2/T3 | 必须存在，全部 agent 自动生成 |
| T1 覆盖率 | 每条 AC 至少对应一条 T1 命令 |
| T3 节制 | 能写 T1 的不写 T3 |
| 标题先锁定 | 确认后才生成 body |
| dry 模式 | 只输出草稿，不创建 |
