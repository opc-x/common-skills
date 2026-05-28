# cursor — 派发 issue 给 Cursor 执行，结果汇总到 result issue

读取指定 issue（单个或按标签批量），用 Cursor agent 并行执行，所有结果汇总写入一个带 `result-${来源标签}` 标签的 issue。

## 使用方式

```
/cursor #N              # 执行单个 issue
/cursor #N #M #K        # 执行多个 issue（并行）
/cursor --label shape   # 执行所有带 shape 标签的 open issues
/cursor --label <任意>  # 执行所有带指定标签的 open issues
```

---

## 执行协议

### Step 1 — 收集 issue 列表

**单个/多个：** 直接使用指定编号，用 `mcp__github__get_issue` 读取每个 issue 的 title + body。

**按标签：**
```bash
REPO=$(git remote get-url origin | sed 's/.*github.com[:/]//' | sed 's/.git$//')
LABEL=<用户指定的标签>   # 同时作为 result issue 的标签前缀
RESULT_LABEL="result-${LABEL}"

gh issue list --repo "$REPO" --label "$LABEL" --state open \
  --json number,title --jq '.[] | [.number, .title] | @tsv'
```

得到 [(编号, 标题)] 列表，再用 `mcp__github__get_issue` 补全各 issue body。

**单个/多个编号（无 label）：** `RESULT_LABEL="result-cursor"`

---

### Step 2 — 并行执行

```bash
CURSOR=/Applications/Cursor.app/Contents/Resources/app/bin/cursor
REPO=$(git remote get-url origin | sed 's/.*github.com[:/]//' | sed 's/.git$//')
WORKSPACE=$(git rev-parse --show-toplevel)
```

并发上限 **16个**（16GB机器实测安全值）。超过16个时自动分批，每批之间 `wait` 等上一批全部完成再启下一批。

```bash
BATCH_SIZE=16
COUNT=0

for NUM in "${ISSUE_NUMS[@]}"; do
  cat > /tmp/cursor-prompt-$NUM.txt << EOF
完整执行这个 GitHub issue 的所有步骤：https://github.com/${REPO}/issues/${NUM}

执行规则：
1. 严格按照 issue body 的「执行步骤」逐步完成
2. 每步完成后对照 AC 自检
3. Out of scope 内容只记录不执行
4. 全部完成后，POST 执行报告到 issue comment：
   gh api repos/${REPO}/issues/${NUM}/comments \
     -X POST -f body="## 执行报告
状态：DONE 或 FAILED
步骤：[每步 ✅/❌ + 一句话说明]
AC：[每条 ✅/❌ + 实际结果]
DoD：tsc=[结果] lint=[结果]"
EOF

  $CURSOR agent -p --yolo \
    --model composer-2.5 \
    --workspace "$WORKSPACE" \
    "$(cat /tmp/cursor-prompt-$NUM.txt)" > /tmp/cursor-$NUM.log 2>&1 &

  sleep 1   # 错开启动，避免 cursor cli-config race condition
  COUNT=$((COUNT + 1))
  if [ $COUNT -eq $BATCH_SIZE ]; then
    wait      # 跑满一批，等全部完成再启下一批
    COUNT=0
  fi
done

wait
echo "all done"
```

---

### Step 3 — 读取结果，创建 result issue

每个 agent 完成后日志落在 `/tmp/cursor-$NUM.log`。判断状态：
- log 含 `DONE` → ✅
- log 含 `FAILED` 或为空 → ❌（提取 log 最后 5 行作为失败原因）

调用 `mcp__github__create_issue` 创建汇总 issue：

**标题：** `🤖 Cursor 执行结果 · YYYY-MM-DD · N个任务`

**body：**
```markdown
## 执行汇总

| Issue | 标题 | 状态 |
|-------|------|------|
| #N | [title] | ✅ / ❌ |
| #M | [title] | ✅ / ❌ |

## 失败项

- #N：[从 log 提取的失败原因，无则写"无"]

## 下一步

- 全部通过 → `/issule-verify #N #M`
- 有失败 → `/cursor #失败编号` 重跑
```

**labels：** `[RESULT_LABEL]`（即 `result-${来源标签}`，按标签执行时动态生成；单号执行时固定为 `result-cursor`）
