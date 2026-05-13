扫描指定目录下所有 git 仓库，自动处理发散（diverged）分支：fetch → rebase → push。高频用于多仓库本地/远程不同步的场景。

## 使用方式

```
/zx-git-sync [目录]
```

- 不传参数默认扫描 `/home/devtrump/gh-projects/`
- 也可指定单个仓库目录

## 执行步骤

**第一步：扫描所有仓库状态**

```bash
for dir in <目标目录>/*/; do
  [ -d "$dir/.git" ] || continue
  sudo -u devtrump git -C "$dir" fetch origin 2>/dev/null
  ahead=$(sudo -u devtrump git -C "$dir" rev-list @{u}..HEAD 2>/dev/null | wc -l)
  behind=$(sudo -u devtrump git -C "$dir" rev-list HEAD..@{u} 2>/dev/null | wc -l)
  branch=$(sudo -u devtrump git -C "$dir" branch --show-current 2>/dev/null)
  echo "$(basename $dir): ahead=$ahead behind=$behind branch=$branch"
done
```

**第二步：对每个发散仓库（ahead>0 且 behind>0）依次处理**

处理顺序：
1. 修复 `.git/` 目录所有权（root 跑过 git 后可能 ownership 错乱）
2. `git stash`（如有 unstaged changes）
3. `git rebase origin/<branch>`
4. `git push origin <branch>`
5. `git stash pop`（如有 stash）

```bash
chown -R devtrump:devtrump "$dir/.git"
cd "$dir"
# stash if dirty
if ! sudo -u devtrump git -C "$dir" diff --quiet 2>/dev/null; then
  sudo -u devtrump git -C "$dir" stash
  STASHED=1
fi
sudo -u devtrump git -C "$dir" rebase origin/$branch && \
  sudo -u devtrump git -C "$dir" push origin $branch
[ "$STASHED" = "1" ] && sudo -u devtrump git -C "$dir" stash pop
```

**第三步：汇报结果**

列出每个仓库最终状态：✅ 已同步 / ⚠️ 有冲突需手动处理 / ⏭️ 无需操作

## 处理原则

- **只 ahead（behind=0）**：直接 push，不 rebase
- **只 behind（ahead=0）**：直接 pull --ff-only
- **both ahead & behind（发散）**：rebase onto remote，再 push
- **rebase 冲突**：abort rebase，输出冲突文件列表，**不强推**，提示用户手动处理
- **不操作 main/master 的 force push**

## 踩坑备忘

1. **`.git/` ownership 问题** — VSCode 扩展以 root 运行，root 跑过 git 后 `.git/objects` 等文件归 root 所有，`sudo -u devtrump git` 会报 `insufficient permission`。每次操作前先 `chown -R devtrump:devtrump .git`
2. **fetch 要先跑** — `rev-list @{u}..HEAD` 依赖本地 remote-tracking ref 是最新的，不 fetch 就看不到 remote 的新 commit
3. **unstaged changes 会让 rebase 失败** — 必须先 stash
4. **`rebase --abort` 清理现场** — 如果上次 rebase 中断了，先 abort 再重试
