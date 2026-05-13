查询或操作 MemPalace 记忆宫殿。用于搜索历史会话、存储记忆、知识图谱查询等操作。

## 环境说明

MCP server 已通过 `/home/devtrump/.local/bin/mempalace-mcp` 连接（`claude mcp list` 可验证）。

**调用方式**：优先尝试直接 MCP tool call（`mempalace_*`）；如果 MCP tool 不可调用，用以下 bash 封装：

```bash
_mp() {
  local tool=$1; local args=${2:-{}}
  echo "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"tools/call\",\"params\":{\"name\":\"$tool\",\"arguments\":$args}}" \
    | /home/devtrump/.local/bin/mempalace-mcp \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['result']['content'][0]['text'])"
}
```

用法示例：`_mp mempalace_search '{"query":"topicinfo 域名"}'`

---

## Palace 结构（当前）

| Wing | 内容 | Drawers |
|------|------|---------|
| claude-code-wsl | 当前 WSL Claude Code 会话记忆 | ~1542 |
| claude-code-windows | Windows Claude Code 会话记忆 | ~6641 |
| hermes | Hermes/云端会话记忆 | ~1964 |
| codex-windows | Codex 会话记忆 | ~894 |
| mempalace | mempalace 系统自身知识 | ~148 |
| openclaw-wsl/windows | OpenClaw 会话记忆 | ~241 |

**Room 命名规则**：`general`（通用）、`technical`（技术细节）、`architecture`（架构决策）、`problems`（踩坑）、`planning`（计划）、`decisions`（决策）。

---

## 工具速查（30 个）

### 读 Palace

| 工具 | 用途 | 关键参数 |
|------|------|---------|
| `mempalace_status` | Palace 概览：drawers 总数、wings、rooms | 无 |
| `mempalace_list_wings` | 列出所有 wing 及 drawer 数 | 无 |
| `mempalace_list_rooms` | 列出 wing 下的 rooms | `wing?` |
| `mempalace_get_taxonomy` | 完整 wing→room→drawer 树 | 无 |
| `mempalace_search` | **语义搜索**（最常用） | `query`, `wing?`, `room?`, `limit?`, `max_distance?` |
| `mempalace_check_duplicate` | 存前查重 | `content`, `threshold?` |
| `mempalace_get_aaak_spec` | 获取 AAAK 压缩格式规范 | 无 |
| `mempalace_list_drawers` | 分页列出 drawers | `wing?`, `room?`, `limit?`, `offset?` |
| `mempalace_get_drawer` | 按 ID 取单个 drawer 全文 | `drawer_id` |

### 写 Palace

| 工具 | 用途 | 关键参数 |
|------|------|---------|
| `mempalace_add_drawer` | 存入新 drawer | `wing`, `room`, `content`, `added_by?` |
| `mempalace_update_drawer` | 更新已有 drawer 内容/位置 | `drawer_id`, `content?`, `wing?`, `room?` |
| `mempalace_delete_drawer` | 删除 drawer（不可逆） | `drawer_id` |
| `mempalace_sync` | 清理已删除/移动文件的 drawers | `project_dir?`, `wing?`, `apply?` |

### 知识图谱（KG）

| 工具 | 用途 | 关键参数 |
|------|------|---------|
| `mempalace_kg_query` | 查询实体关系 | `entity`, `as_of?`, `direction?` |
| `mempalace_kg_add` | 添加三元组事实 | `subject`, `predicate`, `object`, `valid_from?`, `valid_to?` |
| `mempalace_kg_invalidate` | 标记事实失效 | `subject`, `predicate`, `object`, `ended?` |
| `mempalace_kg_timeline` | 实体时间线 | `entity?` |
| `mempalace_kg_stats` | KG 概览：实体数、三元组数 | 无 |

### 图导航（Tunnels）

