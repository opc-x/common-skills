# issule-check — Issule 质检门禁

读取 GitHub Issue 的 body + 所有 comment，对照质检清单逐项评分，
输出"可以交 Cursor 跑"或"还有漏洞需要补"的明确结论。

**在 issule 创建后、交给 Cursor 执行前运行。**

---

## 使用方式

```
/check-issule #N        # 质检指定 issue（默认 talkflow repo）
/check-issule #N <repo> # 指定仓库
```

---

## 执行协议

### Step 1 — 读取 Issue 全文

调用 `mcp__github__get_issue` 读取 issue body。
调用 `mcp__github__get_pull_request_comments` 或直接用
`gh api repos/HcPittBrad/<repo>/issues/<N>/comments` 读取所有 comment。

将 body + 所有 comment 合并为完整上下文。

---

### Step 2 — 逐维度质检

对每个维度：✅ 通过 / ⚠️ 警告（不阻断但需注意）/ ❌ 不通过（阻断）

---

#### 维度 1：人看区完整性
```
检查项：
□ 有「问题」字段，一句话，无技术词汇
□ 有「解决」字段，列出 ① ② ③ 子问题
□ 有「步骤」字段，业务动词，不含文件名/函数名
□ 有「结果」字段，格式为「触发 → 用户看到的变化」
□ 字数克制，5秒能读完
```

#### 维度 2：设计区准确性（有设计内容时检查）
```
检查项：
□ 表结构/字段名是否与 DB 实际一致（跑 SQL 核对）
□ 新增字段是否有类型说明
□ 用了图（ASCII tree/表格/流程图）而非纯文字
□ 字段命名符合项目约定（snake_case，无缩写歧义）
```

跑核对 SQL（以 talkflow 为例）：
```sql
-- 验证 issue 里提到的表和字段是否真实存在
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name IN (/* issue 里提到的表 */)
ORDER BY table_name, ordinal_position;
```

#### 维度 3：文件路径真实性
```
检查项：
□ Agent 区每个文件路径，用 Bash ls/find 确认实际存在
□ 行号引用是否还准确（文件可能改过）
□ 新建文件的目录路径是否存在
```

用 Bash 验证：
```bash
# 逐个验证 issue 里提到的文件
ls <path> 2>/dev/null || echo "❌ 不存在: <path>"
```

#### 维度 4：代码影响范围
```
检查项：
□ 涉及 schema 变更时，是否用 grep 找出所有查了该表的文件
□ 找出的影响文件是否都列在 issue 里
□ TypeScript 类型文件是否有 gen types 步骤
```

用 Bash 核对：
```bash
# 检查 issue 里每个被改动的表，确认影响范围
grep -rn "\"<table_name>\"" app/ components/ lib/ backend/src/ \
  2>/dev/null | grep -v node_modules | grep "\.from\|\.select"
```

#### 维度 5：执行报告机制
```
检查项：
□ Agent 区是否有「执行后 POST 报告到 Issue comment」的逻辑
□ 报告格式包含：状态/做了什么/验收结果/需要人工确认
□ 报告 POST 使用 GITHUB_TOKEN（不是 MCP，是直接 fetch）
```

#### 维度 6：验收标准可量化
```
检查项：
□ 每条验收是「操作 → 预期结果」格式
□ 不含「看起来正常」「功能可用」等主观描述
□ 有 tsc --noEmit 零错误检查
□ 有幂等性验证（重复执行不报错/不重复写入）
□ 边界场景有覆盖（空数据、已有数据、异常输入）
```

#### 维度 7：依赖关系清晰
```
检查项：
□ 上游依赖 issue 明确列出（如"依赖 #19 完成"）
□ 执行顺序无歧义
□ 没有循环依赖
```

#### 维度 8：范围边界干净
```
检查项：
□ 「不要碰」区间列出了 out-of-scope 项及原因
□ 没有把两个独立功能塞进一个 issue
□ 单个 issue 的 Step 数量 ≤ 10（否则建议拆分）
```

---

### Step 3 — 生成质检报告

输出格式：

```
## 质检结果：#N <issue 标题>

总结：✅ 可以交 Cursor / ❌ 有 N 项阻断问题需先修复

| 维度 | 结果 | 问题描述 |
|------|------|---------|
| 人看区完整性   | ✅/⚠️/❌ | ... |
| 设计区准确性   | ✅/⚠️/❌ | ... |
| 文件路径真实性 | ✅/⚠️/❌ | ... |
| 代码影响范围   | ✅/⚠️/❌ | ... |
| 执行报告机制   | ✅/⚠️/❌ | ... |
| 验收可量化     | ✅/⚠️/❌ | ... |
| 依赖关系       | ✅/⚠️/❌ | ... |
| 范围边界       | ✅/⚠️/❌ | ... |

❌ 阻断问题（必须修复后才能交 Cursor）：
1. [具体问题] → 建议：/issule comment #N [补充描述]

⚠️ 警告（不阻断，建议关注）：
1. [具体问题]

✅ 全部通过 → 可以把 issue URL 交给 Cursor 执行
```

---

### Step 4 — 判定结论

**全部 ✅** → 输出：
```
✅ #N 质检通过，可以交 Cursor 执行
链接：https://github.com/HcPittBrad/talkflow/issues/N
```

**有 ❌** → 输出：
```
❌ #N 有 [N] 项阻断问题，修复后重新运行 /check-issule #N
```

**只有 ⚠️** → 输出：
```
⚠️ #N 有警告但可以执行，建议 Cursor 执行前注意以下项：[列出]
✅ 可以交 Cursor 执行
链接：https://github.com/HcPittBrad/talkflow/issues/N
```

---

## 质检行为准则

| 应该做 | 不应该做 |
|--------|---------|
| 实际跑 SQL 验证字段存在 | 只看 issue 文字就下结论 |
| 实际跑 Bash 验证文件路径 | 假设文件路径正确 |
| 给出具体修复建议 | 只说"有问题"不说怎么改 |
| 发现范围过大时建议拆分 | 默认接受任何规模的 issue |
| 逐维度独立评分 | 整体模糊打分 |
