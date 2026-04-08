---
name: workflow-git
description: 原生 git worktree 工作流，统一管理主仓与多工作树分支，支持并行开发、进度保存与安全发布。
---
# Workflow Git

规范基于 Git 原生 `git worktree`，用于在单仓库内安全进行并行开发、提交、合并与发布。

## 主分支自动识别（main/master 兼容）

所有涉及主分支的命令，必须先识别默认分支，不写死 `main`。

统一识别片段：
```bash
BASE_BRANCH=$(git -C "$MAIN_REPO" symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's#^origin/##')
if [ -z "$BASE_BRANCH" ]; then
  if git -C "$MAIN_REPO" show-ref --verify --quiet refs/heads/main; then
    BASE_BRANCH=main
  elif git -C "$MAIN_REPO" show-ref --verify --quiet refs/heads/master; then
    BASE_BRANCH=master
  else
    BASE_BRANCH=main
  fi
fi
echo "BASE_BRANCH=$BASE_BRANCH"
```

后续命令中，把 `origin/main` 统一替换为 `origin/$BASE_BRANCH`。

## 小团队默认策略（轻量 PR 驱动）

适用场景：1-5 人团队、迭代快、希望低成本提升发布稳定性。

默认规则：
1. 功能开发在附加工作树分支进行，不直接向主分支（`$BASE_BRANCH`）提交。
2. 保存进度只推 `origin/<branch>`。
3. 发布走 PR：`push branch` -> `create PR` -> `最小检查通过` -> `squash merge`。
4. 允许自审合并（无人可审时），但必须经过最小检查。

最小检查清单（可按项目裁剪）：
- `lint` 通过（或等价静态检查）。
- 核心单测通过（至少覆盖改动路径）。
- PR 描述包含：改动目的、影响范围、回滚方式（简要即可）。

### 单人值守模式（无人评审时）

触发条件：当前时段无可用评审人，但需要继续以 PR 方式发布。

执行规则：
1. 仍然必须走 PR，不可跳过到直接推 `main`。
2. 使用自审模板完成核对并粘贴到 PR 描述。
3. 仅允许低到中风险改动自审合并；高风险改动必须等待他人复审。

自审模板（可直接粘贴到 PR）：
```md
## Self-review Checklist
- [ ] 我已阅读完整 diff，确认无无关改动/调试残留。
- [ ] lint 已通过，核心测试已通过（或说明未执行原因）。
- [ ] 已评估兼容性影响（接口/配置/数据）。
- [ ] 已给出回滚方案（revert PR 或回退步骤）。
- [ ] 本次改动不涉及高风险域（权限、支付、迁移、删除数据）。
```

合并前强制核对项：
- `git diff --stat origin/$BASE_BRANCH...HEAD` 结果与 PR 目标一致。
- PR 描述含“改动目的、影响范围、回滚方式”三项。
- 未完成项必须在 PR 中明确标注风险与后续计划。

## 自动化快捷命令（推荐）

目标：把高频动作标准化，减少口头沟通和人工漏步。以下命令可直接在 shell 执行，也可由 Agent 按触发词自动执行。

前置约定（每个仓库一次）：
```bash
export MAIN_REPO=/path/to/repo
export WT_REPO=/path/to/repo-wt1
```

### `wt:new` 新任务分支
触发词：“开始新任务”“开新分支”“从 main 拉一条新需求分支”。
```bash
TASK=feature/<task>
BASE_BRANCH=$(git -C "$MAIN_REPO" symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's#^origin/##')
[ -z "$BASE_BRANCH" ] && BASE_BRANCH=main
git -C "$MAIN_REPO" fetch origin && git -C "$MAIN_REPO" switch "$BASE_BRANCH" && git -C "$MAIN_REPO" pull --rebase origin "$BASE_BRANCH"
git -C "$WT_REPO" fetch origin
if git -C "$WT_REPO" show-ref --verify --quiet "refs/heads/$TASK"; then
  echo "分支 $TASK 已存在，将仅切换，不覆盖历史"
  git -C "$WT_REPO" switch "$TASK"
else
  git -C "$WT_REPO" switch -c "$TASK" "origin/$BASE_BRANCH"
fi
git -C "$WT_REPO" status -sb
```

