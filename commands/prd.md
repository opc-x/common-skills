# prd — 调研报告 → PRD master-issue

读取 research-issue(s) 的调研内容，综合生成 PRD，创建 master-issue。等人确认后进入 /batch-issue。

## 使用方式

```
/prd #N               # 从 research-issue #N 生成 PRD
/prd #N #M            # 合并多个 research-issue 生成一份 PRD
/prd #N <repo>        # 指定仓库
/prd #N dry           # 只输出 PRD 草稿，不创建 issue
```

---

## 执行协议

### Step 1 — 读取 research-issue(s)

调用 `mcp__github__get_issue` 读取每个 research-issue 的 body + comments。

提取：
- 问题域描述
- 关键发现
- 风险/注意点

---

### Step 2 — 生成 PRD

综合所有调研内容，生成结构化 PRD：

```markdown
## PRD

**版本：** v1.0  
**状态：** 待确认  
**来源：** research-issue #N [, #M]

---

### 问题定义

[现在什么不对/缺什么，一句话]

### 目标

完成后，有一件事从「不存在/不正确」变成「存在/正确」：
[极限精确，一句话]

### 方案边界

**做：**
- [具体做什么]

**不做：**
- [明确 out of scope，原因]

### 验收标准（AC）

> 功能层面，完成长什么样（用户/产品视角）

- AC1: [可验证的功能条件]
- AC2: [可验证的功能条件]

### 完成定义（DoD）

> 技术层面，必须满足的清单

- [ ] tsc --noEmit → 0 errors
- [ ] npm run lint → 0 warnings
- [ ] 相关测试通过
- [ ] [其他技术条件]

---

**下一步：** `/batch-issue #master-N` 拆解为 child-issues
```

---

### Step 3 — 创建 master-issue

```bash
TODAY=$(date +%Y-%m-%d)
MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  --jq ".[] | select(.title==\"$TODAY\") | .number")
[ -z "$MILESTONE_NUM" ] && MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  -X POST -f title="$TODAY" --jq ".number")
```

调用 `mcp__github__create_issue`：
- title: `🟡 [type]: [目标描述（≤20字）]`
- body: Step 2 生成的 PRD
- labels: `["master-issue"]`
- milestone: 今天日期

关闭来源 research-issue(s)（标注关联）：
```bash
gh api repos/$REPO/issues/$N/comments -X POST \
  -f body="已综合进 master-issue #[master-N]，关闭此调研。"
gh api repos/$REPO/issues/$N -X PATCH -f state=closed
```

---

### Step 4 — 人工确认检查点 🔔

输出：
```
🔔 master-issue #N 已创建（PRD）

请审核 PRD 内容，确认后执行 → /batch-issue #N
若需修改：直接说，我更新 issue body
```

等待用户确认，不自动进入 /batch-issue。

---

## 自测验收

| 检查项 | 预期结果 |
|--------|---------|
| master-issue 创建 | label=master-issue，body 含 AC + DoD 两节 |
| research-issue 关闭 | 写入关联 comment 后 state=closed |
| 🔔 确认点 | 明确告知下一步是 /batch-issue #N |
| dry 模式 | 只输出草稿，不创建/关闭任何 issue |
