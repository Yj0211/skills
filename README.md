# Workflow Skills Collection

本仓库遵循 Anthropics 官方技能仓库结构，将所有技能放在 `skills/` 目录下，方便集中维护与分发。各技能均独立包含 `SKILL.md` 与说明，可与 Codex/Claude `skills` 语义兼容。

## 技能一览

| 技能名 | 说明 | 路径 |
| --- | --- | --- |
| workflow-hub | 全局枢纽：负责环境识别、意图拆解与子流程编排。 | `skills/workflow-hub/SKILL.md` |
| workflow-git | 原生 Git worktree 工作流：覆盖并行开发、分支进度保存与主线发布。 | `skills/workflow-git/SKILL.md` |
| workflow-review-commit | 提交信息生成 + 代码审查模板：统一 Conventional Commits 与评审输出结构。 | `skills/workflow-review-commit/SKILL.md` |
| workflow-quality-release | 代码质量 + 测试 + 构建部署一体化流程。 | `skills/workflow-quality-release/SKILL.md` |

## 使用方式

1. 将本仓库克隆或下载到任意可访问路径，例如 `~/skills-repo`。
2. 在 Codex/Claude 中将 `skills/` 目录加入 `SKILLS_PATH` 或对应加载配置。
3. 当需要更新规范时，只改动单个技能目录即可，通过 Git 统一管理变更。

## 约定

- 所有技能遵循“简体中文、先结论后证据”输出约定。
- 技能间引用只使用相对路径（例如 `../workflow-git/SKILL.md`），以便在仓库内移动或复用。
- 若需添加新技能，请在 `skills/` 下新建文件夹，包含 `SKILL.md` 与必要资源，并更新上表。