### `wt:push` 保存进度
触发词：“保存进度”“推当前分支”“备份一下代码”。
```bash
BRANCH=$(git -C "$WT_REPO" branch --show-current)
git -C "$WT_REPO" status -sb
git -C "$WT_REPO" push -u origin "$BRANCH"
```

### `wt:sync` 同步主线
触发词：“同步 main”“基于最新主线重放”。
```bash
BASE_BRANCH=$(git -C "$MAIN_REPO" symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's#^origin/##')
[ -z "$BASE_BRANCH" ] && BASE_BRANCH=main
git -C "$MAIN_REPO" fetch origin && git -C "$MAIN_REPO" switch "$BASE_BRANCH" && git -C "$MAIN_REPO" pull --rebase origin "$BASE_BRANCH"
git -C "$WT_REPO" fetch origin && git -C "$WT_REPO" rebase "origin/$BASE_BRANCH"
```

### `wt:done` 合并后收尾
触发词：“收尾”“清理本地分支”“准备下个任务”。
```bash
BASE_BRANCH=$(git -C "$MAIN_REPO" symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's#^origin/##')
[ -z "$BASE_BRANCH" ] && BASE_BRANCH=main
git -C "$MAIN_REPO" fetch origin && git -C "$MAIN_REPO" switch "$BASE_BRANCH" && git -C "$MAIN_REPO" pull --rebase origin "$BASE_BRANCH"
NEXT=feature/<next-task>
git -C "$WT_REPO" fetch origin
if git -C "$WT_REPO" show-ref --verify --quiet "refs/heads/$NEXT"; then
  echo "分支 $NEXT 已存在，将仅切换，不覆盖历史"
  git -C "$WT_REPO" switch "$NEXT"
else
  git -C "$WT_REPO" switch -c "$NEXT" "origin/$BASE_BRANCH"
fi
git -C "$WT_REPO" status -sb
```

说明：
- `wt:done` 默认不删除本地历史分支，避免误删；需删分支时单独确认。
- 若项目使用 GitLab MR，把 “PR” 等价理解为 “MR”。

## 结构与识别

| 角色 | 路径特征 | Remote 约定 | 目的 |
| --- | --- | --- | --- |
| 主工作树 | 通常为仓库根目录（例：`/home/zhangyj/htcode/htwb`） | `origin` 指向远程服务器 | 稳定主线、发布与收尾 |
| 附加工作树 | 目录名常含 `-wtN`（例：`/home/zhangyj/htcode/htwb-wt1`） | 与主仓共享同一 `.git` 与 remotes | 并行开发、隔离改动 |

**每次关键操作前的识别清单**

```bash
git rev-parse --show-toplevel
git worktree list --porcelain
git branch --show-current
git status -sb
git remote -v
git fetch origin
BRANCH=$(git branch --show-current)
if git rev-parse --verify --quiet "origin/$BRANCH" >/dev/null; then
  git rev-list HEAD.."origin/$BRANCH" --count
else
  echo 0
fi
```

> 缓存建议：首次执行以上脚本后，把“主工作树路径、当前工作树路径、分支名、是否 clean、ahead/behind 结果”写入当前会话记忆（例如 `ENV_CACHE.git_status`）。后续步骤若检测到分支未变且缓存未过期，可直接引用缓存输出，只在分支变化或用户要求“重新检测”时再跑一次整套命令。

若检测到当前目录不在 `git worktree list` 中，或分支处于 detached HEAD，立即提示并停止后续动作。

