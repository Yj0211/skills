---
name: workflow-hub
description: 全局工作流枢纽，按任务类型串联环境、Git、质量、测试与构建部署规范。
---
# Workflow Hub

本技能整合环境识别、`workflow-git`、`workflow-quality-release` 的关键要求，为所有任务提供统一的操作顺序。

## 固定行为

1. 全程使用简体中文回复，先给结论再给关键证据。
2. 任何关键命令前必须完成环境检测，确认仓库角色、分支、远端与工具版本。
3. 优先使用 `rg`、编辑器等专业工具；Shell 仅执行必要命令。
4. 模糊意图（如“提交代码”）必须先澄清；涉及 `git push --force`、`git reset --hard`、改写历史等危险操作须获明确授权。
5. 流程中断或失败时，说明原因、影响范围，并给出至少一个可执行的下一步。

## 环境识别（来自《01-environment》）

执行关键操作前快速检查：

```bash
# 操作系统与架构
uname -s && uname -m

# Git 仓库角色
git worktree list --porcelain
git branch --show-current
git status -sb

# 项目类型提示
ls package.json pyproject.toml go.mod Cargo.toml 2>/dev/null

# 关键工具版本
node --version 2>/dev/null
python --version 2>/dev/null
go version 2>/dev/null
```

- Node/前端项目：通过 `package.json` 判断框架（`"vue"`, `"react"`, `"next"`, `"nuxt"`），依据 lock 文件识别包管理器。
- Python 项目：查 `pyproject.toml`/`requirements.txt`/`setup.py` 以及 `.venv`/`venv`。
- Go/Rust 项目：查 `go.mod` 或 `Cargo.toml` 并输出版本。
- 环境检测触发：会话首次进入项目、执行 Git/构建/部署前、或用户要求查看环境时。

### 常见环境问题与快速处理
- **Node 版本不符**：`cat package.json | grep '"node"'` 查看要求，使用 `nvm use <version>` 切换。
- **依赖安装失败**：清理缓存后重装，例如 `rm -rf node_modules package-lock.json && npm install` 或 `pnpm install --force`。
- **Python 模块缺失**：激活虚拟环境（`source venv/bin/activate`），`pip install -r requirements.txt`。
- **工作树异常**：`git worktree list --porcelain` 检查是否受管；必要时 `git worktree prune` 清理失效记录。

### 环境检测行为与输出模板
- **检测时机**：会话开始、关键 Git/构建/部署命令前、以及用户说“查看环境/配置”时必须执行。
- **原则**：按需、轻量、可重复、失败不阻断主流程但需提示。
- **输出格式**：
  ```
  === 环境检测 ===
  项目类型: Vue 3 (pnpm)
  Git 环境: 工作树 /home/xxx/project-wt1，分支 feature/x，状态 clean
  工具版本: Node v20.10.0, npm 10.2.3
  结论: ✅ 环境正常
  ```

## 标准作业顺序

1. **环境识别**  
   - 使用上面脚本定位仓库角色。若为附加工作树，记录主工作树路径与当前分支；若工作树异常，立即汇报。  
   - 记录当前分支与工作区脏/洁净状态。
   - 若本会话已缓存前述信息（参见 `workflow-git` 中的缓存建议），且分支/工作树未变化，可直接引用缓存结果，仅在用户要求“重新检测”或自动检测到分支切换时重新执行命令。

2. **意图拆解**  
   - 结合触发词将需求映射到子技能：  
     | 用户意图 | 需调用技能 |
     | --- | --- |
     | “保存进度”“备份代码”“推分支” | `workflow-git`（工作树分支→origin） |
     | “发布代码”“合并主分支”“推送服务器” | `workflow-git`（轻量 PR 发布） |
     | “开始新任务”“同步 main”“收尾清理” | `workflow-git`（自动化快捷命令） |
     | “生成分支名”“branch name”“分支命名” | `workflow-branch-naming`（规范分支名生成） |
     | “写提交信息”“生成 commit message” | `workflow-review-commit`（Conventional Commits） |
     | “代码审查”“review 本次改动” | `workflow-review-commit`（结构化审查模板） |
     | “提交前检查”“质量检查”“跑测试” | `workflow-quality-release`（质量+测试） |
     | “构建”“部署”“上线” | `workflow-quality-release`（构建/部署章节） |

