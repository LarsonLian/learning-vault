# 10. Rules 系统（规则配置）

## 目录
- [10.1 什么是 Rules](#101-什么是-rules)
- [10.2 Rules vs Hooks 的区别](#102-rules-vs-hooks-的区别)
- [10.3 CLAUDE.md 配置](#103-claudemd-配置)
- [10.4 模块化规则目录](#104-模块化规则目录)
- [10.5 路径定向规则](#105-路径定向规则)
- [10.6 权限规则](#106-权限规则)
- [10.7 实际应用场景](#107-实际应用场景)

---

## 10.1 什么是 Rules

**Rules（规则）** 是 Claude Code 的配置系统，用于定义项目级别的指令、限制和最佳实践。

### 两种形式

```
1. CLAUDE.md
   ├─ 项目根目录的主要配置文件
   ├─ 包含代码风格、测试要求、安全策略等
   └─ 适合小型项目或简单配置

2. .claude/rules/ 目录
   ├─ 模块化规则系统
   ├─ 将大型 CLAUDE.md 分解为多个专注的文件
   └─ 适合大型项目或复杂配置
```

### 关键特点

```
特性对比:

┌─────────────────────────────────────────────────────────┐
│                    Rules 特性                            │
├─────────────────────────────────────────────────────────┤
│                                                          │
│ 执行方式:   LLM 解析和理解                               │
│ 强制性:     建议性（可在上下文压力下被覆盖）             │
│ 生效时机:   加载到 Claude 的上下文中                     │
│ 适用场景:   风格指导、最佳实践、开发规范                 │
│ 优势:       灵活、易于编写、自然语言                     │
│ 劣势:       非强制性、依赖 LLM 理解                      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 自动发现机制

Claude Code 会自动加载以下位置的规则：

1. 项目根目录的 `CLAUDE.md`
2. `.claude/rules/` 目录下的所有 `.md` 文件
3. 用户全局规则 `~/.claude/rules/`（如果存在）

---

## 10.2 Rules vs Hooks 的区别

### 对比表

| 特性 | Rules (CLAUDE.md) | Hooks |
|------|------------------|-------|
| **执行方式** | LLM 解析 | Shell 脚本 / LLM 执行 |
| **强制性** | 建议性（可被覆盖） | 强制性（返回 exit code 2 阻止） |
| **灵活性** | 高（自然语言） | 低（需要编写脚本） |
| **适用场景** | 风格指导、最佳实践 | 文件保护、工作流自动化 |
| **错误处理** | Claude 决定如何处理 | 明确的成功/失败状态 |
| **性能影响** | 占用上下文 token | 执行时间开销 |

### 实际例子对比

```bash
# 场景: 防止编辑 .env 文件

# 方法 1: 使用 Rules (CLAUDE.md)
## Security Guidelines
- Never edit .env files
- Use environment variables for secrets
- Keep credentials out of version control

# 结果: 被 LLM 解析 → Claude 理解这是建议 → 可能被忽略

---

# 方法 2: 使用 Hooks
{
  "hooks": {
    "PreToolUse": {
      "enabled": true,
      "rules": [
        {
          "matcher": "Edit|Write",
          "pattern": ".env*",
          "action": "block",
          "message": "🛑 禁止编辑 .env 文件"
        }
      ]
    }
  }
}

# 结果: 总是运行 → 返回 exit code 2 → 操作被强制阻止
```

### 何时使用 Rules？

```
✅ 推荐使用 Rules:

1. 代码风格指导
   - 命名约定
   - 格式化偏好
   - 注释风格

2. 最佳实践建议
   - 设计模式
   - 架构原则
   - 性能优化建议

3. 项目约定
   - 文件组织
   - 模块结构
   - 依赖管理

4. 开发流程
   - PR 审查清单
   - 测试策略
   - 部署流程
```

### 何时使用 Hooks？

```
✅ 推荐使用 Hooks:

1. 强制性保护
   - 防止删除关键文件
   - 阻止危险命令
   - 文件权限限制

2. 自动化工作流
   - 代码格式化
   - 自动测试
   - Git 提交流程

3. 质量检查
   - Lint 检查
   - 类型检查
   - 安全扫描

4. 审计和监控
   - 命令日志
   - 操作追踪
   - 性能监控
```

### 组合使用

```
最佳实践: Rules + Hooks 搭配使用

CLAUDE.md (规则):
## Security Guidelines
- Never commit secrets
- Use .env for configuration
- Validate all user input

.claude/hooks.json (强制):
{
  "PreFileWrite": {
    "rules": [
      {
        "pattern": "**/*",
        "command": "./hooks/secret-scanner.sh {file}",
        "failureAction": "block"
      }
    ]
  }
}

效果:
1. Rules 提供指导和上下文
2. Hooks 强制执行关键规则
3. 双重保护，互补增强
```

---

## 10.3 CLAUDE.md 配置

### 基础结构

```markdown
# Project Name

Brief description of the project and its purpose.

## Tech Stack

- Frontend: React 18 + TypeScript + Tailwind CSS
- Backend: Node.js 20 + Express
- Database: PostgreSQL 15
- Deployment: Docker + AWS ECS

## Code Style

### TypeScript
- Use strict mode
- Prefer `const` over `let`
- Use explicit types for function parameters
- Avoid `any` type

### React
- Use functional components
- Hooks over class components
- Props interface for all components
- Use TypeScript for type safety

### Testing
- Jest for unit tests
- React Testing Library for component tests
- Minimum 80% code coverage
- Test-driven development (TDD) for critical features

## Project Structure

\`\`\`
src/
├── api/           # API endpoints
├── components/    # React components
├── hooks/         # Custom hooks
├── lib/           # Utility functions
├── types/         # TypeScript types
└── __tests__/     # Test files
\`\`\`

## Development Workflow

1. Create feature branch from `main`
2. Write tests first (TDD)
3. Implement feature
4. Run linting and tests
5. Create pull request
6. Code review (minimum 1 approver)
7. Merge to `main`

## Naming Conventions

- Files: `kebab-case.ts`
- Components: `PascalCase.tsx`
- Functions: `camelCase`
- Constants: `UPPER_SNAKE_CASE`
- Interfaces: `IComponentProps`
- Types: `TDataType`

## Security

- Never commit .env files
- Use environment variables for secrets
- Validate all user input
- Sanitize database queries
- Implement rate limiting
- Use HTTPS in production

## Common Pitfalls

### Database Migrations
❌ Don't: Manually edit the database
✅ Do: Create migration files with `npm run migrate:create`

### API Routes
❌ Don't: Skip input validation
✅ Do: Use Zod schema validation

### State Management
❌ Don't: Prop drilling more than 2 levels
✅ Do: Use Context API or state management library

## Resources

- [Design System](https://www.figma.com/file/xxx)
- [API Documentation](https://api.example.com/docs)
- [Deployment Guide](./docs/deployment.md)
```

### 高级 CLAUDE.md 示例

```markdown
# E-Commerce Platform

Full-stack e-commerce platform with microservices architecture.

## Architecture Decisions (ADR)

### ADR-001: Microservices Architecture
**Date**: 2026-01-15
**Decision**: Use microservices instead of monolith
**Reasoning**:
- Independent scaling of services
- Technology flexibility per service
- Better fault isolation

**Consequences**:
- Increased complexity
- Need for service mesh
- Distributed tracing required

### ADR-002: PostgreSQL for User Data
**Date**: 2026-01-20
**Decision**: PostgreSQL for user service
**Reasoning**:
- ACID compliance for transactions
- Strong ecosystem
- JSON support for flexible data

**Alternatives Considered**:
- MongoDB: Rejected due to transaction requirements
- MySQL: Rejected due to inferior JSON support

## Service Guidelines

### User Service
- Port: 3001
- Database: PostgreSQL
- Auth: JWT with RS256
- Cache: Redis
- Testing: 90% coverage requirement

### Product Service
- Port: 3002
- Database: PostgreSQL
- Search: Elasticsearch
- Cache: Redis
- Testing: 85% coverage requirement

### Order Service
- Port: 3003
- Database: PostgreSQL
- Queue: RabbitMQ
- Payment: Stripe integration
- Testing: 95% coverage requirement

## Testing Strategy

### Unit Tests
- Test all business logic
- Mock external dependencies
- Target: 80% coverage

### Integration Tests
- Test API endpoints
- Use test database
- Clean state between tests

### E2E Tests
- Critical user flows only
- Use Playwright
- Run in CI/CD pipeline

## Deployment

### Staging
- Auto-deploy on merge to `develop`
- URL: https://staging.example.com
- Database: Staging DB (anonymized production data)

### Production
- Manual approval required
- Blue-green deployment
- Rollback plan required
- Database migrations run separately

## Monitoring and Alerting

### Metrics
- Error rate > 1% → Alert
- Response time > 500ms → Alert
- CPU > 80% → Warning
- Memory > 90% → Alert

### Logging
- Structured JSON logs
- Log levels: ERROR, WARN, INFO, DEBUG
- Send to CloudWatch

## On-Call Procedures

### Critical Issues
1. Acknowledge alert
2. Assess impact
3. Communicate to team
4. Investigate and fix
5. Post-mortem within 48h

### Runbooks
- [Database Connection Issues](./runbooks/db-connection.md)
- [High Error Rate](./runbooks/high-error-rate.md)
- [Service Downtime](./runbooks/service-downtime.md)
```

---

## 10.4 模块化规则目录

### 为什么需要模块化？

```
问题: CLAUDE.md 过大

单文件 CLAUDE.md (2000+ 行):
- 难以维护
- 合并冲突频繁
- 难以找到特定规则
- 上下文占用过多 token

解决方案: .claude/rules/ 目录

模块化规则:
- 每个文件专注一个主题
- 易于维护和更新
- 减少合并冲突
- 按需加载（未来特性）
```

### 推荐目录结构

```
.claude/rules/
├── README.md                    # 规则概览和索引
├── code-style.md                # 代码风格
├── testing.md                   # 测试规范
├── security.md                  # 安全策略
├── git-workflow.md              # Git 工作流
├── deployment.md                # 部署流程
├── frontend/
│   ├── react.md                 # React 规范
│   ├── styling.md               # CSS/样式规范
│   ├── state-management.md      # 状态管理
│   └── accessibility.md         # 可访问性
├── backend/
│   ├── api-design.md            # API 设计
│   ├── database.md              # 数据库规范
│   ├── authentication.md        # 认证授权
│   └── performance.md           # 性能优化
├── devops/
│   ├── docker.md                # Docker 配置
│   ├── kubernetes.md            # K8s 部署
│   └── monitoring.md            # 监控和告警
└── team/
    ├── code-review.md           # Code Review 清单
    ├── onboarding.md            # 新成员入职
    └── best-practices.md        # 团队最佳实践
```

### 模块化规则示例

#### code-style.md

```markdown
# Code Style Guidelines

## General Principles
- Readability over cleverness
- Consistency across the codebase
- Self-documenting code

## TypeScript Style

### Naming Conventions
```typescript
// ✅ Good
interface IUserProfile {
  userId: string;
  displayName: string;
}

const MAX_RETRY_COUNT = 3;
function getUserById(id: string): Promise<User> {}

// ❌ Bad
interface userprofile {
  user_id: string;
  DisplayName: string;
}

const maxretrycount = 3;
function get_user_by_id(id: string): Promise<User> {}
```

### Type Safety
- Use `unknown` instead of `any`
- Prefer union types over enums for simple cases
- Use `as const` for literal types

### Imports
```typescript
// ✅ Good - Organized imports
import React, { useState, useEffect } from 'react';
import { useRouter } from 'next/router';

import { Button } from '@/components/ui/button';
import { api } from '@/lib/api';

import type { User } from '@/types';

// ❌ Bad - Unorganized imports
import { api } from '@/lib/api';
import type { User } from '@/types';
import React, { useState, useEffect } from 'react';
import { Button } from '@/components/ui/button';
```

## Formatting

- Use Prettier (config: `.prettierrc`)
- Line length: 100 characters
- Indentation: 2 spaces
- Semicolons: Required
- Quotes: Single quotes
- Trailing commas: ES5
```

#### testing.md

```markdown
# Testing Guidelines

## Testing Philosophy

> Write tests that give you confidence, not just coverage.

## Test Structure

### Arrange-Act-Assert Pattern

```typescript
describe('UserService', () => {
  describe('createUser', () => {
    it('should create user with valid data', async () => {
      // Arrange
      const userData = {
        email: '[email protected]',
        password: 'SecurePass123!'
      };

      // Act
      const user = await userService.createUser(userData);

      // Assert
      expect(user).toBeDefined();
      expect(user.email).toBe(userData.email);
      expect(user.password).not.toBe(userData.password); // Should be hashed
    });
  });
});
```

## Coverage Requirements

| Component Type | Minimum Coverage |
|---------------|------------------|
| Business Logic | 90% |
| API Endpoints | 85% |
| UI Components | 70% |
| Utilities | 95% |

## Test Types

### Unit Tests
- Test individual functions/methods
- Mock all dependencies
- Fast execution (<1s per test)

### Integration Tests
- Test API endpoints
- Use test database
- Test real integrations

### E2E Tests
- Critical user flows only
- Login → Purchase → Checkout
- Use Playwright

## Test Naming

```typescript
// ✅ Good - Descriptive test names
it('should throw error when email is invalid', () => {});
it('should return 401 when token is expired', () => {});
it('should display error message when form submission fails', () => {});

// ❌ Bad - Vague test names
it('works', () => {});
it('test email', () => {});
it('handles error', () => {});
```

## Mocking Guidelines

```typescript
// ✅ Good - Mock only what you need
jest.mock('@/lib/api', () => ({
  fetchUser: jest.fn()
}));

// ❌ Bad - Overly complex mocks
jest.mock('@/lib/api', () => {
  const actualApi = jest.requireActual('@/lib/api');
  return {
    ...actualApi,
    // Lots of mock implementations
  };
});
```
```

#### security.md

```markdown
# Security Guidelines

## Security Principles

1. **Defense in Depth**: Multiple layers of security
2. **Least Privilege**: Minimum necessary permissions
3. **Zero Trust**: Verify everything
4. **Fail Secure**: Default to denying access

## Input Validation

### User Input
- Validate all user input
- Sanitize before processing
- Use parameterized queries

```typescript
// ✅ Good - Using Zod for validation
import { z } from 'zod';

const userSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/),
  age: z.number().int().min(13).max(120)
});

function createUser(data: unknown) {
  const validatedData = userSchema.parse(data); // Throws if invalid
  // ...
}

// ❌ Bad - No validation
function createUser(data: any) {
  // Directly using data without validation
  const user = await db.user.create({ data });
}
```

## Authentication & Authorization

### Password Hashing
```typescript
// ✅ Good - Using bcrypt with salt rounds
import bcrypt from 'bcryptjs';

const SALT_ROUNDS = 10;
const hashedPassword = await bcrypt.hash(password, SALT_ROUNDS);

// ❌ Bad - Plain text or weak hashing
const hashedPassword = md5(password); // Weak!
```

### JWT Best Practices
- Use RS256 (asymmetric) for production
- Short expiration (15-30 minutes)
- Implement refresh tokens
- Store secrets in environment variables

## SQL Injection Prevention

```typescript
// ✅ Good - Parameterized query
const user = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);

// ❌ Bad - String concatenation
const user = await db.query(
  `SELECT * FROM users WHERE email = '${email}'`
);
```

## XSS Prevention

```typescript
// ✅ Good - Using DOMPurify
import DOMPurify from 'dompurify';

const clean = DOMPurify.sanitize(userInput);
element.innerHTML = clean;

// ❌ Bad - Direct HTML injection
element.innerHTML = userInput; // Dangerous!
```

## Secrets Management

### Environment Variables
```bash
# ✅ Good - .env (not committed)
JWT_SECRET=random-generated-secret-key-here
DATABASE_URL=postgresql://user:pass@localhost:5432/db

# ❌ Bad - Hardcoded in code
const JWT_SECRET = 'my-secret-key'; // Never do this!
```

### .gitignore
```
.env
.env.local
.env.*.local
secrets/
*.key
*.pem
```

## Rate Limiting

```typescript
// ✅ Good - Implement rate limiting
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per windowMs
  message: 'Too many requests from this IP'
});

app.use('/api/', limiter);
```

## CORS Configuration

```typescript
// ✅ Good - Specific origins
app.use(cors({
  origin: ['https://example.com', 'https://app.example.com'],
  credentials: true
}));

// ❌ Bad - Allow all origins
app.use(cors({ origin: '*' })); // Dangerous in production!
```

## Security Checklist

Before deploying:
- [ ] All secrets in environment variables
- [ ] Input validation on all endpoints
- [ ] SQL injection prevention
- [ ] XSS protection
- [ ] CSRF tokens implemented
- [ ] Rate limiting enabled
- [ ] HTTPS enforced
- [ ] Security headers configured
- [ ] Dependencies updated (no known vulnerabilities)
- [ ] Authentication tested
- [ ] Authorization tested
```

---

## 10.5 路径定向规则

### 什么是路径定向规则？

**Path-Targeted Rules** 允许规则只在特定文件或目录上生效。

### 语法

```markdown
---
paths: pattern
---

# Rules content here
```

### Glob 模式

```bash
src/**/*.ts          # 所有 src 下的 TypeScript 文件
src/**/*.{ts,tsx}    # TypeScript 和 TSX 文件
tests/**/*           # tests 目录下的所有文件
src/api/**           # API 目录下的所有文件
!src/legacy/**       # 排除 legacy 目录
```

### 实际示例

#### React 组件规则

```markdown
---
paths: src/components/**/*.tsx
---

# React Component Standards

## Component Structure

Every component must follow this structure:

1. Imports
2. Type definitions
3. Component definition
4. Default export

## Example

\`\`\`typescript
// 1. Imports
import React from 'react';
import { useState } from 'react';

// 2. Type definitions
interface IButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
}

// 3. Component definition
export default function Button({ label, onClick, disabled = false }: IButtonProps) {
  return (
    <button onClick={onClick} disabled={disabled}>
      {label}
    </button>
  );
}
\`\`\`

## Required Elements

- TypeScript interface for props (prefix with 'I')
- JSDoc comment describing the component
- PropTypes validation (if not using TypeScript)
- Default export
- Functional component (no class components)

## Hooks Rules

- Always use hooks at the top level
- Custom hooks start with 'use'
- Use useMemo for expensive calculations
- Use useCallback for event handlers passed to children
```

#### API 路由规则

```markdown
---
paths: src/api/**/*.ts
---

# API Route Standards

## Structure

Every API route must include:

1. Input validation (Zod schema)
2. Error handling
3. Type-safe response
4. Logging

## Template

\`\`\`typescript
import { z } from 'zod';
import { NextApiRequest, NextApiResponse } from 'next';

// 1. Input validation
const requestSchema = z.object({
  userId: z.string().uuid(),
  action: z.enum(['create', 'update', 'delete'])
});

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    // 2. Validate input
    const validatedData = requestSchema.parse(req.body);

    // 3. Business logic
    const result = await performAction(validatedData);

    // 4. Type-safe response
    return res.status(200).json({
      success: true,
      data: result
    });

  } catch (error) {
    // 5. Error handling
    console.error('API Error:', error);

    if (error instanceof z.ZodError) {
      return res.status(400).json({
        success: false,
        error: 'Validation failed',
        details: error.errors
      });
    }

    return res.status(500).json({
      success: false,
      error: 'Internal server error'
    });
  }
}
\`\`\`

## Security Requirements

- Validate all inputs
- Use parameterized queries
- Implement rate limiting
- Return minimal error details to client
- Log all errors with context

## Response Format

All responses must follow this structure:

\`\`\`typescript
{
  success: boolean;
  data?: T;          // On success
  error?: string;    // On failure
  details?: any;     // Additional error details
}
\`\`\`
```

#### 测试文件规则

```markdown
---
paths: **/*.test.ts,**/*.test.tsx
---

# Test File Standards

## Naming Convention

- Unit tests: `component.test.tsx`
- Integration tests: `api.integration.test.ts`
- E2E tests: `flow.e2e.test.ts`

## Test Structure

Use the AAA (Arrange-Act-Assert) pattern:

\`\`\`typescript
describe('Feature Name', () => {
  describe('functionName', () => {
    it('should do something when condition', () => {
      // Arrange - Setup
      const input = 'test';

      // Act - Execute
      const result = functionName(input);

      // Assert - Verify
      expect(result).toBe('expected');
    });
  });
});
\`\`\`

## Coverage Requirements

- All business logic functions: 100%
- API endpoints: 85%
- UI components: 70%
- Utilities: 95%

## Test Isolation

- Use `beforeEach` to reset state
- Clean up after each test
- No test should depend on another
- Use factories for test data

## Mocking

- Mock external dependencies
- Use `jest.fn()` for function mocks
- Use `jest.spyOn()` for method mocks
- Clean up mocks in `afterEach`
```

#### 数据库迁移规则

```markdown
---
paths: migrations/**/*.ts
---

# Database Migration Standards

## Naming Convention

Format: `YYYYMMDDHHMMSS_description.ts`

Example: `20260208120000_add_user_profile_table.ts`

## Migration Structure

\`\`\`typescript
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  // Create table or modify schema
  await knex.schema.createTable('user_profiles', (table) => {
    table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    table.uuid('user_id').references('id').inTable('users').onDelete('CASCADE');
    table.string('display_name').notNullable();
    table.text('bio');
    table.timestamps(true, true); // created_at, updated_at
  });

  // Add indices
  await knex.schema.alterTable('user_profiles', (table) => {
    table.index('user_id');
  });
}

export async function down(knex: Knex): Promise<void> {
  // Reverse the changes
  await knex.schema.dropTableIfExists('user_profiles');
}
\`\`\`

## Best Practices

1. **Always include a rollback (down)**
   - Test the down migration
   - Ensure it reverses all changes

2. **Additive changes only**
   - Don't delete columns in production
   - Add new columns as nullable first
   - Deprecate old columns over time

3. **Data migrations**
   - Separate data migrations from schema migrations
   - Use transactions
   - Handle large datasets in batches

4. **Testing**
   - Test up and down migrations locally
   - Verify on staging before production
   - Keep backups before running

## Dangerous Operations

❌ **Never do these in production:**

\`\`\`typescript
// Dropping columns
table.dropColumn('old_column');

// Changing column types directly
table.specificType('column', 'new_type');

// Removing NOT NULL constraint on populated columns
table.dropNotNull('column');
\`\`\`

✅ **Instead, do this:**

\`\`\`typescript
// Step 1: Add new column
await knex.schema.alterTable('users', (table) => {
  table.string('new_column').nullable();
});

// Step 2: Migrate data (separate migration)
await knex('users').update({
  new_column: knex.raw('old_column::text')
});

// Step 3: Make NOT NULL (after data migration)
await knex.schema.alterTable('users', (table) => {
  table.string('new_column').notNullable().alter();
});

// Step 4: Drop old column (after verification)
await knex.schema.alterTable('users', (table) => {
  table.dropColumn('old_column');
});
\`\`\`
```

---

## 10.6 权限规则

### Bash 命令权限

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",           // 允许所有 npm scripts
      "Bash(npm test)",            // 允许运行测试
      "Bash(npm run build)",       // 允许构建
      "Bash(git status)",          // 允许查看 git 状态
      "Bash(git diff)",            // 允许查看差异
      "Bash(git log)"              // 允许查看日志
    ],

    "ask": [
      "Bash(npm install *)",       // 安装依赖需要确认
      "Bash(git push *)",          // 推送需要确认
      "Bash(git commit *)",        // 提交需要确认
      "Bash(docker *)"             // Docker 命令需要确认
    ],

    "deny": [
      "Bash(rm -rf *)",            // 禁止递归删除
      "Bash(sudo *)",              // 禁止 sudo 命令
      "Bash(curl http*)",          // 禁止不安全的 HTTP 请求
      "Bash(wget *)",              // 禁止 wget
      "Bash(chmod 777 *)",         // 禁止过度宽松的权限
      "Bash(git push --force *)",  // 禁止强制推送
      "Bash(dd if=*)",             // 禁止 dd 命令（危险）
      "Bash(:(){:|:&};:)"          // 禁止 fork bomb
    ]
  }
}
```

### 通配符语法详解

```bash
# 精确匹配
Bash(npm run build)          # 仅匹配 "npm run build"

