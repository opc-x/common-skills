# git-clean-stage — 清理 git 待提交，把不该提交的加到 .gitignore

扫描当前目录下所有 git 工程，识别不该提交的文件（生成物、大型二进制、本地工具缓存等），自动写入各 repo 的 `.gitignore`，只保留真正有效的待提交文件。

## 触发方式

```
/git-clean-stage
/git-clean-stage /path/to/parent   # 指定扫描根目录
```

---

## 执行流程

### Step 0：找出所有 git 工程

```bash
find <root> -name ".git" -maxdepth 3 -type d | sed 's|/.git||'
```

若未指定路径，`<root>` 默认为当前工作目录。

---

### Step 1：汇总每个 repo 的待提交文件

对每个 repo 运行：

```bash
git -C <repo> status --short -u
```

统计总数，跳过 0 个待提交的 repo，只处理有变更的。

---

### Step 2：识别不该提交的文件

对每个 repo 的未追踪文件（`??`）和已追踪修改（`M`），逐类判断：

**必须 gitignore（加入 .gitignore）：**

| 类型 | 特征 | 示例 |
|---|---|---|
| 大型二进制文件 | PDF / 图片 / 音视频，单文件 > 1MB | `*.pdf`, `*.mp4` |
| 生成的数据目录 | embedding、index、jsonl、向量文件 | `structured/`, `*.jsonl`, `*.json`（如果是生成物） |
| 本地 AI 工具缓存 | `.agents/`、`skills-lock.json`（Claude Code skills） | `.agents/`, `skills-lock.json` |
| 编译/构建产物 | `__pycache__/`, `*.pyc`, `dist/`, `build/` | 通常已在 .gitignore，补漏 |
| 审计/临时输出目录 | 脚本运行输出的 audit/ log/ tmp/ | `audit/`, `logs/` |
| 本地环境文件 | `.env`, `.envrc`, `*.local` | 通常已覆盖，补漏 |

**保留提交（不动）：**

- 源代码文件（`.py`, `.ts`, `.go`, `.sql` 等）
- 文档（`.md`）
- 配置文件（`*.yaml`, `*.toml`, `package.json`）
- DB migration 文件（`supabase/migrations/*.sql`）
- Plugin hooks、.gitignore 本身的修改

**判断大文件：**

```bash
# 检查单个文件大小
du -sh <file>

# 检查目录总大小
du -sh <dir>/
```

超过 10MB 的目录或超过 1MB 的单一非文本文件，默认建议 gitignore。

---

### Step 3：写入 .gitignore

对每个 repo 需要忽略的项，追加到该 repo 的 `.gitignore`：

1. 先读取已有 `.gitignore`，避免重复添加
2. 按类型分组追加，附加注释说明原因
3. 格式：

```
# <原因说明>
<pattern>
```

示例：

```gitignore
# Generated embeddings and index data (rebuilt by scripts)
structured/*.json
structured/*.jsonl

# Local Claude Code skills cache
.agents/
skills-lock.json

# Large binary source files (use LFS or external storage)
callan-method-origin/
```

---

### Step 4：验证结果

写入后重新运行 `git status --short -u`，确认：

1. 忽略前后文件数对比（应该减少）
2. 剩余文件都是可以提交的有效变更
3. 输出每个 repo 的最终待提交清单

---

### Step 5：检查 remote 权限（可选）

如果 repo 有待提交文件，同时检查能否推送：

```bash
git -C <repo> remote -v
```

若 remote 指向别人的 repo（不是自己 GitHub 账号），提示用户：

```
⚠️  <repo>: remote 是 <other-org>/<repo-name>，你没有写权限
   → 建议：gh repo fork <other-org>/<repo-name> --clone=false
            git remote set-url origin git@github.com:<your-account>/<repo-name>.git
```

用 `gh auth status` 获取当前登录的 GitHub 用户名做对比。

---

## 输出格式

```
=== git-clean-stage 扫描报告 ===

扫描根目录：/Users/xxx/gh-projects
发现 git 工程：8 个，有待提交：3 个

--- kb-personal (13 → 2 files) ---
✅ gitignore 新增：callan-method-origin/（12 个 PDF，40MB）
剩余待提交：
  M  .gitignore
  ?? windows-mempalace-global-install-runbook.md

--- callan-method (36 → 26 files) ---
✅ gitignore 新增：structured/*.json, structured/*.jsonl（271MB）, audit/
剩余待提交：
  M  .gitignore
  M  book-1.md ... book-12.md（12 个）
  ?? audit_callan_structure.py
  ?? extract_pymupdf.py
  ...

--- talkflow (46 → 8 files) ---
✅ gitignore 新增：.agents/, skills-lock.json
剩余待提交：
  M  .gitignore
  M  docs/data-pipeline/overview.md
  ?? supabase/migrations/202605120001_callan_kb_schema.sql
  ...

=== 汇总 ===
处理前总计：95 个文件
处理后总计：36 个文件（全部为有效变更）
新增 .gitignore 规则：7 条（跨 3 个 repo）
```

---

## 常见不该提交的模式速查

```
# AI / 向量数据
*.jsonl
*.embeddings
embeddings/
chroma/
faiss/

# Claude Code 本地工具
.agents/
skills-lock.json

# Python 生成物
__pycache__/
*.pyc
*.egg-info/
.venv/
dist/

# 大型原始数据
*.pdf
*.mp4
*.mp3
raw/
original/

# 审计/日志输出
audit/
logs/
*.log

# 本地环境
.env
.envrc
*.local
```
