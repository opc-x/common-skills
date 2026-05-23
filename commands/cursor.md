# cursor — 全自动派发 issule 给 Cursor 并行执行

读取指定 issule，用 cursor agent CLI 并行启动，每个 agent 独立 worktree 执行，结果自动 POST 回 issule comment（含模型标注）。

## 使用方式

```
/cursor #N                        # 派发单个 issue（默认 composer-2.5）
/cursor #N #M #K                  # 批量派发，N个 agent 并行
/cursor #N <repo>                 # 指定仓库（默认读当前 git remote）
/cursor #N --model high           # 强制高配模型（claude-opus-4-7）
/cursor #N --model composer       # 强制 composer-2.5（默认）
/cursor --label child-issue       # 派发当前 repo 所有 open child-issues
/cursor --master #N               # 派发 master-issue #N 的所有 open child-issues
```

`--label child-issue` 模式：
```bash
gh issue list --repo $REPO --label child-issue --state open \
  --json number --jq '.[].number'
```
取出所有编号后并行派发，等同于 `/cursor #N1 #N2 ...`

---

## 模型路由策略

**默认：composer-2.5**（CursorBench #3，$0.55/task，性价比最高）

```
读取 issue 标题 + body
        │
        ▼
包含以下关键词？
重构/refactor/migration/数据库迁移/架构/破坏性/不可逆
        │
   YES  │  NO
        │   └──→ composer-2.5（默认，$0.55）
        ▼
  claude-opus-4-7（$11.02，最准）
  派发前输出提示：
  ⚠️ 检测到高风险任务，已切换 Opus，预计成本 $X
```

| 模型 | CursorBench | 成本/task | 适用场景 |
|------|-------------|-----------|---------|
| `composer-2.5` | 63.2% | $0.55 | 日常功能·bug修复·小重构 |
| `claude-opus-4-7` | 64.8% | $11.02 | 大型重构·破坏性迁移·架构变更 |

手动覆盖用 `--model high`（Opus）或 `--model composer`（默认）。

---

## 执行协议

### Step 0 — Auth 预检（必须先通过）

```bash
CURSOR=/Applications/Cursor.app/Contents/Resources/app/bin/cursor
$CURSOR agent -p --yolo --workspace /tmp "echo auth_ok" 2>&1
```

若输出含 "Authentication required"，立即停止并输出：

```
❌ Cursor Agent 未登录，无法派发任务。

解决方案（选一）：
  方式1：! cursor agent login        （交互登录）
  方式2：! export CURSOR_API_KEY=xxx  （API Key，永久有效建议加到 ~/.zshrc）

登录后重新执行 /cursor #N #M ...
```

**不继续执行 Step 1。**

---

### Step 1 — 读取所有指定 issule

用 `mcp__github__get_issue` 逐个读取，提取：
- issue URL
- issue 标题
- body 中的「执行步骤」段落

---

### Step 2 — 并行启动 Cursor Agent

为每个 issule 后台启动一个 agent（`&`）：

```bash
CURSOR=/Applications/Cursor.app/Contents/Resources/app/bin/cursor
REPO="HcPittBrad/talkflow"          # 从 git remote 读取
WORKSPACE="/path/to/project"        # 当前项目路径
MODEL="composer-2.5"                # 默认：性价比最高（见模型路由策略）

$CURSOR agent -p --yolo \
  --worktree \
  --workspace "$WORKSPACE" \
  --model "$MODEL" \
  "读取并完整执行这个 GitHub issue 的所有步骤：https://github.com/$REPO/issues/N

执行规则：
1. 完整读取 issue body，严格按照「执行步骤」逐步完成
2. 每步完成后对照验收标准自检
3. Out of scope 内容只记录不执行
4. 全部步骤完成后，必须用以下命令将报告 POST 到 issue comment：

gh api repos/$REPO/issues/N/comments -X POST -f body='## 执行报告

🤖 **执行模型：** Cursor Agent ($MODEL)
📋 **派发模型：** Claude (master)

**状态：** ✅ 完成 / ❌ 失败

**执行步骤完成情况：**
[每步 ✅/❌ + 一句话说明]

**AC 验收（功能层面）：**
[每条 AC: ✅通过 / ❌未通过 + 实际结果]

**DoD 验收（技术层面）：**
- tsc --noEmit → [结果]
- npm run lint → [结果]
- [其他 DoD 条件 → 结果]

**T1 机器验证（命令 + 实际值 vs 期望值）：**
[每条 T1: ✅ 实际=[X] 期望=[Y] / ❌ 实际=[X] 期望=[Y]]

**T2 AI推理验证：**
[每条: 置信度 XX% → 通过/不通过 + 理由]

**T3 人工验证（需人操作）：**
[无 / 或精确步骤 + 预期结果]'

POST 成功才算交付完成。" > /tmp/cursor-issue-N.log 2>&1 &
```

**关键：**
- `--worktree` 不传自定义名（让 Cursor 自动生成，避免斜杠报错）
- 日志写到 `/tmp/cursor-issue-N.log`，N 为 issue 编号

所有 agent 启动后：
```bash
wait
echo "所有 agent 完成"
```

---

### Step 3 — 汇总输出

```
🚀 已并行启动 N 个 Cursor Agent (claude-sonnet-4-5)

Agent 1 → #N [title] — 独立 worktree 执行中
Agent 2 → #M [title] — 独立 worktree 执行中
Agent 3 → #K [title] — 独立 worktree 执行中

每个 agent 完成后自动 POST 执行报告到对应 issue。
完成后运行：/issule-verify #N #M #K
```

---

## 模型标注规范

每个 issue comment 必须包含这两行（Claude 直接执行时也一样）：

```
🤖 执行模型：Cursor Agent (claude-sonnet-4-5)
📋 派发模型：Claude (master)
```

若 Claude 自己代劳执行（Cursor 不可用时）：
```
🤖 执行模型：Claude (claude-sonnet-4-6) [fallback]
📋 派发模型：Claude (master)
```

---

## 完整工作流

```
Claude (master)                    Cursor (slave agents)
──────────────────                 ─────────────────────
/browser → 巡检所有页面
    ↓
/issule-goal → 开单（含T1/T2/T3）
    ↓
/cursor #N #M #K ──────────────→  agent-1: #N 独立 worktree
                 ──────────────→  agent-2: #M 独立 worktree
                 ──────────────→  agent-3: #K 独立 worktree
                                     ↓ (POST report 含模型标注)
/issule-verify #N #M #K
    ↓
YES → gh issue close #N #M #K
NO  → /cursor 重跑失败项
```

---

## 前提条件（一次性设置）

```bash
# 方式1：交互登录（浏览器授权）
! cursor agent login

# 方式2：API Key（更稳定，推荐）
! echo 'export CURSOR_API_KEY=your_key' >> ~/.zshrc && source ~/.zshrc
```

API Key 在 Cursor 设置页面 → Account → API Keys 获取。

---

## 参数说明

| 参数 | 作用 |
|------|------|
| `-p` / `--print` | 非交互模式，脚本可调用 |
| `--yolo` | 自动批准所有操作 |
| `--worktree` | 独立 git worktree（不传名，自动生成） |
| `--workspace` | 项目绝对路径 |
| `--model` | 执行模型，默认 claude-sonnet-4-5 |
| `&` | 后台并行，不阻塞主进程 |
