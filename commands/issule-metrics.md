# issule-metrics — 量化自验收指标生成

读取 issule，为执行 agent 生成量化自验收指标，POST 到 issule comment。
执行 agent（Cursor 等）拿到这些指标后，边执行边自检，向 100 分自优化。

**在 /issule-check 通过后、交给执行 agent 前运行。**

---

## 使用方式

```
/issule-metrics #N        # 为指定 issue 生成量化指标
/issule-metrics #N <repo> # 指定仓库
```

---

## 执行协议

### Step 1 — 读取 issule，理解「让什么变成真的」

读取 issue body + 所有 comments。

提取：
- 核心承诺：完成后什么状态变了
- 每个 Step 做了什么
- 已有的验收条件（body 和 comments 里的 checkbox）

---

### Step 2 — 推导量化指标

**从 issule 内容推导，不硬编码类型。**

推导原则：
```
① 核心承诺 → 最高优先级指标（直接验证「变成真的」那件事）
② 每个 Step → 对应的可运行检查
③ 数字化优先 → 能出数字的出数字
④ 数字化不了 → AI 推理，给置信度
⑤ AI 也不确定 → 标记为人工，给操作步骤
```

每条指标必须包含：
- 运行方式（SQL / bash / grep / AI推理）
- 期望值（精确的数字或判断标准）
- 对应的 Step 编号
- 失败时的修复方向

---

### Step 3 — 人工快速确认（5秒）

生成指标后，先给人过目：

```
📋 为 #N 生成了 X 条量化指标，准备 POST 到 issue：

Tier 1（自动跑）：
  [列出所有 SQL/bash 指标]

Tier 2（AI推理）：
  [列出所有 AI 判定指标]

Tier 3（需人工）：
  [列出需要人操作的指标]

回复「ok」POST 到 issue；要调整直接说。
```

---

### Step 4 — POST 到 Issue Comment

格式如下（执行 agent 直接读这个 comment 做自验收）：

```markdown
## 🎯 量化自验收指标（执行 agent 专用）

> 执行每个 Step 后运行对应指标，未达标继续优化，全部通过才算完成。
> 最终得分 = Tier1通过率 × 70% + Tier2置信度均值 × 30%
> 目标：≥ 90 分

---

### Tier 1 — 数字化指标（直接跑，出数字）

**Step N 完成后：**
- 指标：`[SQL 或 bash 命令]`
- 期望：`[精确期望值]`
- 失败时：[一句话修复方向]

**Step N 完成后：**
- 指标：`[命令]`
- 期望：`[期望值]`
- 失败时：[修复方向]

---

### Tier 2 — AI推理指标（模型自判，给置信度）

**全部步骤完成后：**
- 指标：[从 DB/文件取 N 条样本，判断 X]
- 判断标准：[具体标准，不模糊]
- 置信度 < 80% → 继续优化直到达标

---

### Tier 3 — 人工确认项（给步骤，不给判断题）

**交付前：**
- 第1步：[精确操作]
- 第2步：[看什么]
- 第3步：符合 → 标记通过；不符合 → 描述问题

---

### 总分计算

```
Tier1得分 = 通过条数 / 总条数 × 100
Tier2得分 = 各项置信度均值
最终得分 = Tier1得分 × 0.7 + Tier2得分 × 0.3

< 60 → 有明显问题，重新执行
60-80 → 基本可用，但有待改进项
80-90 → 良好，可交付
≥ 90 → 优秀，直接交付
```
```

POST 方式：
```ts
await fetch(`https://api.github.com/repos/HcPittBrad/talkflow/issues/${N}/comments`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.GITHUB_TOKEN}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ body: metricsComment }),
});
```

---

## 行为准则

| 应该做 | 不应该做 |
|--------|---------|
| 从 issule 内容推导，每个 issue 都不同 | 复制粘贴通用模板 |
| 核心承诺对应最高优先级指标 | 检查不重要的过程细节 |
| 期望值精确（数字，不是"越大越好"）| 写模糊的"应该有数据" |
| Tier3 给步骤，不给判断题 | 让 agent 或人"感觉"是否正确 |
| 总分计算透明，agent 知道怎么自优化 | 只说通过/不通过没有分数 |
