# 07. MCP 集成（Model Context Protocol）

## 目录
- [7.1 MCP 在 Claude Code 中的角色](#71-mcp-在-claude-code-中的角色)
- [7.2 MCP 配置优先级](#72-mcp-配置优先级)
- [7.3 Tool Search 优化](#73-tool-search-优化)
- [7.4 实际 MCP 服务器示例](#74-实际-mcp-服务器示例)

---

## 7.1 MCP 在 Claude Code 中的角色

### MCP 架构概览

```
┌─────────────────────────────────────────────────────────┐
│            Claude Code (MCP Client)                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  内置工具层 (Built-in Tools)                             │
│  ┌────────────────────────────────────────────────┐     │
│  │ - File Operations (Read, Write, Edit)         │     │
│  │ - Code Search (Glob, Grep)                     │     │
│  │ - Bash Execution                               │     │
│  │ - Git Operations                               │     │
│  │ - Task Management                              │     │
│  │ - Web (Search, Fetch)                          │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│                          +                               │
│                                                          │
│  MCP 工具层 (MCP Tools via Protocol)                     │
│  ┌────────────────────────────────────────────────┐     │
│  │  MCP Client 实现                               │     │
│  │  ├─ Tool Discovery                             │     │
│  │  ├─ Tool Invocation                            │     │
│  │  ├─ Resource Management                        │     │
│  │  └─ Prompt Integration                         │     │
│  └────────────────────────────────────────────────┘     │
│                          │                               │
└──────────────────────────┼───────────────────────────────┘
                           │ MCP Protocol
           ┌───────────────┼───────────────┐
           │               │               │
           ↓               ↓               ↓
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │ Local MCP   │ │ Remote MCP  │ │  MCP Apps   │
    │ Servers     │ │ Servers     │ │ (UI-based)  │
    └─────────────┘ └─────────────┘ └─────────────┘
           │               │               │
           ↓               ↓               ↓
    ┌─────────────────────────────────────────────┐
    │        External Services & Data              │
    ├─────────────────────────────────────────────┤
    │ - Databases (PostgreSQL, MongoDB, etc.)     │
    │ - APIs (REST, GraphQL)                      │
    │ - Design Tools (Figma, Sketch)              │
    │ - Project Management (Jira, Linear)         │
    │ - Analytics (Google Analytics, Mixpanel)    │
    │ - Cloud Services (AWS, GCP, Azure)          │
    │ - Custom Internal Tools                     │
    └─────────────────────────────────────────────┘
```

### MCP 的三种集成模式

```
1. Local MCP Servers (本地服务器)
   ┌────────────────────────────────┐
   │  特点:                          │
   │  - 与 Claude Code 同一进程      │
   │  - 低延迟                       │
   │  - 完全本地控制                 │
   │                                │
   │  使用场景:                      │
   │  - 本地数据库访问               │
   │  - 文件系统扩展                 │
   │  - 本地工具包装                 │
   │                                │
   │  示例:                          │
   │  - SQLite MCP Server           │
   │  - Local File Indexer          │
   │  - System Monitoring           │
   └────────────────────────────────┘

2. Remote MCP Servers (远程服务器)
   ┌────────────────────────────────┐
   │  特点:                          │
   │  - 通过网络通信                 │
   │  - 可共享给团队                 │
   │  - 集中管理                     │
   │                                │
   │  使用场景:                      │
   │  - 云数据库访问                 │
   │  - 内部 API 封装                │
   │  - 企业级服务                   │
   │                                │
   │  示例:                          │
   │  - Company API Server          │
   │  - Production DB (read-only)   │
   │  - CI/CD Integration           │
   └────────────────────────────────┘

3. MCP Apps (交互式应用)
   ┌────────────────────────────────┐
   │  特点:                          │
   │  - 支持 UI 交互                 │
   │  - 复杂工作流                   │
   │  - 可视化操作                   │
   │                                │
   │  使用场景:                      │
   │  - 设计工具集成                 │
   │  - 数据可视化                   │
   │  - 交互式配置                   │
   │                                │
   │  示例:                          │
   │  - Figma Integration           │
   │  - Database GUI                │
   │  - Chart Builder               │
   └────────────────────────────────┘
```

### MCP 能力

```
MCP 协议提供的核心能力:

1. Tools (工具)
   └─ 供 Claude 调用的函数
   └─ 类似内置工具但由外部提供

2. Resources (资源)
   └─ 外部数据源
   └─ 文件、数据库、API 等

3. Prompts (提示模板)
   └─ 预定义的提示词
   └─ 可复用的指令

4. Sampling (采样)
   └─ 请求 Claude 生成内容
   └─ 子任务委派
```

---

## 7.2 MCP 配置优先级

### 四层配置体系

```
MCP 服务器配置优先级 (从高到低):

┌─────────────────────────────────────────────────────────┐
│ 1. Local Scope (最高优先级)                              │
│    .claude/mcp.json                                      │
├─────────────────────────────────────────────────────────┤
│ 用途: 项目本地特定配置                                   │
│ 特点:                                                    │
│ - 不提交到 Git (在 .gitignore 中)                        │
│ - 个人开发环境配置                                       │
│ - 覆盖所有其他配置                                       │
│                                                          │
│ 示例:                                                    │
│ {                                                        │
│   "mcpServers": {                                        │
│     "my-local-db": {                                     │
│       "command": "npx",                                  │
│       "args": ["-y", "@local/db-server"],                │
│       "env": {                                           │
│         "DB_PATH": "/Users/me/local.db"                  │
│       }                                                  │
│     }                                                    │
│   }                                                      │
│ }                                                        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ 2. Project Scope                                         │
│    .claude-config.json                                   │
├─────────────────────────────────────────────────────────┤
│ 用途: 团队共享的项目配置                                 │
│ 特点:                                                    │
│ - 提交到 Git                                             │
│ - 团队成员共享                                           │
│ - 项目标准配置                                           │
│                                                          │
│ 示例:                                                    │
│ {                                                        │
│   "mcpServers": {                                        │
│     "project-api": {                                     │
│       "command": "node",                                 │
│       "args": ["./mcp/api-server.js"],                   │
│       "env": {                                           │
│         "API_BASE": "https://api.example.com"            │
│       }                                                  │
│     },                                                   │
│     "figma": {                                           │
│       "command": "npx",                                  │
│       "args": ["-y", "@figma/mcp-server"],               │
│       "env": {                                           │
│         "FIGMA_TOKEN": "${FIGMA_TOKEN}"                  │
│       }                                                  │
│     }                                                    │
│   }                                                      │
│ }                                                        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ 3. User Scope                                            │
│    ~/.claude/mcp.json                                    │
├─────────────────────────────────────────────────────────┤
│ 用途: 用户全局配置                                       │
│ 特点:                                                    │
│ - 所有项目通用                                           │
│ - 个人常用 MCP 服务器                                    │
│ - 优先级低于项目配置                                     │
│                                                          │
│ 示例:                                                    │
│ {                                                        │
│   "mcpServers": {                                        │
│     "my-notes": {                                        │
│       "command": "node",                                 │
│       "args": ["~/mcp/notes-server.js"]                  │
│     },                                                   │
│     "personal-db": {                                     │
│       "command": "sqlite-mcp-server",                    │
│       "args": ["~/personal.db"]                          │
│     }                                                    │
│   }                                                      │
│ }                                                        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ 4. Enterprise Policy (最低优先级)                        │
│    /etc/claude/policy.json                               │
├─────────────────────────────────────────────────────────┤
│ 用途: 组织级别强制策略                                   │
│ 特点:                                                    │
│ - IT 管理员设置                                          │
│ - 强制性安全规则                                         │
│ - 可被上层配置覆盖                                       │
│                                                          │
│ 示例:                                                    │
│ {                                                        │
│   "mcpServers": {                                        │
│     "company-api": {                                     │
│       "command": "company-mcp-server",                   │
│       "args": ["--secure"],                              │
│       "required": true  // 强制启用                      │
│     }                                                    │
│   },                                                     │
│   "policies": {                                          │
│     "allowedServers": ["company-*", "approved-*"],       │
│     "blockedServers": ["*-dev", "unsafe-*"],             │
│     "requireAuth": true                                  │
│   }                                                      │
│ }                                                        │
└─────────────────────────────────────────────────────────┘
```

### 配置合并逻辑

```javascript
// 配置解析示例

// 1. 加载所有层级的配置
const enterpriseConfig = loadConfig('/etc/claude/policy.json');
const userConfig = loadConfig('~/.claude/mcp.json');
const projectConfig = loadConfig('.claude-config.json');
const localConfig = loadConfig('.claude/mcp.json');

// 2. 合并配置 (高优先级覆盖低优先级)
const finalConfig = mergeConfigs([
  enterpriseConfig,  // 最低优先级
  userConfig,
  projectConfig,
  localConfig        // 最高优先级
]);

// 3. 应用策略约束
const validatedConfig = applyPolicies(
  finalConfig,
  enterpriseConfig.policies
);

// 4. 启动 MCP 服务器
for (const [name, config] of Object.entries(validatedConfig.mcpServers)) {
  if (isAllowed(name, enterpriseConfig.policies)) {
    startMCPServer(name, config);
  }
}
```

---

## 7.3 Tool Search 优化

### Token 使用对比

```
场景: 项目集成了 50 个 MCP 服务器，每个提供 10 个工具
总工具数: 500 + 15 (内置) = 515 个工具

┌─────────────────────────────────────────────────────────┐
│  传统方法: 发送所有工具定义                              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  每次请求的 Tokens:                                      │
│  ├─ 系统提示: 5,000 tokens                               │
│  ├─ 内置工具 (15): ~15,000 tokens                        │
│  ├─ MCP 工具 (500): ~100,000 tokens                      │
│  └─ 可用上下文: ~80,000 tokens                           │
│  ─────────────────────────────────────                   │
│  总计: 200,000 tokens                                    │
│                                                          │
│  问题:                                                   │
│  ❌ 50% 的上下文被工具定义占用                           │
│  ❌ Claude 难以在 500 个工具中找到相关的                 │
│  ❌ 实际工作空间严重不足                                 │
│  ❌ 响应质量下降                                         │
│                                                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Tool Search: 智能工具发现                               │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  初始 Tokens:                                            │
│  ├─ 系统提示: 5,000 tokens                               │
│  ├─ 内置工具 (15): ~15,000 tokens                        │
│  ├─ Tool Search 工具: ~1,000 tokens                      │
│  └─ 可用上下文: ~179,000 tokens                          │
│  ─────────────────────────────────────                   │
│  总计: 200,000 tokens                                    │
│                                                          │
│  工作流程:                                               │
│  1. Claude 分析用户请求                                  │
│  2. 调用 Tool Search("database query user data")         │
│  3. Tool Search 返回相关工具 (仅 5-10 个)                │
│     ├─ PostgreSQL Query Tool (~2,000 tokens)             │
│     ├─ MongoDB Query Tool (~2,000 tokens)                │
│     └─ SQL Builder Tool (~1,500 tokens)                  │
│  4. 动态加载这些工具的完整定义 (~5,500 tokens)           │
│  5. Claude 使用相关工具完成任务                          │
│                                                          │
│  优势:                                                   │
│  ✅ 89.5% 的空间可用于实际工作                           │
│  ✅ 仅加载相关的 MCP 工具                                │
│  ✅ 显著提升响应质量                                     │
│  ✅ 节省 ~94,500 tokens (47%)                            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Tool Search 配置

```json
// .claude/config.json
{
  "toolSearch": {
    "enabled": true,

    // 搜索策略
    "strategy": "semantic",  // semantic | keyword | hybrid

    // 缓存配置
    "cache": {
      "enabled": true,
      "ttl": 3600,  // 1小时
      "maxSize": 100  // 最多缓存 100 个查询
    },

    // 结果限制
    "maxResults": 10,  // 每次最多返回 10 个工具

    // 工具索引
    "index": {
      "updateInterval": 300,  // 5分钟更新一次索引
      "includeMetadata": true,
      "rankByRelevance": true
    },

    // 预加载常用工具
    "preload": [
      "file-operations",
      "git-commands",
      "web-search"
    ]
  }
}
```

### Tool Search 使用示例

```
User: "查询数据库中最活跃的10个用户"

Claude 内部流程:

1. 分析请求
   关键词: database, query, users

2. 调用 Tool Search
   ToolSearch({
     query: "database query user data",
     context: "find active users",
     limit: 10
   })

3. Tool Search 返回
   [
     {
       name: "postgresql-query",
       description: "Execute SQL queries on PostgreSQL",
       relevance: 0.95
     },
     {
       name: "mongodb-find",
       description: "Query MongoDB collections",
       relevance: 0.82
     },
     {
       name: "sql-builder",
       description: "Build SQL queries programmatically",
       relevance: 0.78
     },
     ...
   ]

4. 动态加载工具定义
   仅加载前 3 个最相关的工具

5. Claude 选择合适的工具
   决定使用 postgresql-query

6. 执行查询
   PostgreSQLQuery({
     query: `
       SELECT user_id, COUNT(*) as activity_count
       FROM user_activities
       WHERE last_active > NOW() - INTERVAL '30 days'
       GROUP BY user_id
       ORDER BY activity_count DESC
       LIMIT 10
     `
   })

Token 节省:
  传统: 100,000 tokens (所有 MCP 工具)
  Tool Search: 5,500 tokens (仅 3 个工具)
  节省: 94,500 tokens (94.5%)
```

---

## 7.4 实际 MCP 服务器示例

### 示例 1: PostgreSQL MCP Server

```json
// .claude-config.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_URL": "postgresql://user:pass@localhost:5432/mydb"
      }
    }
  }
}
```

**使用**:
```
User: "查询 users 表中所有管理员"

