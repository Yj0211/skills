---
name: workflow-quality-release
description: 代码质量、测试、构建与发布的一体化流程，适用于提交前与上线前检查。
---
# Workflow Quality Release

依据《03-code-quality》《04-testing》《05-build-deploy》三份规范整合而成，用于保障提交与发布前的质量闭环。

## 代码质量职责

### 1. 编码前
- 明确需求目标、输入输出、边界情况。
- 预估性能和安全风险，规划可测试的架构。
- 按功能拆解任务，设计数据结构与核心算法。

### 2. 编码中
- 保持与现有代码一致的风格与命名。
- 单个函数≤50行、单文件≤300行，复杂逻辑拆分。
- 仅在必要处添加说明算法意图、约束或副作用的注释。
- 对所有输入做校验，杜绝硬编码密码、token。

### 3. 提交前
- 按顺序执行：格式化 → Lint → 类型检查 → 单元测试 → 安全扫描。
- 清理未使用的导入、变量、注释代码，移除调试输出。
- 提交信息遵循 `<type>: <summary>`（type 为 feat/fix/refactor/perf/docs/style/test/chore）。

### 命名与代码风格
- **变量/函数**：优先语义命名（`getUserById`、`isValid`），避免单字母。
- **常量**：使用全大写下划线（`MAX_RETRY_COUNT`）。
- **类/组件**：PascalCase（`UserService`、`UserProfile`）。
- **缩进**：2 空格；运算符、逗号后保留空格。
- **函数/文件长度**：单函数≤50行，单文件≤300行，超过需拆分。
- **注释**：描述算法意图或边界条件，避免“无意义”注释；复杂函数可使用 JSDoc。

### 质量检查清单
- **安全**：验证所有输入、参数化查询、输出转义、隐藏敏感信息、客户端不存储机密。
- **性能**：避免循环重复计算，选用合适数据结构，列表大时采用虚拟滚动/懒加载，必要时缓存请求。
- **错误处理**：异步操作加 try/catch 或 `Promise.catch`，日志记录+用户友好提示。
- **可维护性**：函数职责单一、无重复代码、魔法数字提常量、复杂逻辑附注释并配套测试。

### 自动化工具基线
- **ESLint**（示例配置）：
  ```json
  {
    "extends": ["eslint:recommended", "plugin:@typescript-eslint/recommended"],
    "rules": {
      "no-console": "warn",
      "no-unused-vars": "error",
      "semi": ["error", "never"],
      "quotes": ["error", "single"]
    }
  }
  ```
- **Prettier**：
  ```json
  { "semi": false, "singleQuote": true, "tabWidth": 2, "trailingComma": "es5", "printWidth": 100 }
  ```
- **TypeScript**：坚决避免 `any`，以 interface/type 精确定义数据结构，使用 `tsc --noEmit` 做类型闸门。

## 自检命令模板

> 请以项目实际脚本为准，缺失时需说明替代方案。

| 检查 | 典型命令 | 目标 |
| --- | --- | --- |
| 格式化 | `npm run format` | 统一风格、避免 diff 噪音 |
| Lint | `npm run lint` / `eslint .` | 静态规则全通过 |
| 类型 | `npm run type-check` / `tsc --noEmit` | 无类型错误 |
| 单元测试 | `npm run test` | 断言通过、覆盖率达标 |
| 覆盖率 | `npm run test:coverage` | 核心逻辑 ≥90%，整体 ≥70% |
| 安全 | `npm audit` + 关键字扫描 | 阻断高危漏洞与泄密 |

任一环节失败立即停止后续步骤，记录报错、影响面与修复建议。

## 测试策略（测试金字塔）

```
        /\
       /  \  E2E：关键业务（≈10%）
      /____\
     /      \  集成：跨模块/外部接口（≈20%）
    /________\
   /          \  单元：核心逻辑与工具（≈70%）
  /______________\
```

- **单元测试**：覆盖工具函数、算法、数据处理；示例结构 `describe/it`，断言包含正反两类场景。
- **集成测试**：重点验证 API、数据库、多模块联动，可用 Jest + Supertest。
- **E2E 测试**：Playwright/其它框架模拟用户登录、支付等关键路径。
- **Bug 修复流程**：先写能复现问题的失败测试，再修复并确认通过。
- **发布前顺序**：`npm run test` → `npm run test:integration` → `npm run test:coverage` → `npm run test:e2e`，并输出覆盖率摘要。

### 测试用例示例
```javascript
describe('getUserById', () => {
  it('返回正确用户', async () => {
    const result = await getUserById('123')
    expect(result).toEqual({ id: '123', name: 'Alice' })
  })

  it('用户不存在时报错', async () => {
    await expect(getUserById('none')).rejects.toThrow('User not found')
  })
})
```

## Mock 与稳定性
- 需要隔离外部依赖、网络或时间因素时使用 Mock（示例：`jest.mock(...)`、`jest.useFakeTimers()`）。
- 针对易抖动测试，避免真实时间/网络依赖，保证 `await` 完整等待异步。
- 测试执行缓慢时可通过 `npm run test -- file.test.ts`、`--testNamePattern` 局部运行。

