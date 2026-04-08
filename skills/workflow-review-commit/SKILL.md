---
name: workflow-review-commit
description: 生成规范的 Conventional Commits 提交信息，并按固定模板执行代码审查。当用户提到 commit message、提交说明、代码审查、review、PR 审查时优先使用。
---
# Workflow Review Commit

用于两类高频文本任务：
1. 基于暂存改动生成 commit message。
2. 基于给定 diff 和提交历史执行结构化代码审查。

## 触发与分流

- 用户说“写提交信息”“生成 commit message” -> 走“提交信息模式”。
- 用户说“代码审查”“review 这次改动” -> 走“代码审查模式”。
- 若用户同时要求两者，先生成提交信息，再做代码审查。

## 提交信息模式

### 规则
- 遵循 Conventional Commits：`<type>(<scope>): <description>`。
- `type` 仅允许：`feat`、`fix`、`docs`、`style`、`refactor`、`perf`、`test`、`chore`、`ci`、`build`。
- `scope` 可选，建议使用模块名或目录名。
- `description` 使用简体中文，简洁明确，建议不超过 50 字。
- 变更复杂时可添加正文，说明动机和影响。

### 输入优先级
1. 用户明确提供的 `{recent_commits}`、`{staged_stat}`、`{staged_diff}`。
2. 若用户未提供这些变量，先提示需要提供或授权采集再继续。

### 输出要求
- 只输出 commit message 正文，不加解释。

## 代码审查模式

### 规则
- 始终使用用户指定语言回复（默认简体中文）。
- 仅基于用户提供的 diff 和提交历史审查，不假设未给出的上下文。
- 审查重点：正确性、边界条件、可读性、性能、测试覆盖。
- 不做无关吹毛求疵，优先报告真实风险和可执行修复建议。

### 输出模板
- 先给“整体质量概览”（1 段）。
- 再给问题表格，列为：`序号` | `行号` | `代码` | `问题` | `潜在解决方案`。
- 若无问题，明确写“未发现阻断问题”，并补充残余风险或测试缺口。

### 行号规则
- 如果输入 diff 没有可靠行号，`行号` 填 `N/A`，不得臆造。

## 可复用提示词模板

### 模板 A：提交信息生成
```text
你是一个 Git commit message 生成助手。请根据以下信息生成规范的 commit message。

要求：
1. 遵循 Conventional Commits 规范
2. 格式：<type>(<scope>): <description>
3. type 包括：feat, fix, docs, style, refactor, perf, test, chore, ci, build
4. scope 可选，表示影响范围
5. description 使用中文，简洁明了（建议 <= 50 字）
6. 如果变更较复杂，可以添加正文说明

参考最近的提交风格：
{recent_commits}

变更摘要：
{staged_stat}

变更详情：
{staged_diff}

请直接输出 commit message，无需解释。
```

### 模板 B：代码审查
```text
请始终使用 {language} 回复。你正在对当前分支的变更进行代码审查。

## 代码审查指南

下面提供了该分支的完整 git diff 以及所有提交记录。

关键提示：你需要的所有信息都已经在下方提供。
请勿运行 git diff、git log、git status 或任何其他 git 命令。

审查 diff 时请：
1. 关注逻辑和正确性（bug、边界情况、潜在问题）
2. 考虑可读性（清晰度、可维护性、仓库最佳实践）
3. 评估性能（明显性能问题或优化点）
4. 评估测试覆盖率（是否有足够测试）
5. 提出澄清问题（仅在确实缺上下文时）
6. 不要过于吹毛求疵

输出格式：
- 提供代码整体质量的概览摘要。
- 将发现问题以表格形式呈现，包含以下列：序号（1, 2, 等）、行号、代码、问题、潜在解决方案。
- 如果没有发现问题，简要说明代码符合最佳实践，并说明残余风险或测试缺口。

行号规则：
- 若输入未提供可靠行号，行号填写 N/A，不要臆造。

## 完整的 Diff
{git_diff}

## 提交历史
{git_log}
```

## 示例参考

- 需要快速对齐输出风格时，读取 [examples.md](examples.md)。
