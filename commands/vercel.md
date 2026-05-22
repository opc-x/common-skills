# vercel — Vercel 全能入口

自然语言操作 Vercel 的一切。说什么做什么。

## 使用方式

```
/vercel <任意描述>
```

示例：

```
/vercel 部署到预览环境
/vercel 部署到生产
/vercel 看看最近的部署状态
/vercel 列出所有环境变量
/vercel 把 NEXT_PUBLIC_API_URL 加到生产环境
/vercel 把线上环境变量同步到本地 .env
/vercel 看看最近的报错日志
/vercel 当前项目信息
/vercel 回滚到上一个部署
```

---

## 执行协议

### Step 1 — 理解意图

| 意图 | 执行方式 |
|------|---------|
| 部署预览 | `vercel deploy` via Bash |
| 部署生产 | `vercel deploy --prod` via Bash |
| 查部署状态 | `vercel ls` / `vercel inspect` via Bash |
| 列出环境变量 | `vercel env ls` via Bash |
| 添加环境变量 | `vercel env add` via Bash |
| 同步到本地 | `vercel env pull` via Bash |
| 查日志 | `vercel logs` via Bash |
| 项目信息 | `vercel project ls` via Bash |
| 回滚 | `vercel rollback` via Bash |
| 域名管理 | `vercel domains` via Bash |

### Step 2 — 执行

直接跑 `vercel` CLI 命令。涉及生产部署或删除操作，先说明将要做什么，等确认后执行。

### Step 3 — 输出

结果直接展示，重点信息用表格。
部署成功给 URL，失败给错误原因。