## 测试工具与流程

- **Jest/集成测试**
  ```bash
  npm install --save-dev jest @types/jest
  npm run test              # 全量
  npm run test:watch        # 监听
  npm run test:coverage     # 覆盖率
  ```
  API 场景可配合 Supertest。

- **Playwright/E2E**
  ```bash
  npm install --save-dev @playwright/test
  npx playwright test          # 无头
  npx playwright test --headed # 可视化
  npx playwright test --debug  # 调试
  ```
  重点模拟登录、支付等关键链路。

- **Claude 测试操作顺序**
  1. 新功能尽量 TDD：先写失败测试→实现→全部通过。
  2. Bug 修复严格“先复现后修复”。
  3. 发布前按金字塔顺序执行全量测试并附覆盖率摘要。

### 常见测试问题
- **Flaky（结果不稳定）**：避免真实时间/网络依赖，使用 `jest.useFakeTimers()`、Mock API，并确认每个异步 `await`。
- **运行过慢**：对第三方请求使用 Mock，利用 `npm run test -- <pattern>` 局部执行，必要时并行或跳过无关 E2E。

## Mock 数据管理
- 使用 `jest.mock()` 或伪造对象隔离外部依赖，API 未就绪可返回固定响应。
- `jest.useFakeTimers()` / `jest.setSystemTime()` 控制时间依赖。
- 数据库/缓存可提供内存 Mock，确保测试可重复。
```javascript
jest.mock('../api/userApi')
import { getUserById } from '../api/userApi'

getUserById.mockResolvedValue({ id: '123', name: 'Alice' })
const mockDb = { users: [{ id: '1', name: 'Alice' }] }
```

## 构建流程

1. **构建前检查**：`npm install` → 测试 → 类型检查 → Lint，全部通过后方可构建。
2. **执行构建**：
   ```bash
   npm run build:dev   # 开发/测试环境
   npm run build       # 生产环境
   NODE_OPTIONS=--max-old-space-size=4096 npm run build  # 内存不足时
   ls -lh dist/        # 校验产物
   ```
3. **构建优化**：启用路由懒加载、Tree Shaking（按需引入）、压缩与 Gzip/Brotli。

## 部署前检查清单

- [ ] 所有测试（含 E2E）通过。
- [ ] `npm run build` 成功且产物可用。
- [ ] `npm run type-check`、`npm run lint` 无错误/警告。
- [ ] 环境变量（API 地址、密钥、开关）与部署目标匹配。
- [ ] `npm outdated`、`npm audit` 检查已完成，重大漏洞已处理。
- [ ] 需要时进行浏览器兼容、移动端适配、性能（Lighthouse）与安全扫描。

## 部署流程

### 环境配置
- `.env.development` / `.env.production` 管理 `VITE_*` 等变量，构建前写入。
- 示例：
  ```
  VITE_API_URL=https://api.example.com
  VITE_DEBUG=false
  ```
  代码通过 `import.meta.env.VITE_API_URL` 读取。

### 前端部署
1. `npm run build`
2. `scp -r dist/* user@server:/var/www/html/` 或 `rsync -avz dist/ ...`
3. Nginx 示例：
   ```nginx
   server {
     listen 80;
     server_name example.com;
     root /var/www/html;
     location / { try_files $uri $uri/ /index.html; }
     gzip on;
     gzip_types text/css application/javascript;
   }
   ```

### 后端部署
1. `npm install --production`
2. 使用 PM2 等守护进程：`pm2 start app.js --name myapp`
3. `pm2 logs myapp` 观察日志，`pm2 restart myapp` 滚动更新。

### Docker 流程
- 参照多阶段 Dockerfile：builder（安装依赖 + build）→ nginx 运行阶段。
- `docker build -t myapp:latest .`，`docker run -d -p 80:80 myapp:latest`。

### CI/CD 提示
- GitHub Actions 示范：checkout → setup Node → install → test → build → rsync 部署。
- 确保 secrets（如 `SERVER_USER`、`SERVER_HOST`）已配置。

## 常见问题

| 场景 | 诊断 | 处理 |
| --- | --- | --- |
| 构建 OOM | 观察日志含 `JavaScript heap out of memory` | 增加 `NODE_OPTIONS=--max-old-space-size=4096` |
| 依赖安装失败 | 版本冲突、缓存污染 | `rm -rf node_modules package-lock.json && npm install` |
| 部署后白屏 | 控制台报错/资源路径不对 | 检查 `vite.config.js` 的 `base`、API 可访问性 |
| 环境变量未生效 | 未以 `.env.production` 命名或缺少 `VITE_` 前缀 | 更正文件名后重新构建 |

## 输出要求

1. 先给出“通过/不通过”结论，随后按执行顺序列出命令及结果。
2. 失败时说明报错、影响范围和至少一个可执行的下一步（重试、修复、回滚等）。
3. 某项不适用时须说明原因与替代检查。
