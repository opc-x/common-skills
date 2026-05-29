# prd — 调研结果 → PRD 文件 → prd-issue

读取 `.opc/tmp/[主题]/research.md`，生成 `.opc/[主题]-prd.md`，你 QC 后开 prd-issue，进入 /batch-issue。

## 使用方式

```
/prd --from .opc/tmp/[主题]/     # 从本地调研文件生成 PRD（主路径）
/prd #N                          # 从 research-issue #N 生成 PRD（兼容旧路径）
/prd --from .opc/tmp/[主题]/ dry # 只输出草稿，不写文件不开 issue
```

---

## 执行协议

### Step 1 — 读取调研内容

**主路径**（`--from` 模式）：
- 读取 `.opc/tmp/[主题]/research.md`
- 若目录下有多个 `*.md`，全部读取合并

**兼容路径**（`#N` 模式）：
- 调用 `mcp__github__get_issue` 读取 research-issue body + comments

提取：问题域 / 关键发现 / 风险注意点 / 建议方向

---

### Step 2 — 生成 PRD 文件

写入 `.opc/[主题]-prd.md`：

```markdown
# [主题] PRD

> 🤖 由 /prd 生成 · [YYYY-MM-DD]
> 来源：.opc/tmp/[主题]/research.md
> 下一步：QC 完成后 → 开 prd-issue → `/batch-issue #N`

---

## 问题定义

[现在什么不对/缺什么，一句话]

## 目标

完成后，有一件事从「不存在/不正确」变成「存在/正确」：
[极限精确，一句话]

## 方案边界

**做：**
- [具体做什么]

**不做：**
- [明确 out of scope] — 原因：[为什么]

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

## 上下文（Cursor 必读，拿到即可跑）

> 技术栈：[从 CLAUDE.md 提取，如 Next.js 15 + Supabase + Hono :4000]
> 关键约束：[硬限制，如"不改 RLS"、"只动 backend/ 目录"]
> 相关文件：[必须参考的路径+行号]
> 不做：[排除项] — 原因：[为什么]

---

## 🚦 待拍板

| # | 决策 | 推荐 | 原因 |
|---|------|------|------|
| D1 | [决策项] | [推荐方案] | [原因] |
```

---

### Step 3 — 人工确认检查点 🔔

```
🔔 PRD 草稿已写入 → .opc/[主题]-prd.md

QC 完后执行：开 prd-issue
需要修改：直接说，我更新文件
```

等你确认，**不自动开 issue**。

---

### Step 4 — 开 prd-issue（你确认后）

```bash
TODAY=$(date +%Y-%m-%d)
MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  --jq ".[] | select(.title==\"$TODAY\") | .number")
[ -z "$MILESTONE_NUM" ] && MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  -X POST -f title="$TODAY" --jq ".number")
```

调用 `mcp__github__create_issue`：
- title: `🟡 prd: [目标描述（≤20字）]`
- body: `.opc/[主题]-prd.md` 全文
- labels: `["prd-issue"]`
- milestone: 今天日期

输出：
```
✅ prd-issue #N 已创建

下一步：/batch-issue #N
→ 拆解为 child-issues（无依赖可并发，有依赖标注串行顺序）

---
> 🤖 由 /prd 生成 · [YYYY-MM-DD]
```

---

## 自测验收

| 检查项 | 预期结果 |
|--------|---------|
| PRD 文件位置 | `.opc/[主题]-prd.md` |
| 文件结构 | 含问题定义/目标/边界/AC/DoD/上下文/待拍板 |
| 署名 | 文件头部含 `🤖 由 /prd 生成` |
| 🔔 确认点 | 你确认后才开 prd-issue |
| prd-issue label | `prd-issue`（不再用 master-issue） |
| dry 模式 | 只输出草稿到终端，不写文件不开 issue |
