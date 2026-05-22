# supabase — Supabase 全能入口

自然语言操作 Supabase 的一切。说什么做什么。

## 使用方式

```
/supabase <任意描述>
```

示例：

```
/supabase 查一下所有表
/supabase 看看 profiles 表的结构
/supabase 查 user_progress 里最近10条数据
/supabase 给 review_schedule 加一个 last_reviewed_at 字段
/supabase 看看有没有慢查询或性能问题
/supabase 查一下今天的错误日志
/supabase 当前项目的 URL 和 anon key 是什么
/supabase 生成最新的 TypeScript 类型
```

---

## 执行协议

### Step 1 — 理解意图

从描述判断要做什么：

| 意图 | 使用的工具 |
|------|-----------|
| 看表结构 / 列出表 | `list_tables` |
| 查数据 / 统计 | `execute_sql` |
| 改表结构 / 加字段 | `apply_migration` |
| 看日志 / 报错 | `get_logs` |
| 性能问题 / 建议 | `get_advisors` |
| 项目信息 / URL / Key | `get_project_url` + `get_publishable_keys` |
| 生成 TS 类型 | `generate_typescript_types` |
| 查项目列表 | `list_projects` |
| 部署函数 | `deploy_edge_function` |

不确定用哪个 → 先 `list_tables` 摸清现状再决定。

### Step 2 — 执行

直接调对应工具。涉及写操作（migration / deploy）先输出将要执行的内容，确认后执行。

### Step 3 — 输出

结果用表格或 ASCII 图展示，不用代码块包普通文字。
有问题直接说问题，没问题说没问题。