Claude: [Tool Search 找到 postgres 工具]
  [调用 PostgreSQL Query]

PostgreSQLQuery({
  query: "SELECT * FROM users WHERE role = 'admin'"
})

结果:
  ┌────┬──────────┬─────────┬────────────────────┐
  │ id │   name   │  role   │       email        │
  ├────┼──────────┼─────────┼────────────────────┤
  │  1 │ Alice    │  admin  │ alice@example.com  │
  │  5 │ Bob      │  admin  │ bob@example.com    │
  │ 12 │ Charlie  │  admin  │ charlie@ex.com     │
  └────┴──────────┴─────────┴────────────────────┘

  找到 3 个管理员
```

### 示例 2: Figma MCP Server

```json
{
  "mcpServers": {
    "figma": {
      "command": "node",
      "args": ["./mcp/figma-server.js"],
      "env": {
        "FIGMA_TOKEN": "${FIGMA_TOKEN}",
        "FIGMA_FILE_ID": "${FIGMA_FILE_ID}"
      }
    }
  }
}
```

**自定义 Figma MCP Server** (`mcp/figma-server.js`):
```javascript
const { MCPServer } = require('@modelcontextprotocol/sdk');
const axios = require('axios');

class FigmaMCPServer extends MCPServer {
  constructor() {
    super('figma-server', '1.0.0');

    // 注册工具
    this.registerTool({
      name: 'figma-get-styles',
      description: 'Get color styles from Figma file',
      parameters: {
        type: 'object',
        properties: {}
      },
      handler: this.getStyles.bind(this)
    });

    this.registerTool({
      name: 'figma-export-component',
      description: 'Export Figma component as SVG',
      parameters: {
        type: 'object',
        properties: {
          componentId: { type: 'string' }
        },
        required: ['componentId']
      },
      handler: this.exportComponent.bind(this)
    });
  }