### 自动识别辅助规则
- **目录名**：`basename $PWD` 含 `-wt[数字]` 视为附加工作树，否则默认主工作树。
- **工作树来源**：`git worktree list --porcelain` 中出现当前路径即为受管工作树。
- **分支占用冲突**：同一分支默认只能在一个工作树检出，若冲突应先切换分支或新建分支。

## 工作树操作流程

### 1. 日常提交
- 允许高频小提交：`git add <files> && git commit -m "<type>: <summary>"`。
- 提交类型：`feat`、`fix`、`refactor`、`perf`、`docs`、`style`、`test`、`chore`、`wip`（仅本地开发阶段）。

### 2. 创建并进入新工作树（并行开发）
触发词：“开一个隔离分支”“并行做需求”“新建 worktree”。

步骤：
1. 在主工作树选择基线分支（`$BASE_BRANCH`）：`git fetch origin && git switch "$BASE_BRANCH" && git pull --rebase origin "$BASE_BRANCH"`。
2. 新建分支并创建工作树：
   ```bash
   git worktree add -b <feature-branch> ../<repo>-wt1 "$BASE_BRANCH"
   ```
3. 进入新工作树后确认：`git branch --show-current` 与 `git status -sb`。
4. 如分支已存在，改用：
   ```bash
   git worktree add ../<repo>-wt1 <existing-branch>
   ```

### 3. 保存进度（推送到 origin）
触发词：“保存进度”“备份代码”“先推上去”。

步骤：
1. `git status -sb`，确保无未提交修改。
2. 首次推送分支：`git push -u origin <branch>`。
3. 后续推送：`git push origin <branch>`。
4. 失败时按报错分类处理：权限、分支保护、网络或非 fast-forward。

### 4. 正式发布（轻量 PR 驱动）
触发词：“发布代码”“合并到主分支”“推送服务器”。

1. **确认功能分支已推送**：`git push origin <branch>`。
2. **创建或更新 PR**（建议）：
   ```bash
   gh pr create --fill || gh pr view
   ```
3. **执行最小检查**：确认 lint 和核心测试通过，再继续。
4. **合并 PR（默认 squash）**：
   ```bash
   gh pr merge --squash --delete-branch
   ```
   若未使用 `gh`，可在代码托管平台手动执行同等操作。
5. **切回主工作树并同步**：
   ```bash
   cd <main-worktree-path>
   git fetch origin
   if [ "$(git rev-list HEAD..origin/$BASE_BRANCH --count)" -gt 0 ]; then
     git pull --rebase origin "$BASE_BRANCH"
   fi
   ```
6. **Squash 合并后分支处理**：默认结束该功能分支，不继续在原分支上 `rebase`。如需继续开发，基于最新 `origin/$BASE_BRANCH` 新建分支；若必须复用分支，需先明确授权再执行 `git reset --hard origin/$BASE_BRANCH` 并在日志中记录当次授权人。

### 5. 发布后收尾
- 删除无价值的临时分支。
- 清理已完成工作树：
  ```bash
  git worktree remove ../<repo>-wt1
  git branch -d <feature-branch>   # 若已合并
  git worktree prune
  ```
- 再次 `git status -sb`，确认主工作树与其他工作树均干净。

## 主线操作准则

1. 所有整理、压缩、合并均在主工作树完成。
2. 推送前固定流程：`git fetch origin` -> 判断是否需要 `git pull --rebase` -> `git push origin $BASE_BRANCH`。
3. 不在未同步 `origin/$BASE_BRANCH` 的状态下执行主线合并。
4. 主线提交信息需说明背景、核心改动与影响面。

**提交信息要求**
- 功能工作树：允许 `feat: xxx`、`wip: xxx` 等简短说明。
- 主线：描述功能背景、核心改动点及涉及文件，可追加条列说明，必要时标记联合作者。

## 合并策略

