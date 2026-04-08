---
name: workflow-git
description: 三层 Git 工作流，串联工作区、cache 与主项目，支持保存进度与正式发布。
---
# Workflow Git

规范来自《02-git-workflow》，用于在主项目、工作区与 cache 之间安全流转代码。

## 仓库架构与识别

| 角色 | 路径特征 | Remote 约定 | 目的 |
| --- | --- | --- | --- |
| 主项目 | 目录名不含 `-wt*`（例：`/home/zhangyj/htcode/htwb`） | `origin` 指向远程服务器 | 存放整洁、可发布的历史 |
| 工作区 | 目录名含 `-wtN`（例：`/home/zhangyj/htcode/htwb-wt1`） | 必有 `origin`（主项目本地路径）+`cache`（裸仓库） | 快速迭代、小步提交 |
| cache 仓库 | 裸仓库（例：`/home/zhangyj/htcode-ai/htwb.git`） | 只被推送/拉取，不检出 | 工作区向主项目的中转 |

**每次关键操作前的识别清单**

```bash
git remote get-url cache >/dev/null 2>&1 && echo "工作区" || echo "主项目"
git branch --show-current
git status -sb
git remote -v   # 确认 cache/origin 路径
git fetch <remote> && git rev-list HEAD..<remote>/<branch> --count
```

若检测到 cache remote 缺失或路径不存在，立即提示并停止后续动作。

### 自动识别辅助规则
- **目录名**：`basename $PWD` 含 `-wt[数字]` 视为工作区，否则默认主项目。
- **origin 指向**：`git remote get-url origin` 若为本地路径，则通常说明当前位置是工作区。
- **cache 路径验证**：`ls -ld $(git remote get-url cache)`，确认裸仓库存在且可写。

## 工作区操作流程

### 1. 日常提交
- 允许高频小提交：`git add <files> && git commit -m "<type>: <summary>"`。
- 提交类型：`feat`、`fix`、`refactor`、`perf`、`docs`、`style`、`test`、`chore`、`wip`（仅工作区）。

### 2. 保存进度（推送到 cache）
触发词：“保存进度”“提交到工作区”“备份代码”“推送到 cache”。

步骤：
1. `git status -sb`，确保无未提交修改。
2. `git push cache <branch>`。
3. 失败时根据报错分类处理：磁盘空间、权限、远端被检出等，修复后再推。

### 3. 正式发布（推主项目 + 远程）
触发词：“发布代码”“合并到主项目”“推送到服务器”。

1. **推送 cache**：`git push cache <branch>`，保证主项目可拉取最新提交。
2. **切换至主项目仓库**：`cd /home/zhangyj/htcode/<project>`，必要时补充 `git remote add cache <path>`。
3. **主项目同步远程**：
   ```bash
   git fetch origin
   if [ "$(git rev-list HEAD..origin/main --count)" -gt 0 ]; then
     git pull --rebase origin main
   fi
   ```
4. **从 cache 合并**：
   ```bash
   git fetch cache
   commit_count=$(git rev-list HEAD..cache/main --count)
   if [ "$commit_count" -le 3 ]; then
     git merge cache/main --no-edit
   else
     git merge --squash cache/main
     git commit -m "feat: 合并工作区的 ${commit_count} 个提交"
   fi
   ```
5. **推送主项目**：`git push origin main`，禁止直接在工作区对远程 `origin` 推送。
6. **回到工作区同步**：`git fetch origin && git merge --ff-only origin/main`，保持历史一致。

### 4. 发布后收尾
- 删除无价值的临时分支。
- 再次 `git status -sb`，确认主项目与工作区均干净。

## 主项目操作准则

1. 所有整理、压缩、合并均在主项目仓库完成。
2. 推送前固定流程：`git fetch origin` → 判断是否需要 `git pull --rebase` → `git push origin main`。
3. 不得跳过 cache 直接合并工作区代码。
4. 主项目提交信息需详细说明背景、改动与影响面。

**提交信息要求**
- 工作区：允许 `feat: xxx`、`wip: xxx` 等简短说明。
- 主项目：描述功能背景、核心改动点及涉及文件，可追加条列说明，必要时标记联合作者。

## 合并策略

| 提交数 | 策略 | 说明 |
| --- | --- | --- |
| ≤ 3 | `git merge cache/main --no-edit` | 保留完整历史 |
| > 3 | `git merge --squash cache/main` | 将多个提交压缩为一条易读记录 |

脚本示例已在正式发布流程中给出，可直接复制执行。

## 模糊指令与响应

| 用户表述 | 工作区行为 | 主项目行为 |
| --- | --- | --- |
| “提交代码” | 先澄清：“A. 保存进度 / B. 正式发布？” | 执行常规 `git add/commit/push origin` |
| “保存进度” | `git push cache <branch>` | 提示当前已在主项目，仅提交不推送 |
| “合并到主项目 / 发布” | 执行完整发布流程 | 推送 `origin`，无需 cache |
| “查看状态” | 显示 `git status -sb` + cache ahead/behind | 显示 `git status -sb` + origin ahead/behind |

## 危险操作与确认

- `git push --force`、`git reset --hard`、删除分支或改写已发布历史，必须在解释风险后获得明确许可。
- 自动化脚本不得暗自修改全局 Git 配置。
- 检测到冲突时立即停下，列出冲突文件并等待用户决策。

## 自动检查与守则

在执行 push/merge 之前，必须：
- `git diff --quiet` 与 `git diff --cached --quiet`，若有未提交改动先处理。
- 工作区发布前确认 cache、主项目路径存在；否则提示创建或修复 remote。
- 推送任一 remote 前执行 `git fetch <remote>`，确保无落后或超前提交未处理。

## 异常处理

1. **推送失败**：照错误类型区分网络、磁盘、权限、远端被检出，提供逐项解决方案并记录状态。
2. **合并冲突**：标注冲突阶段（工作区/主项目）、冲突文件、建议解决顺序。
3. **cache 缺失**：提示路径不存在，可指导重新创建裸仓库或请用户指定新地址。
4. **远端有新提交**：所有 push 前必须 `git fetch`，如检测到 ahead/behind 不一致，先同步再继续。
5. **分支不一致**：当工作区与主项目分支不相同时，提示用户并确认处理策略（切换/新建分支/重建工作区）。
