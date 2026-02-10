# 03. Subagent（子代理）系统

## 目录
- [3.1 内置子代理类型及特性](#31-内置子代理类型及特性)
- [3.2 工具访问权限矩阵](#32-工具访问权限矩阵)
- [3.3 生命周期管理](#33-生命周期管理)
- [3.4 代理选择策略](#34-代理选择策略)
- [3.5 协作机制](#35-协作机制)
- [3.6 自定义 Subagent](#36-自定义-subagent)

---

## 3.1 内置子代理类型及特性

Claude Code 提供三种专门化的子代理类型，每种针对特定场景优化。

### Explore Subagent - 代码探索专家

```yaml
┌─────────────────────────────────────────────────────────┐
│              Explore Subagent 配置详情                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  角色定位: 只读探索者、代码库分析专家                     │
│                                                          │
│  工具权限:                                               │
│  ✓ Read - 读取文件内容                                   │
│  ✓ Glob - 文件模式匹配搜索                               │
│  ✓ Grep - 代码内容搜索 (ripgrep)                         │
│  ✓ WebSearch - 在线搜索                                  │
│  ✓ WebFetch - 获取网页内容                               │
│  ✓ Task - 管理和创建任务                                 │
│  ✗ Write - 不能写入文件                                  │
│  ✗ Edit - 不能编辑文件                                   │
│  ✗ Bash - 不能执行命令                                   │
│                                                          │
│  特点:                                                   │
│  - 轻量级上下文窗口                                      │
│  - 快速启动和执行                                        │
│  - 零副作用（只读）                                      │
│  - 适合并行搜索                                          │
│                                                          │
│  性能特征:                                               │
│  - 启动时间: <1s                                         │
│  - 内存占用: 最小                                        │
│  - 并发限制: 7 个实例                                    │
│                                                          │
│  最佳使用场景:                                           │
│  1. 代码库结构分析                                       │
│     "探索项目中所有的 API 端点定义"                      │
│                                                          │
│  2. 依赖关系追踪                                         │
│     "找出所有使用 useState 的组件"                       │
│                                                          │
│  3. 模式识别                                             │
│     "搜索所有的错误处理代码"                             │
│                                                          │
│  4. 文档和注释查找                                       │
│     "找到关于认证流程的所有文档"                         │
│                                                          │
│  5. 影响范围分析                                         │
│     "如果修改这个函数，会影响哪些文件？"                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Explore Subagent 使用示例

```bash
# 场景 1: 理解代码库结构
User: "这个项目的路由是如何组织的？"

Claude: [启动 Explore Subagent]

Explore Agent 执行:
  ├─ Glob: "**/*route*.{ts,js,tsx,jsx}"
  │  找到: 23 个路由相关文件
  │
  ├─ Read: src/routes/index.ts
  │  分析主路由配置
  │
  ├─ Grep: "createBrowserRouter"
  │  找到路由定义模式
  │
  └─ 生成报告:
      "项目使用 React Router v6
       路由配置在 src/routes/ 目录
       主要路由:
       - /auth/* (认证相关)
       - /dashboard/* (仪表板)
       - /api/* (API 路由)"

# 场景 2: 查找特定实现
User: "找到所有使用 useEffect 清理函数的地方"

Claude: [并行启动 3 个 Explore Subagents]

Explore Agent 1:
  ├─ Grep: "useEffect.*return.*=>" in src/
  └─ 找到 12 处使用

Explore Agent 2:
  ├─ Grep: "useEffect.*cleanup" in src/
  └─ 找到 5 处使用

Explore Agent 3:
  ├─ Grep: "useEffect.*unmount" in src/
  └─ 找到 3 处使用

合并结果: 20 个独特的清理函数用例

# 场景 3: Plan Mode 的自动使用
User: /plan "重构认证系统"

Claude: [自动使用 Explore Subagent 进行分析]

Explore Agent:
  ├─ 第 1 步: 定位认证相关文件
  │  Glob: "**/*auth*.{ts,tsx}"
  │  找到: src/auth/, src/hooks/useAuth.ts, ...
  │
  ├─ 第 2 步: 分析依赖关系
  │  Grep: "import.*auth" (全局搜索)
  │  发现 34 个文件依赖认证模块
  │
  ├─ 第 3 步: 检查测试覆盖
  │  Glob: "**/*auth*.test.ts"
  │  找到 8 个测试文件
  │
  └─ 第 4 步: 生成重构计划
      影响范围: 34 个文件
      建议步骤:
      1. 保持向后兼容的 API
      2. 逐步迁移依赖文件
      3. 更新测试
      4. 最后删除旧代码
```

### General Subagent - 全能执行者

```yaml
┌─────────────────────────────────────────────────────────┐
│             General Subagent 配置详情                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  角色定位: 完整功能代理、复杂任务执行者                   │
│                                                          │
│  工具权限:                                               │
│  ✓ Read - 读取文件                                       │
│  ✓ Write - 创建文件                                      │
│  ✓ Edit - 编辑文件                                       │
│  ✓ Glob - 文件搜索                                       │
│  ✓ Grep - 内容搜索                                       │
│  ✓ Bash - 命令执行（需权限）                             │
│  ✓ Task - 任务管理                                       │
│  ✓ WebSearch - 在线搜索                                  │
│  ✓ WebFetch - 网页获取                                   │
│  ✓ Git - 版本控制（需权限）                              │
│                                                          │
│  特点:                                                   │
│  - 完整的代理能力                                        │
│  - 独立的上下文窗口                                      │
│  - 可以并行运行多个实例                                  │
│  - 适合复杂的多步骤任务                                  │
│                                                          │
│  性能特征:                                               │
│  - 启动时间: 1-2s                                        │
│  - 内存占用: 中等                                        │
│  - 上下文窗口: 独立 200k tokens                          │
│  - 并发限制: 7 个实例                                    │
│                                                          │
│  最佳使用场景:                                           │
│  1. 独立功能实现                                         │
│     "实现用户注册流程（前后端）"                         │
│                                                          │
│  2. 复杂重构                                             │
│     "将 Class 组件迁移到 Function 组件"                  │
│                                                          │
│  3. 测试编写                                             │
│     "为所有 API 端点编写单元测试"                        │
│                                                          │
│  4. 文档生成                                             │
│     "生成 API 文档和使用示例"                            │
│                                                          │
│  5. Bug 修复                                             │
│     "修复登录失败的问题（需要调试）"                     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### General Subagent 使用示例

```bash
# 场景 1: 并行实现多个功能
User: "实现用户注册、登录和密码重置功能"

Claude: [创建 3 个并行任务，分配给 General Subagents]

General Subagent #1 - 用户注册:
  ├─ Read: src/api/auth.ts (理解现有结构)
  ├─ Edit: src/api/auth.ts (添加 register 端点)
  ├─ Write: src/pages/Register.tsx (注册页面)
  ├─ Write: tests/auth.register.test.ts (测试)
  └─ Bash: npm test -- auth.register

General Subagent #2 - 用户登录:
  ├─ Read: src/api/auth.ts
  ├─ Edit: src/api/auth.ts (添加 login 端点)
  ├─ Write: src/pages/Login.tsx
  ├─ Write: tests/auth.login.test.ts
  └─ Bash: npm test -- auth.login

General Subagent #3 - 密码重置:
  ├─ Read: src/api/auth.ts
  ├─ Edit: src/api/auth.ts (添加 resetPassword 端点)
  ├─ Write: src/pages/ResetPassword.tsx
  ├─ Write: tests/auth.reset.test.ts
  └─ Bash: npm test -- auth.reset

主代理: 等待所有子代理完成，然后集成

# 场景 2: 复杂的多文件重构
User: "将所有 Axios 调用替换为 Fetch API"

General Subagent:
  ├─ 第 1 步: 分析现有代码
  │  Grep: "import.*axios"
  │  找到 45 个文件使用 axios
  │
  ├─ 第 2 步: 创建工具函数
  │  Write: src/utils/fetch.ts
  │  (包含 axios 兼容的 fetch 封装)
  │
  ├─ 第 3 步: 逐个文件迁移
  │  For each file in [45 files]:
  │    ├─ Read: file
  │    ├─ Edit: 替换 axios 为 fetch
  │    └─ 验证语法
  │
  ├─ 第 4 步: 更新依赖
  │  Edit: package.json (移除 axios)
  │  Bash: npm install
  │
  └─ 第 5 步: 运行测试
     Bash: npm test
     (确保所有测试通过)
```

### Bash Subagent - 命令行大师

```yaml
┌─────────────────────────────────────────────────────────┐
│              Bash Subagent 配置详情                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  角色定位: 命令行操作专家、脚本执行者                     │
│                                                          │
│  工具权限:                                               │
│  ✓ Bash - 完整的 Bash 命令执行（需权限）                 │
│  ✓ Glob - 文件查找（用于构建命令参数）                   │
│  ✗ Read - 不直接读取文件（使用 cat/head/tail）          │
│  ✗ Write - 不直接写入（使用重定向）                      │
│  ✗ Task - 不管理任务                                     │
│                                                          │
│  特点:                                                   │
│  - 专门优化的命令执行                                    │
│  - 支持复杂的 shell 脚本                                 │
│  - 流式输出处理                                          │
│  - 适合长时间运行的命令                                  │
│                                                          │
│  性能特征:                                               │
│  - 启动时间: <500ms                                      │
│  - 内存占用: 最小                                        │
│  - 支持后台运行 (Ctrl+B)                                 │
│  - 并发限制: 7 个实例                                    │
│                                                          │
│  最佳使用场景:                                           │
│  1. 构建和编译                                           │
│     "运行生产构建"                                       │
│                                                          │
│  2. 测试执行                                             │
│     "运行所有单元测试和集成测试"                         │
│                                                          │
│  3. 依赖管理                                             │
│     "安装项目依赖并更新"                                 │
│                                                          │
│  4. Git 操作                                             │
│     "提交更改并推送到远程"                               │
│                                                          │
│  5. 开发服务器                                           │
│     "启动开发服务器（后台）"                             │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Bash Subagent 使用示例

```bash
# 场景 1: 并行运行多个构建任务
User: "运行 lint、测试和构建"

Claude: [创建 3 个并行 Bash Subagents]

Bash Subagent #1:
  └─ npm run lint
     (检查代码规范)

Bash Subagent #2:
  └─ npm test
     (运行所有测试)

Bash Subagent #3:
  └─ npm run build
     (生产构建)

所有任务并行执行，总时间缩短 70%

# 场景 2: 后台运行开发服务器
User: "启动开发服务器并监控日志"

Claude: [启动 Bash Subagent，后台模式]

Bash Subagent (background):
  └─ npm run dev | tee dev.log

用户按 Ctrl+B 将其移到后台

主代理: 继续接受其他命令
用户: "现在修改 Login 组件"
Claude: [在开发服务器运行时进行修改]

# 场景 3: 复杂的 Git 工作流
User: "创建功能分支、提交更改、推送并创建 PR"

Bash Subagent:
  ├─ git checkout -b feature/user-dashboard
  ├─ git add src/pages/Dashboard.tsx
  ├─ git add src/components/UserStats.tsx
  ├─ git commit -m "$(cat <<'EOF'
     feat: Add user dashboard

     - Implemented dashboard layout
     - Added user statistics component
     - Integrated real-time data fetching

     Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
     EOF
     )"
  ├─ git push -u origin feature/user-dashboard
  └─ gh pr create --title "User Dashboard" --body "..."

完整的 Git 工作流一次性完成
```

---

## 3.2 工具访问权限矩阵

### 完整权限对照表

```
┌──────────────────┬─────────┬─────────┬──────┬────────────┬─────────────┐
│ 工具 / 代理      │ Explore │ General │ Bash │ Plan Agent │ Build Agent │
├──────────────────┼─────────┼─────────┼──────┼────────────┼─────────────┤
│ Read             │    ✓    │    ✓    │  -   │     ✓      │      ✓      │
│ Write            │    ✗    │    ✓    │  ✓*  │     ✗      │      ✓      │
│ Edit             │    ✗    │    ✓    │  ✓*  │     ✗      │      ✓      │
│ Glob             │    ✓    │    ✓    │  ✓   │     ✓      │      ✓      │
│ Grep             │    ✓    │    ✓    │  ✓   │     ✓      │      ✓      │
│ Bash (读命令)    │    ✗    │    ✓    │  ✓   │     ✗      │      ✓      │
│ Bash (写命令)    │    ✗    │   ✓**   │ ✓**  │     ✗      │     ✓**     │
│ Bash (危险命令)  │    ✗    │   ✗***  │ ✗*** │     ✗      │    ✗***     │
│ Git (read)       │    ✓    │    ✓    │  ✓   │     ✓      │      ✓      │
│ Git (write)      │    ✗    │   ✓**   │ ✓**  │     ✗      │     ✓**     │
│ Git (push)       │    ✗    │   ✗***  │ ✗*** │     ✗      │    ✗***     │
│ Task Management  │    ✓    │    ✓    │  ✗   │     ✓      │      ✓      │
│ WebSearch        │    ✓    │    ✓    │  ✓   │     ✓      │      ✓      │
│ WebFetch         │    ✓    │    ✓    │  ✓   │     ✓      │      ✓      │
│ MCP Tools        │  视配置  │  视配置  │视配置│   视配置    │    视配置    │
└──────────────────┴─────────┴─────────┴──────┴────────────┴─────────────┘

图例:
  ✓   - 总是允许
  ✓*  - 通过 Bash 重定向实现
  ✓** - 需要权限许可（根据运行模式）
  ✗***- 需要显式用户确认
  ✗   - 不允许
  -   - 不适用
  视配置 - 取决于 MCP 服务器配置
```

### 权限继承链

```
代理权限决策流程:

1. 基础权限 (Subagent Type)
   │
   ├─ Explore: 只读权限集合
   ├─ General: 完整权限集合
   └─ Bash: 命令执行权限
   │
   ↓
2. 运行模式过滤 (Running Mode)
   │
   ├─ YOLO: 允许所有
   ├─ Accepting Edits: 过滤危险操作
   ├─ Plan Mode: 仅只读
   └─ Default: 逐个确认
   │
   ↓
3. 用户配置覆盖 (User Config)
   │
   ├─ Allow List: 显式允许
   ├─ Deny List: 显式拒绝
   └─ Ask List: 需要确认
   │
   ↓
4. Hook 验证 (PreToolUse Hook)
   │
   ├─ 自定义安全检查
   ├─ 审计日志
   └─ 条件性拦截
   │
   ↓
5. 最终决策
   │
   ├─ Allow → 执行
   ├─ Deny → 拒绝并报错
   └─ Ask → 提示用户确认
```

### 示例: 权限决策过程

```bash
场景: General Subagent 尝试执行 "rm -rf node_modules"

Step 1: 基础权限检查
  General Subagent 有 Bash 工具权限 ✓

Step 2: 运行模式检查
  当前模式: Accepting Edits
  rm -rf 是危险命令
  结果: 需要进一步检查 ?

Step 3: 用户配置检查
  ~/.claude/config.json:
  {
    "deny": [
      {
        "tool": "Bash",
        "pattern": "rm -rf *",
        "description": "禁止递归删除"
      }
    ]
  }
  结果: DENY ✗

Step 4: 最终决策
  拒绝执行，返回错误:
  "❌ 命令被拒绝: rm -rf node_modules
   原因: 匹配拒绝规则 'rm -rf *'
   建议: 使用 'rm -rf node_modules/' (末尾加斜杠)
   或者手动执行"

---

场景: Bash Subagent 执行 "npm install"

Step 1: 基础权限 ✓
Step 2: 运行模式 (Accepting Edits) ✓
Step 3: 用户配置 (无特殊规则) ✓
Step 4: Hook 检查
  PreToolUse Hook:
    检查 package.json 是否有变化
    记录安装日志
  结果: 允许 ✓

Step 5: 执行命令
  npm install
  (成功)

Step 6: PostToolUse Hook
  记录安装的包
  更新依赖图
```

---

## 3.3 生命周期管理

### Subagent 完整生命周期

```
┌─────────────────────────────────────────────────────────┐
│            Subagent Lifecycle (详细)                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  阶段 1: 创建 (Creation)                                 │
│  ┌────────────────────────────────────────────────┐     │
│  │  触发条件:                                     │     │
│  │  ├─ 主代理决定需要专门化处理                   │     │
│  │  ├─ 任务分解后的子任务分配                     │     │
│  │  └─ 用户显式请求 (如 /plan)                    │     │
│  │                                                │     │
│  │  创建过程:                                     │     │
│  │  1. 选择子代理类型 (Explore/General/Bash)     │     │
│  │  2. 分配独立的 Agent ID                        │     │
│  │  3. 初始化独立上下文窗口                       │     │
│  │  4. 加载子代理特定的系统提示                   │     │
│  │  5. 应用权限限制                               │     │
│  │                                                │     │
│  │  示例:                                         │     │
│  │  AgentID: general-subagent-1                   │     │
│  │  Type: General                                 │     │
│  │  Context: 200k tokens (独立)                   │     │
│  │  Task: "实现用户认证功能"                      │     │
│  └────────────────────────────────────────────────┘     │
│                          ↓                               │
│  阶段 2: 初始化 (Initialization)                         │
│  ┌────────────────────────────────────────────────┐     │
│  │  上下文传递:                                   │     │
│  │  ├─ 任务描述和目标                             │     │
│  │  ├─ 相关文件列表                               │     │
│  │  ├─ 项目上下文 (CLAUDE.md)                     │     │
│  │  ├─ 依赖任务的结果                             │     │
│  │  └─ 权限和约束                                 │     │
│  │                                                │     │
│  │  系统提示注入:                                 │     │
│  │  "你是一个 General Subagent                    │     │
│  │   专注于: 实现用户认证功能                     │     │
│  │   可用工具: [Read, Write, Edit, Bash, ...]    │     │
│  │   约束: 遵循项目代码规范                       │     │
│  │   完成后: 报告结果并更新任务状态"              │     │
│  │                                                │     │
│  │  环境准备:                                     │     │
│  │  ├─ 设置工作目录                               │     │
│  │  ├─ 加载环境变量                               │     │
│  │  └─ 验证依赖可用性                             │     │
│  └────────────────────────────────────────────────┘     │
│                          ↓                               │
│  阶段 3: 执行 (Execution)                                │
│  ┌────────────────────────────────────────────────┐     │
│  │  工作循环:                                     │     │
│  │                                                │     │
│  │  while (task_not_complete):                    │     │
│  │      1. 分析当前状态                           │     │
│  │      2. 决定下一步操作                         │     │
│  │      3. 调用工具 (受权限限制)                  │     │
│  │      4. 处理工具结果                           │     │
│  │      5. 更新内部状态                           │     │
│  │      6. 检查是否完成                           │     │
│  │                                                │     │
│  │  并行执行 (如果有多个子代理):                  │     │
│  │  ┌──────────────┐  ┌──────────────┐            │     │
│  │  │ Subagent #1  │  │ Subagent #2  │            │     │
│  │  │ (独立运行)   │  │ (独立运行)   │            │     │
│  │  └──────────────┘  └──────────────┘            │     │
│  │        │                  │                    │     │
│  │        └────────┬─────────┘                    │     │
│  │                 ↓                              │     │
│  │         结果汇总到主代理                       │     │
│  │                                                │     │
│  │  错误处理:                                     │     │
│  │  ├─ 工具调用失败 → 重试或替代方案              │     │
│  │  ├─ 权限拒绝 → 请求用户许可                    │     │
│  │  └─ 任务阻塞 → 等待依赖任务完成                │     │
│  └────────────────────────────────────────────────┘     │
│                          ↓                               │
│  阶段 4: 完成 (Completion)                               │
│  ┌────────────────────────────────────────────────┐     │
│  │  结果汇总:                                     │     │
│  │  ├─ 成功/失败状态                              │     │
│  │  ├─ 生成的输出 (文件、数据等)                  │     │
│  │  ├─ 执行日志和警告                             │     │
│  │  └─ 下一步建议                                 │     │
│  │                                                │     │
│  │  任务更新:                                     │     │
│  │  TaskUpdate(                                   │     │
│  │    task_id: "task-123",                        │     │
│  │    status: "completed",                        │     │
│  │    output: {                                   │     │
│  │      summary: "认证功能已实现",                │     │
│  │      files_modified: ["src/auth.ts", ...],     │     │
│  │      tests_passing: true                       │     │
│  │    }                                           │     │
│  │  )                                             │     │
│  │                                                │     │
│  │  依赖任务唤醒:                                 │     │
│  │  └─ 通知所有 blockedBy 此任务的其他任务        │     │
│  └────────────────────────────────────────────────┘     │
│                          ↓                               │
│  阶段 5: 清理 (Cleanup)                                  │
│  ┌────────────────────────────────────────────────┐     │
│  │  上下文处理:                                   │     │
│  │  ├─ 压缩关键信息返回主代理                     │     │
│  │  ├─ 丢弃临时数据                               │     │
│  │  └─ 释放资源                                   │     │
│  │                                                │     │
│  │  持久化 (如需要):                              │     │
│  │  └─ 保存会话状态到磁盘                         │     │
│  │                                                │     │
│  │  Agent 终止:                                   │     │
│  │  └─ 子代理实例销毁                             │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 上下文传递详解

```
主代理 → 子代理的上下文传递:

┌─────────────────────────────────┐
│  Main Agent Context (主代理)    │
│  ├─ 完整的对话历史 (100k)       │
│  ├─ 项目全局理解 (30k)          │
│  ├─ 所有任务状态 (10k)          │
│  └─ 用户偏好 (5k)               │
│  总计: 145k tokens              │
└─────────────────────────────────┘
              │
              │ 精简传递
              ↓
┌─────────────────────────────────┐
│ Subagent Context (子代理)       │
│ ├─ 任务描述 (2k)                │
│ │  "实现用户认证，使用 JWT"     │
│ │                               │
│ ├─ 相关文件指引 (3k)            │
│ │  - src/auth.ts (需要修改)     │
│ │  - src/api/user.ts (参考)     │
│ │                               │
│ ├─ 项目规范 (5k)                │
│ │  - 使用 TypeScript            │
│ │  - 遵循 ESLint 规则           │
│ │  - 必须有单元测试             │
│ │                               │
│ ├─ 依赖任务结果 (10k)           │
│ │  Task 1 的输出:               │
│ │  "数据库schema已准备好"       │
│ │                               │
│ └─ 约束和限制 (2k)              │
│    - 不要修改 package.json      │
│    - 保持向后兼容               │
│                                 │
│ 总计: 22k tokens                │
│ (压缩比: 6.6:1)                 │
└─────────────────────────────────┘

优势:
  ✓ 子代理有更多工作空间 (178k tokens)
  ✓ 聚焦于特定任务
  ✓ 减少无关信息干扰
  ✓ 提高响应质量
```

---

## 3.4 代理选择策略

### 自动选择决策树

```
任务分析
    │
    ↓
┌────────────────────────────────────┐
│ 是否只需要读取和搜索？              │
├────────────────────────────────────┤
│ YES → Explore Subagent              │
│       - 代码库分析                  │
│       - 查找特定模式                │
│       - 理解架构                    │
│       - 影响范围评估                │
└────────────────────────────────────┘
    │ NO
    ↓
┌────────────────────────────────────┐
│ 是否主要是命令行操作？              │
├────────────────────────────────────┤
│ YES → Bash Subagent                 │
│       - 构建和编译                  │
│       - 测试执行                    │
│       - 依赖管理                    │
│       - Git 操作                    │
│       - 脚本运行                    │
└────────────────────────────────────┘
    │ NO
    ↓
┌────────────────────────────────────┐
│ 是否是复杂的多步骤任务？            │
├────────────────────────────────────┤
│ YES → General Subagent              │
│       - 功能实现                    │
│       - 代码重构                    │
│       - Bug 修复                    │
│       - 测试编写                    │
│       - 文档生成                    │
└────────────────────────────────────┘
    │ NO
    ↓
┌────────────────────────────────────┐
│ 是否需要规划和分析？                │
├────────────────────────────────────┤
│ YES → Plan Mode (Explore)           │
│       自动使用 Explore 分析后       │
│       生成实施计划                  │
│       等待用户批准                  │
└────────────────────────────────────┘
    │ NO
    ↓
┌────────────────────────────────────┐
│ 主代理直接处理                      │
│ (简单、快速的任务)                  │
└────────────────────────────────────┘
```

### 显式指定代理

虽然 Claude Code 会自动选择合适的子代理，但你也可以通过提示来引导:

```bash
# 方式 1: 在提示中明确指定
User: "使用 Explore 子代理搜索所有使用 useEffect 的地方"

Claude: [理解意图，启动 Explore Subagent]

# 方式 2: 通过 Plan Mode
User: /plan
[自动使用 Explore 进行分析]

# 方式 3: 任务系统
User: "创建以下任务:
1. 分析代码库结构 (Explore)
2. 实现新功能 (General)
3. 运行测试 (Bash)"

Claude: [创建3个任务，分配给相应子代理]

# 方式 4: 在 CLAUDE.md 中定义偏好
CLAUDE.md:
## Agent Preferences

- 代码分析: 总是使用 Explore Subagent
- 构建任务: 总是使用 Bash Subagent (后台)
- 功能实现: 使用 General Subagent，最多并行3个
```

---

## 3.5 协作机制

### 模式 1: 顺序依赖 (Sequential Dependencies)

```
任务链: A → B → C

┌─────────────────────────────────────────────────────────┐
│  示例: 实现、测试、部署流程                              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Task 1: 代码实现                                        │
│  ├─ Agent: General Subagent #1                           │
│  ├─ 操作:                                                │
│  │  ├─ Read: 现有代码                                    │
│  │  ├─ Write: 新功能代码                                 │
│  │  └─ Edit: 集成到现有系统                              │
│  └─ 输出: 实现完成的代码                                 │
│                                                          │
│                    ↓ blockedBy: Task 1                   │
│                                                          │
│  Task 2: 单元测试                                        │
│  ├─ Agent: General Subagent #2                           │
│  ├─ 等待: Task 1 完成                                    │
│  ├─ 操作:                                                │
│  │  ├─ Read: Task 1 生成的代码                           │
│  │  ├─ Write: 单元测试                                   │
│  │  └─ Bash: 运行测试                                    │
│  └─ 输出: 测试报告 (所有测试通过)                        │
│                                                          │
│                    ↓ blockedBy: Task 2                   │
│                                                          │
│  Task 3: 构建和部署                                      │
│  ├─ Agent: Bash Subagent #1                              │
│  ├─ 等待: Task 2 通过                                    │
│  ├─ 操作:                                                │
│  │  ├─ npm run build                                     │
│  │  ├─ npm run deploy                                    │
│  │  └─ 验证部署                                          │
│  └─ 输出: 部署成功，URL                                  │
│                                                          │
└─────────────────────────────────────────────────────────┘

依赖配置:

TaskCreate({
  subject: "实现新功能",
  description: "...",
  // 无依赖，立即开始
})

TaskCreate({
  subject: "编写单元测试",
  description: "...",
  add_blocked_by: ["task-1"]  // 等待 Task 1
})

TaskCreate({
  subject: "构建和部署",
  description: "...",
  add_blocked_by: ["task-2"]  // 等待 Task 2
})
```

### 模式 2: 并行执行 (Parallel Execution)

```
任务扇出: A → {B, C, D} → E

┌─────────────────────────────────────────────────────────┐
│  示例: 多功能并行实现                                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Task 1: 项目准备                                        │
│  ├─ Agent: Bash Subagent #1                              │
│  ├─ 操作: 安装依赖、设置环境                             │
│  └─ 输出: 环境就绪                                       │
│                                                          │
│            ↓ 完成后触发并行任务                           │
│            ┌─────────┬─────────┬─────────┐               │
│            ↓         ↓         ↓         ↓               │
│                                                          │
│  Task 2      Task 3      Task 4      Task 5              │
│  用户注册    用户登录    个人资料    权限管理            │
│  General #2  General #3  General #4  General #5          │
│                                                          │
│  (同时执行，互不干扰)                                    │
│                                                          │
│            └─────────┬─────────┬─────────┘               │
│                      ↓                                   │
│                                                          │
│  Task 6: 集成和测试                                      │
│  ├─ Agent: General Subagent #6                           │
│  ├─ 等待: Tasks 2-5 全部完成                             │
│  ├─ 操作:                                                │
│  │  ├─ 集成所有功能                                      │
│  │  ├─ 运行集成测试                                      │
│  │  └─ E2E 测试                                          │
│  └─ 输出: 所有功能集成完毕                               │
│                                                          │
└─────────────────────────────────────────────────────────┘

性能提升:

串行执行:
  Task 2 (30 min) → Task 3 (25 min) → Task 4 (20 min) → Task 5 (15 min)
  总时间: 90 分钟

并行执行:
  Task 2, 3, 4, 5 同时运行
  总时间: max(30, 25, 20, 15) = 30 分钟

  提速: 3 倍 ⚡
```

### 模式 3: DAG (Directed Acyclic Graph)

```
复杂的依赖图:

                    ┌─→ Task 4 (API Tests)
                    │         ↓
Task 1 (Setup) ──→ Task 2 ──→ Task 6 (Integration)
                    │         ↓
                    └─→ Task 3 (UI) ──→ Task 7 (E2E)
                            ↓
                        Task 5 (Docs)

实际示例: 电商结账流程

┌─────────────────────────────────────────────────────────┐
│  Task 1: 数据库 Schema 设计                              │
│  ├─ Agent: Explore + General                             │
│  ├─ 分析现有 schema                                      │
│  ├─ 设计订单表、支付表                                   │
│  └─ 执行迁移                                             │
└─────────────────────────────────────────────────────────┘
              │
              ├─────────────┬─────────────┐
              ↓             ↓             ↓
┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐
│ Task 2: 后端 API │ │ Task 3: 前端 │ │ Task 4: 支付集成 │
│ General #1       │ │ General #2   │ │ General #3       │
│ - 订单 CRUD API  │ │ - 购物车 UI  │ │ - Stripe SDK     │
│ - 库存管理       │ │ - 结账流程   │ │ - Webhook 处理   │
└──────────────────┘ └──────────────┘ └──────────────────┘
      │                    │                    │
      └──────────┬─────────┴──────────┬─────────┘
                 ↓                    ↓
        ┌─────────────────┐  ┌──────────────────┐
        │ Task 5: API 测试 │  │ Task 6: UI 测试   │
        │ General #4       │  │ General #5       │
        └─────────────────┘  └──────────────────┘
                 │                    │
                 └──────────┬─────────┘
                            ↓
                ┌─────────────────────────┐
                │ Task 7: 集成测试         │
                │ General #6              │
                │ - 完整结账流程          │
                │ - 支付成功/失败场景     │
                │ - 并发订单处理          │
                └─────────────────────────┘
                            ↓
                ┌─────────────────────────┐
                │ Task 8: 文档和部署       │
                │ General #7 + Bash #1    │
                │ - API 文档              │
                │ - 用户指南              │
                │ - 部署到 staging        │
                └─────────────────────────┘

任务配置:

{
  "tasks": [
    {
      "id": "1",
      "subject": "数据库 Schema",
      "blocked_by": []
    },
    {
      "id": "2",
      "subject": "后端 API",
      "blocked_by": ["1"]
    },
    {
      "id": "3",
      "subject": "前端 UI",
      "blocked_by": ["1"]
    },
    {
      "id": "4",
      "subject": "支付集成",
      "blocked_by": ["1"]
    },
    {
      "id": "5",
      "subject": "API 测试",
      "blocked_by": ["2", "4"]
    },
    {
      "id": "6",
      "subject": "UI 测试",
      "blocked_by": ["3"]
    },
    {
      "id": "7",
      "subject": "集成测试",
      "blocked_by": ["5", "6"]
    },
    {
      "id": "8",
      "subject": "文档和部署",
      "blocked_by": ["7"]
    }
  ]
}

执行流程:
  t=0:  Task 1 开始
  t=10: Task 1 完成 → Tasks 2, 3, 4 并行开始
  t=40: Task 2, 4 完成 → Task 5 开始
  t=45: Task 3 完成 → Task 6 开始
  t=50: Task 5 完成
  t=55: Task 6 完成 → Task 7 开始
  t=70: Task 7 完成 → Task 8 开始
  t=85: Task 8 完成 → 全部完成 ✓

总时间: 85 分钟 (串行需要 ~200 分钟)
```

---

## 3.6 自定义 Subagent

从 Claude Code v2.0+ 开始，支持创建自定义子代理。

### 自定义 Subagent 配置

```yaml
# .claude/subagents/test-runner.yaml

name: "test-runner"
description: "专门用于运行和管理测试的子代理"

# 工具权限
tools:
  allow:
    - Bash  # 运行测试命令
    - Read  # 读取测试结果
    - Grep  # 搜索失败的测试
    - Task  # 管理测试任务
  deny:
    - Write  # 不允许修改代码
    - Edit   # 不允许编辑

# 系统提示
system_prompt: |
  你是一个测试运行专家。

  你的职责:
  1. 执行各种测试 (单元、集成、E2E)
  2. 分析测试失败原因
  3. 生成测试报告
  4. 建议修复方案 (但不直接修改代码)

  你不应该:
  - 修改任何源代码或测试代码
  - 安装新的依赖
  - 修改配置文件

  工作流程:
  1. 理解要运行的测试类型
  2. 执行测试命令
  3. 收集和分析结果
  4. 如果失败，定位问题文件和行号
  5. 生成详细报告

# 默认行为
defaults:
  timeout: 600000  # 10 分钟超时
  retry_on_failure: true
  max_retries: 2

# 元数据
metadata:
  author: "Your Team"
  version: "1.0.0"
  tags: ["testing", "quality"]
```

### 使用自定义 Subagent

```bash
# 方式 1: 在提示中引用
User: "使用 test-runner 子代理运行所有测试"

Claude: [加载自定义 test-runner subagent]

Test-Runner Subagent:
  ├─ Bash: npm run test:unit
  ├─ Bash: npm run test:integration
  ├─ Bash: npm run test:e2e
  └─ 生成综合报告

# 方式 2: 在任务中指定
TaskCreate({
  subject: "运行测试套件",
  description: "执行所有测试并生成报告",
  agent_type: "test-runner",  # 使用自定义子代理
  metadata: {
    test_types: ["unit", "integration", "e2e"]
  }
})

# 方式 3: 在 CLAUDE.md 中配置默认行为
## Agent Configuration

测试任务总是使用 test-runner 子代理
部署任务使用 deploy-agent 子代理
```

### 更多自定义 Subagent 示例

```yaml
# 1. Deploy Agent
# .claude/subagents/deploy-agent.yaml

name: "deploy-agent"
description: "部署和发布专家"

tools:
  allow:
    - Bash
    - Read
    - Task
  deny:
    - Write
    - Edit

behaviors:
  - name: "pre-deploy-check"
    command: "npm run build && npm run test"
    required: true

  - name: "deploy-staging"
    command: "npm run deploy:staging"
    requires_confirmation: true

  - name: "deploy-production"
    command: "npm run deploy:prod"
    requires_confirmation: true
    safety_checks:
      - "所有测试必须通过"
      - "必须有 git tag"
      - "需要两个人批准"

---

# 2. Documentation Agent
# .claude/subagents/doc-agent.yaml

name: "doc-agent"
description: "文档生成和维护专家"

tools:
  allow:
    - Read
    - Write  # 只允许写 markdown 文件
    - Glob
    - Grep

  file_restrictions:
    write_only:
      - "**/*.md"
      - "docs/**/*"

system_prompt: |
  你是文档专家。

  根据代码自动生成:
  - API 文档
  - 使用示例
  - 架构说明
  - README

  遵循项目的文档标准。

---

# 3. Security Audit Agent
# .claude/subagents/security-agent.yaml

name: "security-agent"
description: "安全审计专家"

tools:
  allow:
    - Read
    - Grep
    - Bash  # 运行安全扫描工具
  deny:
    - Write
    - Edit

system_prompt: |
  你是安全审计专家。

  检查:
  - 依赖漏洞 (npm audit)
  - 代码安全问题 (ESLint security plugin)
  - 密钥泄露 (git-secrets)
  - OWASP Top 10 漏洞

  生成安全报告和修复建议。

audit_checklist:
  - command: "npm audit"
    severity: "critical"

  - command: "git-secrets --scan"
    severity: "high"

  - command: "eslint . --ext .js,.ts --rule 'security/*: error'"
    severity: "medium"
```

---

## 下一步

- 学习 [异步 Agent](./04-async-agents.md)，了解后台运行和并行执行
- 查看 [高级特性](./05-advanced-features.md)，掌握任务管理系统
- 参考 [最佳实践](./08-best-practices.md)，优化子代理使用

---

**参考资源**:
- [创建自定义 Subagents - Claude Code Docs](https://code.claude.com/docs/zh-CN/sub-agents)
- [重新定义 AI 编程协作](https://www.cnblogs.com/Chary/p/19312692)
- [Claude Code 多智能体系统架构](https://www.eesel.ai/blog/claude-code-multiple-agent-systems-complete-2026-guide)