| 提交数 | 策略 | 说明 |
| --- | --- | --- |
| <= 3 | `git merge --no-ff origin/<branch>` | 保留分支语义与历史 |
| > 3 | `git merge --squash origin/<branch>` | 压缩为一条可读提交 |

脚本示例已在正式发布流程中给出，可直接复制执行。

### 轻量 PR 合并约定（默认）
- 默认 `squash merge`，保持 `main` 历史简洁。
- 仅当用户明确要求保留完整分支历史时，改用 `merge commit`。
- 小改动可自审合并；高风险改动（数据库迁移、权限、支付链路）建议至少 1 人复审。
- Squash 合并后不得在原功能分支继续 `rebase` 迭代，避免重复提交；应删除或重建分支，并把处理结果写进“发布后收尾”记录。

## 模糊指令与响应

| 用户表述 | 功能工作树行为 | 主工作树行为 |
| --- | --- | --- |
| “提交代码” | 先澄清：“A. 保存进度到分支 / B. 创建 PR 并发布到 `$BASE_BRANCH`？” | 执行常规 `git add/commit/push origin` |
| “保存进度” | `git push origin <branch>` | 提示当前在主线，若需隔离开发应新建 worktree |
| “合并到主线 / 发布” | 执行 PR 发布流程（建 PR、检查、squash 合并） | 同步 `origin/$BASE_BRANCH` 并回传结果 |
| “查看状态” | 显示 `git status -sb` + `origin/<branch>` ahead/behind | 显示 `git status -sb` + `origin/$BASE_BRANCH` ahead/behind |

## 危险操作与确认

- `git push --force`、`git reset --hard`、删除分支或改写已发布历史，必须在解释风险后获得明确许可。
- 自动化脚本不得暗自修改全局 Git 配置。
- 检测到冲突时立即停下，列出冲突文件并等待用户决策。

## 自动检查与守则

在执行 push/merge 之前，必须：
- `git diff --quiet` 与 `git diff --cached --quiet`，若有未提交改动先处理。
- 工作树发布前确认目标分支已推送 `origin` 且基于最新 `origin/$BASE_BRANCH`。
- 推送任一 remote 前执行 `git fetch <remote>`，确保无落后或超前提交未处理。

## 异常处理

1. **推送失败**：照错误类型区分网络、磁盘、权限、远端被检出，提供逐项解决方案并记录状态。
2. **合并冲突**：标注冲突阶段（功能工作树/主工作树）、冲突文件、建议解决顺序。
3. **工作树创建失败**：常见原因是路径已存在、分支已被其他工作树占用、未提交改动阻塞；需逐项排查。
4. **PR 检查失败**：先修复并补充提交，再 `git push origin <branch>` 更新 PR，禁止绕过检查直接发主线。
5. **远端有新提交**：所有 push 前必须 `git fetch`，如检测到 ahead/behind 不一致，先同步再继续。
6. **分支不一致**：当功能工作树与主线基线偏离过大时，提示用户并确认策略（新建分支重做优先；必要时经授权后 reset）。

## 高频动作速查表

| 场景 | 快捷命令 | 备注 |
| --- | --- | --- |
| 查看当前状态 | `git status -sb` + 缓存的 ahead/behind | 缓存过期或分支变动时重新执行完整识别脚本 |
| 新建任务 | `wt:new <task>` | 已自动处理“分支存在”情况 |
| 保存进度 | `wt:push` | 默认 `push -u`；已有跟踪关系时自动降级为 `push` |
| 同步主线 | `wt:sync` | 在主/工作树均执行 `fetch`，需确保当前分支 clean |
| 发布流程 | `wt:push` → `gh pr create/merge` → 主工作树同步 | 发布后立即执行“Squash 合并后分支处理” |
| 收尾/准备下个任务 | `wt:done <next-task>` | 会拉取最新主线并切到下个任务模板 |

这些速查指令位于脚本章节顶部，可在简单询问（如“帮我推一下分支”）时直接引用，无需整份 Skill 全量加载。
