# 11. Plugins 系统（插件扩展）

## 目录
- [11.1 什么是 Plugins](#111-什么是-plugins)
- [11.2 插件架构和组件](#112-插件架构和组件)
- [11.3 安装和管理插件](#113-安装和管理插件)
- [11.4 官方插件列表](#114-官方插件列表)
- [11.5 自定义插件开发](#115-自定义插件开发)
- [11.6 插件配置和权限](#116-插件配置和权限)
- [11.7 发布和分享插件](#117-发布和分享插件)

---

## 11.1 什么是 Plugins

### 插件定义

**Plugins（插件）** 是扩展 Claude Code 功能的可分发包，封装了一组相关的配置和功能。

### 插件的组成部分

```
插件可以包含:

┌─────────────────────────────────────────────────────────┐
│                   Plugin Components                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│ 1. Slash Commands (斜杠命令)                             │
│    └─ 自定义快捷操作，如 /deploy, /test                  │
│                                                          │
│ 2. Subagents (子代理)                                    │
│    └─ 专业化的 AI 助手，如代码审查员、架构师             │
│                                                          │
│ 3. MCP Servers                                           │
│    └─ 连接外部工具和数据源                               │
│                                                          │
│ 4. Hooks (钩子)                                          │
│    └─ 自动化工作流和验证                                 │
│                                                          │
│ 5. Skills (技能)                                         │
│    └─ 可重用的任务模块                                   │
│                                                          │
│ 6. Rules (规则)                                          │
│    └─ 代码风格和项目约定                                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 插件的优势

```
为什么使用插件?

✅ 一次安装，多处使用
   └─ 终端和 VS Code 中都能使用
   └─ 所有项目共享配置

✅ 团队协作
   └─ 共享团队工作流
   └─ 统一开发规范

✅ 版本管理
   └─ 语义化版本
   └─ 轻松更新和回滚

✅ 模块化
   └─ 专注单一功能域
   └─ 易于维护和测试

✅ 分发和发现
   └─ 官方 marketplace
   └─ GitHub 安装
   └─ 社区共享
```

---

## 11.2 插件架构和组件

### 标准插件结构

```
my-plugin/
├── .claude-plugin/
│   ├── plugin.json              # 插件元数据 (必需)
│   └── marketplace.json         # marketplace 配置 (可选)
│
├── commands/                     # 斜杠命令 (可选)
│   ├── feature-dev.md
│   ├── code-review.md
│   └── deploy.md
│
├── agents/                       # 子代理 (可选)
│   ├── code-reviewer.md
│   ├── architect.md
│   └── security-auditor.md
│
├── skills/                       # 技能 (可选)
│   ├── python-formatter/
│   │   ├── SKILL.md
│   │   └── scripts/
│   ├── git-workflow/
│   │   └── SKILL.md
│   └── api-generator/
│       └── SKILL.md
│
├── mcp-servers/                  # MCP 服务器配置 (可选)
│   ├── github.json
│   ├── slack.json
│   └── custom-api.json
│
├── hooks/                        # 钩子 (可选)
│   ├── hooks.json
│   └── scripts/
│       ├── pre-commit.sh
│       └── post-build.sh
│
├── rules/                        # 规则 (可选)
│   ├── code-style.md
│   ├── security.md
│   └── testing.md
│
├── scripts/                      # 辅助脚本 (可选)
│   ├── install.sh
│   └── uninstall.sh
│
├── docs/                         # 文档 (推荐)
│   ├── getting-started.md
│   ├── usage.md
│   └── api.md
│
├── tests/                        # 测试 (推荐)
│   └── plugin.test.js
│
├── README.md                     # 项目说明 (必需)
├── LICENSE                       # 许可证 (推荐)
└── CHANGELOG.md                  # 变更日志 (推荐)
```

### plugin.json 配置

```json
{
  // 基本信息
  "name": "my-development-toolkit",
  "version": "1.2.0",
  "description": "Complete development toolkit with code review, testing, and deployment workflows",

  // 作者信息
  "author": {
    "name": "Your Name",
    "email": "[email protected]",
    "url": "https://github.com/yourname"
  },

  // 仓库和许可
  "repository": {
    "type": "git",
    "url": "https://github.com/yourname/my-development-toolkit"
  },
  "license": "MIT",

  // 分类和搜索
  "keywords": [
    "code-review",
    "testing",
    "deployment",
    "typescript",
    "security",
    "ci-cd"
  ],
  "categories": ["development", "testing", "deployment"],

  // 依赖
  "dependencies": {
    "other-plugin": "^1.0.0"
  },
  "peerDependencies": {
    "claude-code": ">=2.0.0"
  },

  // 组件定义
  "commands": {
    "feature-dev": "commands/feature-dev.md",
    "code-review": "commands/code-review.md",
    "deploy": "commands/deploy.md"
  },

  "agents": {
    "code-reviewer": "agents/code-reviewer.md",
    "architect": "agents/architect.md",
    "security-auditor": "agents/security-auditor.md"
  },

  "skills": [
    "skills/python-formatter",
    "skills/git-workflow",
    "skills/api-generator"
  ],

  "mcpServers": {
    "github": "mcp-servers/github.json",
    "slack": "mcp-servers/slack.json"
  },

  "hooks": "hooks/hooks.json",

  "rules": [
    "rules/code-style.md",
    "rules/security.md",
    "rules/testing.md"
  ],

  // 生命周期钩子
  "scripts": {
    "install": "bash scripts/install.sh",
    "uninstall": "bash scripts/uninstall.sh",
    "update": "bash scripts/update.sh"
  },

  // 配置选项
  "config": {
    "defaultModel": "claude-opus-4-5",
    "maxTurns": 30,
    "timeout": 300000
  },

  // 引擎要求
  "engines": {
    "claude-code": ">=2.0.0",
    "node": ">=18.0.0"
  }
}
```

### marketplace.json 配置

```json
{
  // 显示信息
  "displayName": "My Development Toolkit",
  "description": "Complete toolkit for enterprise development workflows including code review, testing, and deployment automation",

  // 分类
  "categories": [
    "development",
    "testing",
    "security",
    "deployment"
  ],

  // 链接
  "homepage": "https://github.com/yourname/my-development-toolkit",
  "issues": "https://github.com/yourname/my-development-toolkit/issues",
  "documentation": "https://github.com/yourname/my-development-toolkit/wiki",

  // 媒体
  "icon": "icon.png",
  "screenshots": [
    "screenshots/code-review.png",
    "screenshots/deployment.png"
  ],
  "video": "https://www.youtube.com/watch?v=xxxxx",

  // Marketplace 元数据
  "publisher": "yourname",
  "pricing": "free",  // free | paid | trial
  "verified": false,

  // 统计
  "stats": {
    "downloads": 1250,
    "rating": 4.8,
    "reviews": 42
  },

  // 标签
  "tags": [
    "code-quality",
    "automation",
    "ci-cd",
    "enterprise"
  ]
}
```

---

## 11.3 安装和管理插件

### 安装插件

```bash
# 方法 1: 从官方 marketplace 安装
/plugin install commit-commands

输出:
"安装插件: commit-commands
 版本: 1.0.0
 来源: Official Anthropic Plugins

 已安装组件:
 - 3 个斜杠命令
 - 2 个技能
 - 1 个钩子配置

 ✓ 安装成功

 可用命令:
 /commit-message
 /commit-best-practices
 /commit-format"

# 方法 2: 从特定 marketplace 安装
/plugin install ultrathink@cc-marketplace

# 方法 3: 从 GitHub 安装（完整 URL）
/plugin install https://github.com/yourname/my-plugin

# 方法 4: 从本地目录安装（开发模式）
/plugin install file:///path/to/plugin

# 方法 5: 指定版本安装
/plugin install commit-commands@1.2.0
```

### 列出已安装插件

```bash
/plugin list

输出:
┌────────────────────────────────────────────────────────┐
│ Installed Plugins                                      │
├────────────────────────────────────────────────────────┤
│                                                        │
│ commit-commands (v1.0.0)                               │
│ ├─ Author: Anthropic                                   │
│ ├─ Description: Git commit workflow automation        │
│ ├─ Commands: /commit-message, /commit-best-practices  │
│ └─ Status: ✓ Active                                   │
│                                                        │
│ code-review-toolkit (v2.1.0)                           │
│ ├─ Author: Community                                   │
│ ├─ Description: Code review automation                │
│ ├─ Commands: /code-review, /pr-review                 │
│ ├─ Agents: code-reviewer, security-auditor            │
│ └─ Status: ✓ Active                                   │
│                                                        │
│ my-plugin (v0.1.0) [Local]                             │
│ ├─ Path: /path/to/local/plugin                        │
│ ├─ Description: Development version                    │
│ └─ Status: ✓ Active (Dev Mode)                        │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 更新插件

```bash
# 更新单个插件
/plugin update commit-commands

输出:
"检查更新: commit-commands
 当前版本: 1.0.0
 最新版本: 1.2.0

 更新日志:
 - 添加了新的提交模板
 - 修复了 emoji 支持
 - 性能改进

 更新? [Y/n]"

# 更新所有插件
/plugin update-all

输出:
"检查所有插件更新...

 可更新插件:
 1. commit-commands: 1.0.0 → 1.2.0
 2. code-review-toolkit: 2.0.0 → 2.1.0

 更新所有? [Y/n]"
```

### 卸载插件

```bash
# 卸载插件
/plugin uninstall commit-commands

输出:
"卸载插件: commit-commands

 将移除:
 - 3 个斜杠命令
 - 2 个技能
 - 1 个钩子配置

 ⚠️  这将删除所有插件数据
 确认卸载? [y/N]"
```

### 启用/禁用插件

```bash
# 禁用插件（保留数据）
/plugin disable commit-commands

输出:
"✓ 已禁用插件: commit-commands
 命令和功能已暂时不可用
 使用 /plugin enable commit-commands 重新启用"

# 启用插件
/plugin enable commit-commands

输出:
"✓ 已启用插件: commit-commands
 所有功能现已可用"
```

### 查看插件详情

```bash
/plugin info commit-commands

输出:
┌────────────────────────────────────────────────────────┐
│ Plugin: commit-commands                                │
├────────────────────────────────────────────────────────┤
│                                                        │
│ Version: 1.2.0                                         │
│ Author: Anthropic                                      │
│ License: MIT                                           │
│ Homepage: https://github.com/anthropics/commit-cmds    │
│                                                        │
│ Description:                                           │
│ Automated Git commit message generation and            │
│ best practices enforcement                             │
│                                                        │
│ Components:                                            │
│ ├─ Commands (3)                                        │
│ │  ├─ /commit-message                                  │
│ │  ├─ /commit-best-practices                           │
│ │  └─ /commit-format                                   │
│ │                                                      │
│ ├─ Skills (2)                                          │
│ │  ├─ conventional-commits                             │
│ │  └─ semantic-versioning                              │
│ │                                                      │
│ └─ Hooks (1)                                           │
│    └─ PreCommit validation                             │
│                                                        │
│ Dependencies:                                          │
│ └─ None                                                │
│                                                        │
│ Statistics:                                            │
│ ├─ Downloads: 5,230                                    │
│ ├─ Rating: ⭐⭐⭐⭐⭐ (4.9/5.0)                       │
│ └─ Last Updated: 2026-01-15                            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## 11.4 官方插件列表

### Anthropic 官方插件

来自 `anthropics/claude-plugins-official` 仓库：

#### 1. Agent SDK Plugin

```
名称: agent-sdk-plugin
命令: /new-sdk-app

功能:
- 创建新的 Claude Agent SDK 应用
- 自动设置项目结构
- 配置开发环境

使用示例:
/new-sdk-app my-agent
```

#### 2. Code Review Plugin

```
名称: code-review-plugin
命令: /code-review

功能:
- 5 个 Sonnet 代理并行审查代码
- 全面的质量检查
- 安全漏洞扫描
- 性能优化建议

使用示例:
/code-review src/
/code-review --strict
```

#### 3. Feature Development Plugin

```
名称: feature-dev-plugin
命令: /feature-dev

功能:
- 完整的功能开发工作流
- TDD 方法论
- 自动生成测试
- 代码审查集成

使用示例:
/feature-dev "用户个人资料页面"
```

#### 4. Plugin Development Toolkit

```
名称: plugin-dev-toolkit
命令:
- /plugin-dev:create-plugin
- /plugin-dev:add-command
- /plugin-dev:add-agent

功能:
- 8 阶段引导式插件创建
- 模板和脚手架
- 最佳实践指导

使用示例:
/plugin-dev:create-plugin my-toolkit
```

#### 5. PR Review Toolkit

```
名称: pr-review-toolkit
命令: /pr-review

功能:
- GitHub PR 审查自动化
- 代码差异分析
- 审查意见生成
- 自动化检查清单

使用示例:
/pr-review #123
```

#### 6. Claude Opus 4.5 Migration

```
名称: opus-migration
命令: /migrate-to-opus

功能:
- 自动化模型迁移
- API 调用更新
- 性能优化建议

使用示例:
/migrate-to-opus
```

#### 7. Commit Commands

```
名称: commit-commands
命令:
- /commit-message
- /commit-best-practices
- /commit-format

功能:
- Git 工作流自动化
- 提交消息生成
- Conventional Commits 支持

使用示例:
/commit-message
/commit-format --emoji
```

### 社区热门插件

```
以下是社区创建的优秀插件:

1. Python Development Toolkit
   - Python 项目脚手架
   - 虚拟环境管理
   - Poetry/pip 集成

2. React Component Generator
   - React 组件模板
   - TypeScript 类型生成
   - Storybook 集成

3. Docker & Kubernetes Helper
   - Dockerfile 生成
   - K8s manifest 创建
   - 容器调试工具

4. API Documentation Generator
   - OpenAPI/Swagger 生成
   - API 测试用例
   - Postman collection 导出

5. Database Migration Manager
   - 数据库迁移生成
   - Schema 版本控制
   - 回滚支持
```

---

## 11.5 自定义插件开发

### 创建插件的基本步骤

```bash
# 1. 创建插件目录结构
mkdir my-plugin
cd my-plugin

# 2. 初始化插件
mkdir -p .claude-plugin commands agents skills

# 3. 创建 plugin.json
cat > .claude-plugin/plugin.json << 'EOF'
{
  "name": "my-plugin",
  "version": "0.1.0",
  "description": "My custom Claude Code plugin",
  "author": {
    "name": "Your Name",
    "email": "[email protected]"
  },
  "license": "MIT"
}
EOF

# 4. 创建 README
cat > README.md << 'EOF'
# My Plugin

Description of what your plugin does.

## Installation

\`\`\`bash
/plugin install https://github.com/yourname/my-plugin
\`\`\`

## Usage

...
EOF
```

### 示例: 创建一个简单的插件

#### 插件功能: 数据库工具包

```
功能列表:
1. /db-migrate - 生成数据库迁移
2. /db-seed - 生成种子数据
3. database-schema agent - 数据库架构设计代理
```

#### 目录结构

```
database-toolkit/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── db-migrate.md
│   └── db-seed.md
├── agents/
│   └── database-schema.md
├── skills/
│   └── migration-generator/
│       └── SKILL.md
└── README.md
```

#### plugin.json

```json
{
  "name": "database-toolkit",
  "version": "1.0.0",
  "description": "Database development tools including migrations, seeding, and schema design",
  "author": {
    "name": "Your Name",
    "email": "[email protected]"
  },
  "repository": "https://github.com/yourname/database-toolkit",
  "license": "MIT",
  "keywords": ["database", "migration", "orm", "schema"],

  "commands": {
    "db-migrate": "commands/db-migrate.md",
    "db-seed": "commands/db-seed.md"
  },

  "agents": {
    "database-schema": "agents/database-schema.md"
  },

  "skills": [
    "skills/migration-generator"
  ]
}
```

#### commands/db-migrate.md

```markdown
---
name: db-migrate
description: Generate database migration files
---

# Database Migration Command

When invoked, create a new database migration file.

## Process

1. Ask user for migration description
2. Detect ORM/migration tool:
   - Knex.js
   - Prisma
   - TypeORM
   - Sequelize

3. Generate migration file based on detected tool

4. Create both `up` and `down` migrations

5. Add timestamp to filename

## Template (Knex.js)

\`\`\`typescript
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  await knex.schema.createTable('table_name', (table) => {
    table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    table.timestamps(true, true);
  });
}

export async function down(knex: Knex): Promise<void> {
  await knex.schema.dropTableIfExists('table_name');
}
\`\`\`

## After Creation

- Run `npm run migrate:latest` to apply
- Commit the migration file
```

#### agents/database-schema.md

```markdown
---
name: database-schema
description: Expert database architect for schema design and optimization
tools: Read, Grep, Glob, WebSearch
model: claude-opus-4-5
permissionMode: auto
---

# Database Schema Agent

You are an expert database architect with deep knowledge of:
- Relational database design
- Normalization (1NF, 2NF, 3NF, BCNF)
- Index optimization
- Query performance
- Data integrity
- Migration strategies

## Your Responsibilities

1. **Schema Design**
   - Analyze requirements
   - Design normalized schemas
   - Define relationships (1:1, 1:N, N:M)
   - Choose appropriate data types

2. **Performance Optimization**
   - Index strategy
   - Denormalization when beneficial
   - Query optimization
   - Partitioning strategies

3. **Data Integrity**
   - Foreign key constraints
   - Check constraints
   - Unique constraints
   - Default values

4. **Migration Planning**
   - Zero-downtime migrations
   - Backward compatibility
   - Data migration strategies

## Output Format

Provide:
1. ER diagram (Mermaid syntax)
2. CREATE TABLE statements
3. Index definitions
4. Migration plan
5. Performance considerations
```

#### skills/migration-generator/SKILL.md

```markdown
---
name: migration-generator
description: Generate database migrations for various ORMs
requiredTools: [Read, Write]
tags: [database, migration, orm]
---

# Migration Generator Skill

Automatically generate database migration files for various ORMs.

## Supported ORMs

- Knex.js
- Prisma
- TypeORM
- Sequelize
- Drizzle ORM

## Usage

When creating a migration:

1. Detect the ORM from package.json
2. Ask for migration description
3. Generate appropriate migration file
4. Follow ORM's naming convention

## Templates

### Knex.js
\`\`\`typescript
export async function up(knex: Knex): Promise<void> {
  // Migration up
}

export async function down(knex: Knex): Promise<void> {
  // Migration down
}
\`\`\`

### Prisma
\`\`\`sql
-- CreateTable
CREATE TABLE "User" (
    "id" TEXT NOT NULL,
    "email" TEXT NOT NULL,
    PRIMARY KEY ("id")
);
\`\`\`

### TypeORM
\`\`\`typescript
import { MigrationInterface, QueryRunner } from "typeorm";

export class Migration1234567890 implements MigrationInterface {
    public async up(queryRunner: QueryRunner): Promise<void> {
        // Migration up
    }

    public async down(queryRunner: QueryRunner): Promise<void> {
        // Migration down
    }
}
\`\`\`
```

---

## 11.6 插件配置和权限

### 用户级插件配置

```json
// ~/.claude/config.json
{
  "plugins": {
    "database-toolkit": {
      "enabled": true,
      "permissions": {
        "allow": [
          "Read",
          "Write(migrations/**)",
          "Bash(npm run migrate*)"
        ],
        "deny": [
          "Bash(npm run migrate:rollback)"
        ]
      },
      "settings": {
        "defaultORM": "knex",
        "migrationsPath": "migrations/",
        "model": "claude-opus-4-5"
      }
    },

    "code-review-toolkit": {
      "enabled": true,
      "permissions": {
        "allow": ["Read", "Grep", "Glob"],
        "ask": ["Edit", "Write"]
      },
      "settings": {
        "strictMode": true,
        "autoFix": false,
        "maxAgents": 5
      }
    }
  }
}
```

### 项目级插件配置

```json
// .claude/config.json
{
  "plugins": {
    "database-toolkit": {
      "settings": {
        "migrationsPath": "src/db/migrations/",
        "seedsPath": "src/db/seeds/"
      }
    }
  }
}
```

### 插件权限最佳实践

```json
{
  "plugins": {
    "my-plugin": {
      // 推荐: 明确权限
      "permissions": {
        "allow": [
          "Read",                      // 读取文件
          "Bash(npm run test)",        // 特定命令
          "Write(src/generated/**)"    // 特定目录
        ],
        "ask": [
          "Edit(src/**)",              // 需要确认的操作
          "Bash(npm install *)"
        ],
        "deny": [
          "Bash(rm -rf *)",            // 禁止的操作
          "Edit(.env*)"
        ]
      },

      // 避免: 过度权限
      // "permissions": "all"  // ❌ 不安全
    }
  }
}
```

---

## 11.7 发布和分享插件

### 发布到 GitHub

```bash
# 1. 创建 GitHub 仓库
gh repo create my-plugin --public

# 2. 推送代码
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/yourname/my-plugin.git
git push -u origin main

# 3. 创建 release
git tag v1.0.0
git push origin v1.0.0

gh release create v1.0.0 \
  --title "v1.0.0" \
  --notes "Initial release"
```

### 发布到 NPM（可选）

```json
// package.json
{
  "name": "@yourname/claude-plugin-my-plugin",
  "version": "1.0.0",
  "description": "My Claude Code plugin",
  "main": ".claude-plugin/plugin.json",
  "files": [
    ".claude-plugin/",
    "commands/",
    "agents/",
    "skills/",
    "README.md"
  ],
  "keywords": [
    "claude-code",
    "claude-plugin",
    "your-keywords"
  ]
}
```

```bash
# 发布到 NPM
npm login
npm publish --access public
```

### 提交到官方 Marketplace

```
步骤:

1. Fork anthropics/claude-plugins-community
2. 添加你的插件到 plugins/ 目录
3. 更新 plugins/index.json
4. 创建 Pull Request
5. 等待审核

PR 模板:
---
Plugin Name: My Plugin
Version: 1.0.0
Category: Development
Description: Brief description
Testing: How you tested it
---
```

### README 模板

```markdown
# My Plugin

Brief description of what your plugin does.

## Features

- Feature 1
- Feature 2
- Feature 3

## Installation

\`\`\`bash
/plugin install https://github.com/yourname/my-plugin
\`\`\`

Or via NPM:

\`\`\`bash
npm install -g @yourname/claude-plugin-my-plugin
claude plugin link @yourname/claude-plugin-my-plugin
\`\`\`

## Usage

### Commands

#### /command-name

Description of what this command does.

\`\`\`bash
/command-name [options]
\`\`\`

**Options:**
- `--option1` - Description
- `--option2` - Description

**Example:**
\`\`\`bash
/command-name --option1 value
\`\`\`

### Agents

#### agent-name

Description of what this agent does.

**How to use:**
\`\`\`
Ask Claude to use the agent-name agent for...
\`\`\`

### Skills

#### skill-name

Description of what this skill does.

## Configuration

\`\`\`json
{
  "plugins": {
    "my-plugin": {
      "enabled": true,
      "settings": {
        "option1": "value1",
        "option2": "value2"
      }
    }
  }
}
\`\`\`

## Examples

### Example 1: Basic Usage

\`\`\`bash
/command-name
\`\`\`

### Example 2: Advanced Usage

\`\`\`bash
/command-name --option value
\`\`\`

## Permissions

This plugin requires the following permissions:

- `Read` - To read project files
- `Write(generated/**)` - To write generated files
- `Bash(npm run build)` - To run build command

## Requirements

- Claude Code >= 2.0.0
- Node.js >= 18.0.0

## Development

\`\`\`bash
# Clone the repository
git clone https://github.com/yourname/my-plugin.git
cd my-plugin

# Install dependencies
npm install

# Link for local development
/plugin install file://$(pwd)

# Run tests
npm test
\`\`\`

## Contributing

Contributions are welcome! Please read CONTRIBUTING.md for details.

## License

MIT License - see LICENSE file for details

## Support

- Issues: https://github.com/yourname/my-plugin/issues
- Discussions: https://github.com/yourname/my-plugin/discussions

## Changelog

See CHANGELOG.md for version history.
```

---

## 总结

### Plugins 的核心价值

1. **可重用性** - 一次开发，多处使用
2. **可分享性** - 轻松与团队和社区分享
3. **模块化** - 专注单一功能域
4. **版本控制** - 清晰的版本管理和更新
5. **生态系统** - 构建丰富的工具生态

### 最佳实践回顾

✅ **开发插件时**:
- 单一职责原则
- 清晰的文档和示例
- 遵循语义化版本
- 包含测试
- 明确权限要求

✅ **使用插件时**:
- 审查权限要求
- 阅读文档和示例
- 及时更新
- 反馈问题和建议

❌ **避免**:
- 过度权限请求
- 缺乏文档
- 不兼容的版本更新
- 缺少测试

---

## 下一步

- 浏览 [官方插件仓库](https://github.com/anthropics/claude-plugins-official)
- 学习 [Skills 系统](./05-advanced-features.md#54-skills-系统)
- 参考 [最佳实践](./08-best-practices.md)

---

**参考资源**:
- [Create Plugins - Claude Code Docs](https://code.claude.com/docs/en/plugins)
- [Official Claude Plugins](https://github.com/anthropics/claude-plugins-official)
- [Community Plugins](https://github.com/anthropics/claude-plugins-community)
- [Plugin Development Guide](https://code.claude.com/docs/en/plugin-development)