| 工具 | 用途 | 关键参数 |
|------|------|---------|
| `mempalace_traverse` | 从某 room 出发走知识图 | `start_room`, `max_hops?` |
| `mempalace_find_tunnels` | 找跨 wing 的连接点 | `wing_a?`, `wing_b?` |
| `mempalace_graph_stats` | 图概览 | 无 |
| `mempalace_create_tunnel` | 手动建 cross-wing 隧道 | `source_wing`, `source_room`, `target_wing`, `target_room`, `label?` |
| `mempalace_list_tunnels` | 列出所有隧道 | `wing?` |
| `mempalace_delete_tunnel` | 删除隧道 | `tunnel_id` |
| `mempalace_follow_tunnels` | 从某 room 跟着隧道看连接 | `wing`, `room` |

### Agent Diary

| 工具 | 用途 | 关键参数 |
|------|------|---------|
| `mempalace_diary_write` | 写 agent 日记（AAAK 格式） | `agent_name`, `entry`, `topic?`, `wing?` |
| `mempalace_diary_read` | 读最近日记 | `agent_name`, `last_n?`, `wing?` |

### 系统

| 工具 | 用途 | 关键参数 |
|------|------|---------|
| `mempalace_hook_settings` | 查/改 hook 行为 | `silent_save?`, `desktop_toast?` |
| `mempalace_memories_filed_away` | 确认最近 checkpoint 是否已保存 | 无 |
| `mempalace_reconnect` | 强制重连 palace 数据库 | 无 |

---

## 常见工作流

### 1. 搜索记忆（最频繁）

```bash
_mp mempalace_search '{"query":"topicinfo 线上域名", "wing":"claude-code-wsl"}'
# 不指定 wing 则全局搜
_mp mempalace_search '{"query":"友圈 AES 加密", "limit":5}'
```

### 2. 存入新记忆

```bash
# 先查重
_mp mempalace_check_duplicate '{"content":"要存的内容"}'
# 再存
_mp mempalace_add_drawer '{"wing":"claude-code-wsl","room":"technical","content":"具体内容","added_by":"claude"}'
```

### 3. 知识图谱查/写

```bash
# 查询某实体的关系
_mp mempalace_kg_query '{"entity":"topicinfo","direction":"outgoing"}'
# 添加事实
_mp mempalace_kg_add '{"subject":"topicinfo","predicate":"domain","object":"topicinfo.nntisixsixsix.com","valid_from":"2025-01-01"}'
# 标记失效
_mp mempalace_kg_invalidate '{"subject":"topicinfo","predicate":"domain","object":"old-domain.com"}'
```

### 4. 探索 Palace 结构

```bash
_mp mempalace_status '{}'
_mp mempalace_get_taxonomy '{}'
_mp mempalace_list_rooms '{"wing":"claude-code-wsl"}'
```

### 5. Agent Diary

```bash
# 写日记
_mp mempalace_diary_write '{"agent_name":"claude","entry":"今天处理了 topicinfo 报警，根因是 DB 连接池耗尽","topic":"incident"}'
# 读最近 5 条
_mp mempalace_diary_read '{"agent_name":"claude","last_n":5}'
```

---

## 踩坑备忘

1. **MCP panel 不显示本地 MCP** — UI 面板只显示云端 MCP，本地 MCP 不在面板里，但已加载。用 `claude mcp list` 验证。
2. **root vs devtrump 用户** — Claude Code VSCode 扩展在 WSL2 以 root 跑，配置在 `/root/.claude/claude.json`，binary 用 `/home/devtrump/.local/bin/mempalace-mcp`。
3. **config.json 两个用户都要有** — `/root/.mempalace/config.json` 和 `/home/devtrump/.mempalace/config.json` 内容相同，指向同一 palace 路径 `/mnt/c/Users/DEVTrump/.mempalace/palace`。
4. **Stop hook 不等于自动识别** — Stop hook 自动存档，但 Claude 不会主动去读；需要明确调用 `/zx-mempalace` 或在 CLAUDE.md 里写规则才能让 Claude 先查。
5. **外部 CLI 操作后需 reconnect** — 手动 mine 或命令行写入后，调用 `mempalace_reconnect` 刷新连接。