  async getStyles() {
    const response = await axios.get(
      `https://api.figma.com/v1/files/${process.env.FIGMA_FILE_ID}/styles`,
      {
        headers: {
          'X-Figma-Token': process.env.FIGMA_TOKEN
        }
      }
    );

    const colors = Object.values(response.data.meta.styles)
      .filter(s => s.style_type === 'FILL')
      .map(s => ({
        name: s.name,
        id: s.node_id
      }));

    return { colors };
  }

  async exportComponent({ componentId }) {
    const response = await axios.get(
      `https://api.figma.com/v1/images/${process.env.FIGMA_FILE_ID}`,
      {
        params: {
          ids: componentId,
          format: 'svg'
        },
        headers: {
          'X-Figma-Token': process.env.FIGMA_TOKEN
        }
      }
    );

    return {
      url: response.data.images[componentId]
    };
  }
}

// 启动服务器
const server = new FigmaMCPServer();
server.start();
```

**使用**:
```
User: "从 Figma 获取所有颜色样式并生成 Tailwind 配置"

Claude: [调用 figma-get-styles]
  结果: {
    colors: [
      { name: "Primary/500", id: "..." },
      { name: "Secondary/500", id: "..." },
      ...
    ]
  }

  [生成 Tailwind 配置]

  Write: tailwind.config.js
  ```javascript
  module.exports = {
    theme: {
      colors: {
        primary: {
          500: '#3B82F6'
        },
        secondary: {
          500: '#8B5CF6'
        },
        ...
      }
    }
  }
  ```

  ✓ Tailwind 配置已生成
    基于 Figma 的 12 个颜色样式
