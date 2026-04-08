---
name: workflow-branch-naming
description: 根据需求描述生成规范 Git 分支名。用户提到“生成分支名”“branch name”“分支命名”时使用，输出单行结果。
---
# Workflow Branch Naming

用于把自然语言需求描述转换为可落地的分支名。

## 输入

- `desc`：需求描述（必填）。
- `date`：可选，格式 `yyyyMMdd`；若缺失使用当前日期。
- `branch_style`：可选；默认格式为 `{type}-{date}-{slug}`。

## 输出

- 仅输出一行分支名。
- 不输出解释、前后缀、标点或额外文本。

## 生成规则

### 1) type 判定

允许值：`feat fix hotfix perf refactor docs test chore ci build`

优先级：
`hotfix(紧急/回滚/prod) > perf(性能) > fix(bug/修复) > feat(新增) > refactor(结构不变) > docs > test > ci > build > chore`

### 2) ticket 提取

识别并标准化：
- `ABC-123`
- `#456`
- `issue 789`

处理方式：
- 全部转小写，去掉 `#`。
- 若存在 ticket，置于 `slug` 最前，形成 `ticket-slug`。

### 3) slug 生成

- 取核心关键词 3-8 词。
- 词间用 `-`。
- 过滤泛词：`一下/相关/进行/支持/增加/优化/问题/功能/页面/接口/调整/更新/修改` 等。
- 必要时将中文概念转换为常见英文词（如 `login/order/pay`）；无法稳定转换的词丢弃。
- 默认长度约束：最多 4 词，且总长度不超过 24 个字符。
- 若超长，优先减少词数，再对末词做安全截断；截断后仍需保持可读语义。

### 4) 字符约束与清洗

- 仅允许：`a-z`、`0-9`、`-`、`/`、`.`
- 全小写
- 禁止空格、中文、下划线及其他符号
- 渲染后规范化：
  - 连续分隔符压缩（如 `--`、`//`）
  - 去掉首尾 `-`、`/`、`.`
  - 空变量不产生多余分隔符
  - 若最终分支名超过 48 字符，继续压缩 `slug` 直到满足限制

## 默认输出模板

`{type}-{date}-{slug}`

## 示例

输入：
- `date=20260408`
- `desc=修复 #456 登录后订单页白屏问题`

输出：
`fix-20260408-456-login-order-blank-screen`