3. **流程执行**  
   - **Git 操作**：严格遵循 `workflow-git` 的工作树识别、分支推送、主线合并与发布策略，禁止跳过任一步。  
   - **质量与测试**：按 `workflow-quality-release` 的顺序执行格式化→Lint→类型→单测→安全→集成/E2E，必要时提供 Mock 与覆盖率数据。  
   - **构建与部署**：确认环境变量、依赖、构建命令；构建成功后按目标环境选择 `scp/rsync`、PM2、Docker 或 CI/CD，并做上线验证。

4. **结果输出**  
   - 模板：“结论（通过/未完成/阻塞） + 关键命令与结果 + 下一步”。  
   - 对未执行的步骤说明原因（例如依赖脚本缺失）及补救计划。

## 轻量加载与模块化规则

1. **懒加载优先**：根据意图拆解的结果，只在确实需要执行某模块动作时才加载对应 Skill；例如单纯查询 “当前分支状态” 时，仅引用 `workflow-git` 的速查表即可，不必全量复述整份文档。
2. **速查通道**：`workflow-git` 顶部的 `wt:new/wt:push/...` 快捷指令可独立引用；若操作符合速查表内容，可直接贴命令并说明出处。
3. **只读场景例外**：当用户明确为“仅 review / 仅说明流程 / 只读 diff”且不会执行 Git 命令时，可跳过 `workflow-git`，改为引用 `workflow-review-commit` 或 `workflow-branch-naming`。
4. **质量/构建模块**：只有在要执行 lint/测试/构建/部署命令时才加载 `workflow-quality-release`，否则保持引用级别说明。
5. **命名/审查模块**：分支命名请求加载 `workflow-branch-naming`；提交信息与代码审查请求加载 `workflow-review-commit`。
6. **环境诊断**：若任务仅为“确认环境/工具版本”，引用当前章节即可，无需进入 Git 流程；检测完成后如任务继续触发 Git 操作，再按需加载子 Skill。

## 触发词与预期响应

| 触发词 | 答复要点 |
| --- | --- |
| “提交代码” | 询问是保存进度（工作树分支→origin）还是创建 PR 后发布（分支→PR→`$BASE_BRANCH`）。 |
| “生成分支名 / branch name” | 路由到 `workflow-branch-naming`，按规则只返回单行分支名。 |
| “写提交信息 / commit message” | 路由到 `workflow-review-commit`，按 Conventional Commits 模板输出。 |
| “代码审查 / review” | 路由到 `workflow-review-commit`，按概览 + 问题表格输出。 |
| “保存进度” | 解释将推送到 `origin/<branch>`，不直接改动 `$BASE_BRANCH`。 |
| “开始新任务 / 同步 main / 收尾” | 直接套用 `workflow-git` 中 `wt:new`、`wt:sync`、`wt:done` 模板执行。 |
| “发布/上线/推送服务器” | 列出完整步骤（推分支 → 建 PR → 最小检查 → squash 合并 → 同步 `origin/$BASE_BRANCH`）。 |
| “质量/测试” | 输出计划（格式化→Lint→类型→测试→安全），逐项执行并给结果。 |
| “构建/部署” | 先确认环境与变量，再运行构建和部署脚本，最后做上线验证。 |

## 输出风格与失败处理

- 列出执行过的命令、关键参数和摘要输出，例如 `npm run test`、`git worktree add ...`、`git push origin <branch>`。
- 若引用规范，可标注章节名（如“参照《03-code-quality》提交前自检”）。
- 失败时需覆盖：错误信息、受影响范围、下一步（重试、修复、回滚或等待信息）。
- 不得静默跳过任一步骤；缺项要明确说明原因（脚本缺失、权限不足等）并提供替代建议。