```

### 示例 3: GitHub MCP Server

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}",
        "GITHUB_REPO": "owner/repo"
      }
    }
  }
}
```

**使用**:
```
User: "创建一个 issue 来追踪今天发现的 bug"

Claude: [调用 github-create-issue]

GitHubCreateIssue({
  title: "Bug: 用户登录失败",
  body: `
## 描述
用户在登录时遇到 500 错误

## 复现步骤
1. 访问 /login
2. 输入正确的凭证
3. 点击提交
4. 看到 500 错误

## 相关文件
- src/auth/login.ts:45

## 任务
- [ ] 调查 server logs
- [ ] 修复认证逻辑
- [ ] 添加测试
  `,
  labels: ["bug", "high-priority"],
  assignees: ["current-user"]
})

结果:
  ✓ Issue 已创建
    Issue #123: Bug: 用户登录失败
    URL: https://github.com/owner/repo/issues/123
```

### 示例 4: Custom API MCP Server

```javascript
// mcp/company-api-server.js

const { MCPServer } = require('@modelcontextprotocol/sdk');
const axios = require('axios');

class CompanyAPIMCPServer extends MCPServer {
  constructor() {
    super('company-api', '1.0.0');

    // 注册多个工具
    this.registerTools([
      {
        name: 'get-employee-info',
        description: 'Get employee information',
        parameters: {
          type: 'object',
          properties: {
            employeeId: { type: 'string' }
          }
        },
        handler: this.getEmployeeInfo.bind(this)
      },
      {
        name: 'create-leave-request',
        description: 'Create a leave request',
        parameters: {
          type: 'object',
          properties: {
            startDate: { type: 'string' },
            endDate: { type: 'string'},
            reason: { type: 'string' }
          }
        },
        handler: this.createLeaveRequest.bind(this)
      },
      {
        name: 'get-project-status',
        description: 'Get current project status',
        parameters: {
          type: 'object',
          properties: {
            projectId: { type: 'string' }
          }
        },
        handler: this.getProjectStatus.bind(this)
      }
    ]);

    // 注册资源
    this.registerResource({
      uri: 'company://policies',
      name: 'Company Policies',
      description: 'Access company policy documents',
      mimeType: 'application/json',
      handler: this.getPolicies.bind(this)
    });
  }

  async getEmployeeInfo({ employeeId }) {
    const response = await this.apiCall(`/employees/${employeeId}`);
    return response.data;
  }

  async createLeaveRequest({ startDate, endDate, reason }) {
    const response = await this.apiCall('/leave-requests', {
      method: 'POST',
      data: { startDate, endDate, reason }
    });
    return response.data;
  }

  async getProjectStatus({ projectId }) {
    const response = await this.apiCall(`/projects/${projectId}/status`);
    return response.data;
  }

  async getPolicies() {
    const response = await this.apiCall('/policies');
    return response.data;
  }

  async apiCall(endpoint, options = {}) {
    return axios({
      baseURL: process.env.COMPANY_API_BASE,
      url: endpoint,
      headers: {
        'Authorization': `Bearer ${process.env.COMPANY_API_TOKEN}`,
        'Content-Type': 'application/json'
      },
      ...options
    });
  }
}

const server = new CompanyAPIMCPServer();
server.start();
```

**使用**:
```
User: "查看项目 PRJ-123 的状态并创建请假申请"

Claude: [并行调用两个工具]

1. GetProjectStatus({ projectId: "PRJ-123" })
   结果: {
     status: "on-track",
     completion: 75,
     nextMilestone: "2026-02-15"
   }

2. CreateLeaveRequest({
     startDate: "2026-02-10",
     endDate: "2026-02-14",
     reason: "Personal"
   })
   结果: {
     id: "LR-456",
     status: "pending",
     approver: "manager@example.com"
   }

Claude: "✓ 项目状态: 进度正常 (75%)
         下一个里程碑: 2026-02-15

         ✓ 请假申请已提交
         申请 ID: LR-456
         状态: 等待批准
         审批人: manager@example.com"
```

---

## 下一步

- 查看 [最佳实践](./08-best-practices.md)，优化 MCP 使用
- 参考 [参考资源](./09-references.md)，深入学习 MCP 开发

---

**参考资源**:
- [MCP Integration - Claude Code Docs](https://code.claude.com/docs/en/mcp)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [MCP Server Examples](https://github.com/modelcontextprotocol/servers)