# 前缀匹配
Bash(npm run *)              # 匹配 "npm run" 开头的任何命令
                             # ✓ npm run build
                             # ✓ npm run test
                             # ✗ npm install

# 命令通配
Bash(npm *)                  # 匹配 "npm" 开头的任何命令
                             # ✓ npm run build
                             # ✓ npm install
                             # ✓ npm test

# 命令+空格通配
Bash(ls *)                   # 匹配 "ls " (注意空格) 开头
                             # ✓ ls -la
                             # ✗ lsof (不是 "ls " 开头)

# 命令通配（无空格）
Bash(ls*)                    # 匹配 "ls" 开头
                             # ✓ ls -la
                             # ✓ lsof

# 多个通配符
Bash(git * origin *)         # 匹配 "git ... origin ..."
                             # ✓ git push origin main
                             # ✓ git pull origin develop
```

### 文件权限

```json
{
  "permissions": {
    "allow": [
      "Read",                      // 允许读取所有文件
      "Edit(src/**/*.ts)",         // 允许编辑 src 下的 TS 文件
      "Edit(src/**/*.tsx)",        // 允许编辑 TSX 文件
      "Edit(tests/**/*)",          // 允许编辑测试文件
      "Write(dist/**/*)",          // 允许写入构建目录
      "Write(.claude/**/*)"        // 允许写入 Claude 配置
    ],

    "ask": [
      "Edit(package.json)",        // 修改依赖需要确认
      "Edit(package-lock.json)",   // 修改 lock 文件需要确认
      "Edit(.github/**/*)",        // 修改 CI 配置需要确认
      "Edit(docker-compose.yml)"   // 修改 Docker 配置需要确认
    ],

    "deny": [
      "Edit(.env)",                // 禁止编辑环境变量
      "Edit(.env.*)",              // 禁止编辑所有 .env 文件
      "Edit(.git/**)",             // 禁止编辑 git 配置
      "Edit(secrets/**)",          // 禁止编辑 secrets 目录
      "Write(../**)",              // 禁止在上级目录写文件
      "Write(///**)"               // 禁止写入根目录
    ]
  }
}
```

### 路径模式详解

```bash
# 相对路径（相对于配置文件）
Edit(src/**/*.ts)            # src 目录下的所有 TS 文件
Edit(*.json)                 # 当前目录的 JSON 文件
Edit(docs/*.md)              # docs 目录的 MD 文件

# 递归匹配
Edit(src/**/*.ts)            # src 下所有子目录的 TS 文件
Edit(**/test.ts)             # 任何位置的 test.ts

# 单层匹配
Edit(src/*.ts)               # 仅 src 直接子文件
                             # ✓ src/index.ts
                             # ✗ src/utils/helper.ts

# 多扩展名
Edit(**/*.{ts,tsx,js,jsx})   # 所有 JS/TS 文件

# 排除模式
Edit(!node_modules/**)       # 排除 node_modules
Edit(src/**,!src/legacy/**)  # src 下除了 legacy

# 绝对路径
Edit(///etc/config.json)     # 绝对路径（3个斜杠）
Edit(///var/log/*.log)       # 系统日志文件

# 上级目录
Edit(../**)                  # 上级目录的所有文件
                             # 通常应该 deny
```

### 工具权限

```json
{
  "permissions": {
    "allow": [
      "Read",                      // 允许读取文件
      "Grep",                      // 允许搜索
      "Glob",                      // 允许文件匹配
      "WebSearch",                 // 允许网络搜索
      "WebFetch"                   // 允许获取网页
    ],

    "ask": [
      "Edit",                      // 编辑需要确认
      "Write",                     // 写入需要确认
      "Bash"                       // Bash 需要确认
    ],

    "deny": [
      // 没有完全禁止的工具
      // 可以通过 Bash/Edit/Write 的 pattern 限制
    ]
  }
}
```

---

## 10.7 实际应用场景

### 场景 1: 开源项目

```markdown
# Open Source Project

## Contribution Guidelines

### Code of Conduct
- Be respectful
- Constructive feedback only
- Help newcomers

### Before Contributing

1. Read CONTRIBUTING.md
2. Check existing issues
3. Fork the repository
4. Create feature branch
5. Write tests
6. Submit PR

### PR Requirements

- [ ] Passes all CI checks
- [ ] Includes tests
- [ ] Updates documentation
- [ ] Follows code style
- [ ] No merge conflicts
- [ ] Signed commits

### Code Style

Follow project conventions:
- ESLint configuration
- Prettier formatting
- TypeScript strict mode

### Commit Messages

Format: `type(scope): description`

Types: feat, fix, docs, style, refactor, test, chore

Example:
\`\`\`
feat(auth): add OAuth 2.0 login
fix(api): handle null responses
docs(readme): update installation steps
\`\`\`
```

### 场景 2: 企业项目

```markdown
# Enterprise Application

## Compliance Requirements

### Data Protection
- All PII must be encrypted
- Data retention: 90 days
- GDPR compliance required
- SOC 2 Type II controls

### Security Standards
- OWASP Top 10 prevention
- Quarterly security audits
- Penetration testing annually
- Vulnerability scanning weekly

### Access Control
- Role-based access control (RBAC)
- Principle of least privilege
- MFA for production access
- Audit all privileged actions

## Development Standards

### Code Review
- Mandatory for all changes
- Minimum 2 approvers
- Security review for auth changes
- Architecture review for major changes

### Testing
- Unit test coverage > 80%
- Integration tests required
- Security testing in CI/CD
- Performance testing for critical paths

### Deployment

- Production changes require change request
- Deployment window: Tuesdays 2-4 AM EST
- Rollback plan required
- Monitoring dashboard ready

## Documentation

- API documentation in OpenAPI format
- Architecture diagrams up to date
- Runbooks for all services
- Incident post-mortems required
```

### 场景 3: 教育项目

```markdown
# Student Project

## Learning Objectives

1. Understand React fundamentals
2. Practice TypeScript
3. Learn state management
4. Implement testing

## Project Requirements

### Functional Requirements
- User registration and login
- CRUD operations for posts
- Real-time comments
- User profiles

### Technical Requirements
- React 18+
- TypeScript strict mode
- React Router for navigation
- Context API for state
- Jest + RTL for testing

## Coding Standards

### Keep It Simple
- Start with basic implementation
- Refactor after it works
- Don't over-engineer

### Learn by Doing
- Try to implement yourself first
- Research when stuck
- Ask for help when needed

### Documentation
- Comment complex logic
- Write README with setup instructions
- Document API endpoints

## Getting Help

1. Read the error message carefully
2. Search for the error online
3. Check official documentation
4. Ask in class discussion
5. Office hours: Thursdays 3-5 PM
```

### 场景 4: 微服务架构

```markdown
# Microservices Architecture

## Service Communication

### Synchronous (REST API)
- Use for user-facing operations
- Implement circuit breakers
- Timeout: 5 seconds max
- Retry: 3 times with exponential backoff

### Asynchronous (Message Queue)
- Use for background jobs
- RabbitMQ for events
- Idempotent message handlers
- Dead letter queue for failures

## Data Management

### Database per Service
- Each service owns its data
- No direct database access between services
- Use events for data synchronization
- Eventual consistency acceptable

### Shared Data
- Read replicas for queries
- Event sourcing for audit trail
- CQRS for complex queries

## Deployment

### Container Strategy
- Docker for all services
- Docker Compose for local development
- Kubernetes for production
- Helm charts for configuration

### Service Mesh
- Istio for traffic management
- mTLS for service-to-service encryption
- Distributed tracing with Jaeger
- Metrics with Prometheus

## Monitoring

### Health Checks
- Liveness probe: /health/live
- Readiness probe: /health/ready
- Response time < 100ms

### Logging
- Structured JSON logs
- Correlation ID for request tracing
- Log level: INFO in production
- Centralized logging with ELK stack

### Metrics
- Request rate
- Error rate
- Request duration (p50, p95, p99)
- Service dependencies
```

---

## 总结

### Rules 的核心价值

1. **团队协作** - 统一代码风格和开发规范
2. **知识传递** - 新成员快速了解项目约定
3. **质量保证** - 建立和维护代码质量标准
4. **效率提升** - 减少重复说明和讨论

### 最佳实践回顾

✅ **推荐做法**:
- 保持 CLAUDE.md 精简（< 300 行）
- 使用 .claude/rules/ 模块化大型配置
- 使用路径定向规则针对特定文件类型
- Rules + Hooks 搭配使用
- 定期审查和更新规则

❌ **避免**:
- 过度详细的规则（导致 token 浪费）
- 矛盾的规则
- 过时的规则
- 依赖 Rules 实现强制性约束（应使用 Hooks）

---

## 下一步

- 学习 [Plugins 系统](./11-plugins-system.md)，了解如何打包和分享配置
- 查看 [Skills 系统](./05-advanced-features.md#54-skills-系统)，掌握可重用任务模块
- 参考 [最佳实践](./08-best-practices.md)，优化规则配置

---

**参考资源**:
- [A Complete Guide to CLAUDE.md](https://www.builder.io/blog/claude-md-guide)
- [Rules Directory Documentation](https://claudefa.st/blog/guide/mechanics/rules-directory)
- [What is .claude/rules/](https://claudelog.com/faqs/what-are-claude-rules/)
- [Permissions Configuration Guide](https://www.eesel.ai/blog/claude-code-permissions)
