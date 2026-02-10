# 04. 异步 Agent（Async Agents）

## 目录
- [4.1 后台运行机制](#41-后台运行机制)
- [4.2 TaskOutput 获取](#42-taskoutput-获取)
- [4.3 异步任务管理](#43-异步任务管理)
- [4.4 并行执行模式](#44-并行执行模式)
- [4.5 实战场景](#45-实战场景)

---

## 4.1 后台运行机制

异步代理是 Claude Code 实现长时间运行任务的关键特性。

### 异步模式概览

```
同步模式 vs 异步模式:

┌─────────────────────────────────────────────────────────┐
│  同步模式 (Synchronous)                                  │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  User: "运行测试"                                        │
│     ↓                                                    │
│  Main Agent: [等待...]                                   │
│     ↓                                                    │
│  Subagent: 执行 npm test (30 秒)                         │
│     ↓                                                    │
│  Main Agent: 收到结果                                    │
│     ↓                                                    │
│  User: 看到输出                                          │
│                                                          │
│  问题:                                                   │
│  ❌ 用户必须等待                                         │
│  ❌ 无法并行处理其他任务                                 │
│  ❌ 长时间任务阻塞交互                                   │
│                                                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  异步模式 (Asynchronous)                                 │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  User: "运行测试"                                        │
│     ↓                                                    │
│  Main Agent: 启动 Subagent (后台)                        │
│     ↓                                                    │
│  Main Agent: "测试正在后台运行..."                       │
│     ↓                                                    │
│  User: 立即可以继续其他工作                              │
│                                                          │
│  ┌──────────────────────────────┐                       │
│  │ Background Subagent          │                       │
│  │ 执行 npm test (30 秒)        │                       │
│  │ 完成后自动通知主代理         │                       │
│  └──────────────────────────────┘                       │
│                                                          │
│  Main Agent: 收到完成通知                                │
│     ↓                                                    │
│  User: "✓ 测试完成，所有测试通过"                        │
│                                                          │
│  优势:                                                   │
│  ✅ 非阻塞，用户可以继续工作                             │
│  ✅ 支持真正的并行任务                                   │
│  ✅ 适合长时间运行的操作                                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 启用后台运行

```bash
# 方式 1: 主动移至后台 (Ctrl+B)
User: "启动开发服务器"

Claude: [启动 Bash Subagent]
  Bash: npm run dev
  [输出: Server running on http://localhost:3000]

用户按 Ctrl+B
  → 子代理移至后台
  → 继续运行，输出重定向到文件

Main Agent: "开发服务器已移至后台，你可以继续其他工作"

User: "现在修改 Login 组件"
  → 主代理可以立即响应
  → 开发服务器继续运行

# 方式 2: 自动后台运行 (v2.0.64+)
Claude 自动检测长时间运行的任务并建议后台运行:

User: "运行所有 E2E 测试"

Claude: "E2E 测试可能需要 10-15 分钟
是否在后台运行？[Y/n]"

User: Y

Claude: [启动后台 Bash Subagent]
  "E2E 测试正在后台运行
   你会在完成时收到通知

   同时，你可以继续其他工作"

# 方式 3: 在任务创建时指定
TaskCreate({
  subject: "运行性能测试",
  description: "执行完整的性能基准测试",
  run_in_background: true,  // 明确指定后台运行
  agent_type: "bash"
})
```

### 后台任务生命周期

```
┌─────────────────────────────────────────────────────────┐
│          Background Task Lifecycle                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. 创建和启动                                           │
│  ┌────────────────────────────────────────────────┐     │
│  │  User 请求或 Claude 决定后台运行                │     │
│  │     ↓                                          │     │
│  │  创建 Background Subagent                       │     │
│  │     ├─ 分配唯一的 Task ID                      │     │
│  │     ├─ 创建输出文件 (output.log)               │     │
│  │     └─ 启动独立进程                            │     │
│  │                                                │     │
│  │  示例:                                         │     │
│  │    Task ID: bg_task_abc123                     │     │
│  │    Output: ~/.claude/tasks/bg_task_abc123/     │     │
│  │            output.log                          │     │
│  └────────────────────────────────────────────────┘     │
│                          ↓                               │
│  2. 运行中                                               │
│  ┌────────────────────────────────────────────────┐     │
│  │  后台进程独立运行                              │     │
│  │     ├─ 不阻塞主代理                            │     │
│  │     ├─ 输出实时写入日志文件                    │     │
│  │     └─ 定期更新状态                            │     │
│  │                                                │     │
│  │  用户可以:                                     │     │
│  │     ├─ 使用 /tasks 查看所有后台任务            │     │
│  │     ├─ 使用 tail -f output.log 监控输出        │     │
│  │     └─ 继续其他工作                            │     │
│  └────────────────────────────────────────────────┘     │
│                          ↓                               │
│  3. 完成和通知                                           │
│  ┌────────────────────────────────────────────────┐     │
│  │  后台任务完成                                  │     │
│  │     ↓                                          │     │
│  │  自动唤醒主代理                                │     │
│  │     ↓                                          │     │
│  │  Main Agent 通知用户:                          │     │
│  │  "✓ 后台任务完成: 运行性能测试                 │     │
│  │   结果: 所有基准测试通过                       │     │
│  │   详细日志: ~/.claude/tasks/.../output.log"    │     │
│  │                                                │     │
│  │  如果有失败:                                   │     │
│  │  "⚠️  后台任务失败: 运行性能测试               │     │
│  │   错误: 3 个测试超时                           │     │
│  │   查看日志获取详情"                            │     │
│  └────────────────────────────────────────────────┘     │
│                          ↓                               │
│  4. 清理                                                 │
│  ┌────────────────────────────────────────────────┐     │
│  │  保留输出日志 (可配置)                         │     │
│  │  释放进程资源                                  │     │
│  │  更新任务状态为 completed                      │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 4.2 TaskOutput 获取

### TaskOutput API

```typescript
// TaskOutput 数据结构

interface TaskOutput {
  task_id: string;          // 任务唯一标识
  status: TaskStatus;       // 任务状态
  output: {
    stdout: string;         // 标准输出
    stderr: string;         // 错误输出
    exitCode: number;       // 退出代码
  };
  metadata: {
    start_time: string;     // 开始时间
    end_time?: string;      // 结束时间（如果完成）
    duration?: number;      // 运行时长（毫秒）
    agent_type: string;     // 代理类型
    agent_id: string;       // 代理 ID
  };
  files_modified?: string[];  // 修改的文件列表
  error?: ErrorInfo;          // 错误信息（如果失败）
}

type TaskStatus =
  | "pending"      // 等待开始
  | "in_progress"  // 运行中
  | "completed"    // 成功完成
  | "failed"       // 失败
  | "cancelled";   // 已取消

interface ErrorInfo {
  message: string;
  stack?: string;
  code?: string;
}
```

### 获取任务输出的方式

```bash
# 方式 1: 自动推送（推荐）
# 后台任务完成时自动通知

Claude: "后台任务完成: 运行测试套件
✓ 单元测试: 245/245 通过
✓ 集成测试: 67/67 通过
⚠️ E2E 测试: 12/15 通过 (3 个失败)

失败的测试:
  - login.spec.ts:23 - 超时
  - checkout.spec.ts:45 - 断言失败
  - profile.spec.ts:12 - 元素未找到

查看完整日志: ~/.claude/tasks/task_xyz/output.log"

# 方式 2: 显式查询
User: "/tasks"

Output:
┌────────────────────────────────────────────────────┐
│ Background Tasks                                   │
├────────────────────────────────────────────────────┤
│ [1] ⏳ 运行 E2E 测试 (IN PROGRESS)                 │
│     Started: 2 min ago                             │
│     Agent: bash-subagent-3                         │
│     Output: ~/.claude/tasks/task_abc/output.log    │
│                                                    │
│ [2] ✓ 构建生产版本 (COMPLETED)                    │
│     Duration: 3m 24s                               │
│     Exit Code: 0                                   │
│                                                    │
│ [3] ⚠️ 部署到 staging (FAILED)                     │
│     Error: 权限拒绝                                │
│     Exit Code: 1                                   │
└────────────────────────────────────────────────────┘

User: "查看任务 1 的输出"

Claude: [使用 TaskGet 工具]
  TaskGet(task_id: "task_abc")

  Output:
  "E2E 测试进行中...
   [12:30:45] ✓ login.spec.ts (5/5 通过)
   [12:31:20] ⏳ checkout.spec.ts (运行中...)
   ..."

# 方式 3: 实时监控
User: "监控任务 1 的输出"

Claude: [启动监控 Bash Subagent]
  Bash: tail -f ~/.claude/tasks/task_abc/output.log

  实时输出:
  [12:32:10] ✓ checkout.spec.ts (8/8 通过)
  [12:32:45] ⏳ profile.spec.ts (运行中...)
  ...

用户按 Ctrl+C 停止监控

# 方式 4: 通过 Hook 自动处理
# .claude/hooks.json
{
  "PostTaskComplete": {
    "enabled": true,
    "script": "./scripts/notify-slack.sh",
    "args": ["${task_id}", "${status}", "${output}"]
  }
}

# 任务完成时自动发送 Slack 通知
```

### 高级输出处理

```bash
# 场景 1: 过滤和摘要
User: "摘要任务 1 的测试结果"

Claude: [读取并分析输出]

分析后的摘要:
┌────────────────────────────────────┐
│ E2E 测试结果摘要                   │
├────────────────────────────────────┤
│ 总测试: 45                         │
│ 通过: 42 (93%)                     │
│ 失败: 3 (7%)                       │
│ 跳过: 0                            │
│                                    │
│ 失败详情:                          │
│ 1. login.spec.ts:23                │
│    错误: 超时 (30s)                │
│    可能原因: API 响应慢            │
│                                    │
│ 2. checkout.spec.ts:45             │
│    错误: 期望 200, 实际 500        │
│    可能原因: 支付网关问题          │
│                                    │
│ 3. profile.spec.ts:12              │
│    错误: 元素 #avatar 未找到       │
│    可能原因: UI 结构变更           │
│                                    │
│ 建议:                              │
│ - 检查 API 性能                    │
│ - 调试支付集成                     │
│ - 更新 UI 选择器                   │
└────────────────────────────────────┘

# 场景 2: 错误提取和诊断
User: "为什么任务 3 失败了？"

Claude: [分析 stderr 和 exit code]

诊断结果:
错误类型: 权限错误
退出代码: 1 (EPERM)

详细错误:
  Error: EACCES: permission denied, mkdir '/var/www/app'
  at Object.mkdirSync (node:fs:1349:3)
  at deploy.js:45:8

根本原因:
  部署脚本尝试在 /var/www/app 创建目录
  但当前用户 (deploy-bot) 没有写权限

解决方案:
  1. 授予 deploy-bot 写权限:
     sudo chown -R deploy-bot:deploy-bot /var/www/app

  2. 或者使用 sudo 运行部署:
     sudo npm run deploy

  3. 或者修改部署目录为用户目录:
     DEPLOY_DIR=~/app npm run deploy

# 场景 3: 性能分析
User: "分析任务 2 的性能"

Claude: [解析构建日志]

性能分析:
┌────────────────────────────────────────┐
│ 构建性能分析                           │
├────────────────────────────────────────┤
│ 总时间: 3m 24s                         │
│                                        │
│ 阶段分解:                              │
│ 1. 依赖安装: 1m 12s (35%)              │
│ 2. TypeScript 编译: 1m 45s (51%)       │
│ 3. 资源打包: 27s (13%)                 │
│                                        │
│ 瓶颈:                                  │
│ ⚠️ TypeScript 编译耗时最长             │
│                                        │
│ 优化建议:                              │
│ 1. 启用增量编译 (tsconfig.json)       │
│    "incremental": true                 │
│    预计节省: ~40s                      │
│                                        │
│ 2. 使用 SWC 代替 tsc                   │
│    预计节省: ~60s                      │
│                                        │
│ 3. 并行构建 (使用 esbuild)             │
│    预计节省: ~30s                      │
│                                        │
│ 优化后预计总时间: ~1m 50s (46% 提升)  │
└────────────────────────────────────────┘
```

---

## 4.3 异步任务管理

### 任务状态机

```
任务状态转换:

     ┌─────────┐
     │ pending │ ← 创建时的初始状态
     └─────────┘
          │
          │ 开始执行
          ↓
   ┌──────────────┐
   │ in_progress  │ ← 正在运行
   └──────────────┘
          │
          ├─────────────┬─────────────┐
          │             │             │
          ↓             ↓             ↓
    ┌───────────┐  ┌─────────┐  ┌───────────┐
    │ completed │  │ failed  │  │ cancelled │
    └───────────┘  └─────────┘  └───────────┘
     (成功)         (失败)       (取消)

状态转换规则:

1. pending → in_progress
   触发: 任务开始执行
   条件: 所有 blockedBy 任务已完成

2. in_progress → completed
   触发: 任务成功完成
   条件: exitCode === 0

3. in_progress → failed
   触发: 任务失败
   条件: exitCode !== 0 或抛出异常

4. in_progress → cancelled
   触发: 用户取消任务
   条件: 用户显式操作

5. pending → cancelled
   触发: 用户取消等待中的任务
   条件: 用户显式操作

不允许的转换:
  ✗ completed → in_progress (不能重新运行)
  ✗ failed → completed (不能修改历史状态)
  ✗ cancelled → in_progress (不能恢复取消的任务)
```

### 任务管理操作

```bash
# 1. 列出所有任务
/tasks

Output:
┌──────────────────────────────────────────────────┐
│ All Tasks                                        │
├──────────────────────────────────────────────────┤
│ Active (3):                                      │
│ [1] ⏳ 运行 E2E 测试                             │
│ [2] ⏳ 监控开发服务器日志                        │
│ [3] ⏸  部署到 staging (BLOCKED BY: 1)           │
│                                                  │
│ Completed (5):                                   │
│ [4] ✓ 实现用户认证                              │
│ [5] ✓ 编写单元测试                              │
│ [6] ✓ 构建生产版本                              │
│ [7] ✓ 更新文档                                  │
│ [8] ✓ 代码审查                                  │
│                                                  │
│ Failed (1):                                      │
│ [9] ❌ 部署到生产 (权限错误)                     │
└──────────────────────────────────────────────────┘

# 2. 获取特定任务详情
User: "任务 1 的详情"

Claude: [TaskGet(id: "1")]

Output:
┌────────────────────────────────────────┐
│ Task #1: 运行 E2E 测试                 │
├────────────────────────────────────────┤
│ Status: in_progress                    │
│ Agent: bash-subagent-3                 │
│ Started: 5 min ago                     │
│ Estimated: ~10 min total               │
│                                        │
│ Progress:                              │
│ ▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░ 50%              │
│                                        │
│ Current Step:                          │
│ 运行 checkout.spec.ts                  │
│                                        │
│ Blocks:                                │
│ - Task #3 (部署到 staging)             │
│                                        │
│ Output File:                           │
│ ~/.claude/tasks/task_001/output.log    │
└────────────────────────────────────────┘

# 3. 取消任务
User: "取消任务 2"

Claude: [TaskUpdate(id: "2", status: "cancelled")]

Output:
"✓ 任务 2 已取消: 监控开发服务器日志
 后台进程已终止"

# 4. 重试失败的任务
User: "重试任务 9"

Claude: [创建新任务，复制配置]

Output:
"创建新任务 #10: 部署到生产
 已应用之前的配置
 修复建议:
 - 使用 sudo 或更新权限
 - 检查部署密钥

 是否立即开始? [Y/n]"

# 5. 批量操作
User: "取消所有等待中的任务"

Claude: [筛选 pending 状态的任务并取消]

Output:
"取消了 3 个等待中的任务:
 - Task #3: 部署到 staging
 - Task #11: 性能测试
 - Task #12: 生成文档"

# 6. 清理已完成的任务
User: "清理所有已完成的任务"

Claude:
"发现 5 个已完成的任务
 总计占用空间: 142 MB (日志文件)

 是否删除? [y/N]"

User: "y"

Claude:
"✓ 已清理 5 个任务
 释放空间: 142 MB

 保留的任务:
 - 最近 7 天的任务
 - 标记为重要的任务"
```

---

## 4.4 并行执行模式

### 并发限制和调度

```
Claude Code 并发配置:

┌─────────────────────────────────────────┐
│ 最大并发: 7 个子代理                    │
├─────────────────────────────────────────┤
│                                         │
│ Slot 1: [Explore #1]                    │
│ Slot 2: [Explore #2]                    │
│ Slot 3: [General #1]                    │
│ Slot 4: [General #2]                    │
│ Slot 5: [General #3]                    │
│ Slot 6: [Bash #1]                       │
│ Slot 7: [Bash #2]                       │
│                                         │
│ 等待队列:                               │
│ - [General #4] (等待 Slot 3-5)          │
│ - [Bash #3] (等待 Slot 6-7)             │
│                                         │
└─────────────────────────────────────────┘

调度策略:

1. 优先级调度
   高优先级任务优先分配 slot

2. 类型亲和性
   相同类型的任务倾向于连续运行
   (减少上下文切换)

3. 依赖优先
   被其他任务依赖的任务优先执行

4. 公平调度
   长时间等待的任务提升优先级
```

### 并行模式示例

#### 模式 1: 构建流水线

```
并行构建多个目标:

┌────────────────────────────────────────────────────────┐
│  Build Pipeline (并行)                                 │
├────────────────────────────────────────────────────────┤
│                                                         │
│  准备阶段:                                              │
│  ┌──────────────────────────────────────┐              │
│  │ Task 0: 安装依赖 (Bash #1)           │              │
│  │ npm install                          │              │
│  └──────────────────────────────────────┘              │
│                   ↓                                    │
│  ┌─────────────────────┬──────────────────────────┐   │
│  │                     │                          │   │
│  ↓                     ↓                          ↓   │
│  并行阶段 1:                                            │
│  ┌───────────┐  ┌───────────┐  ┌──────────────┐       │
│  │ Task 1    │  │ Task 2    │  │ Task 3       │       │
│  │ Lint      │  │ Type Check│  │ Unit Tests   │       │
│  │ Bash #2   │  │ Bash #3   │  │ Bash #4      │       │
│  └───────────┘  └───────────┘  └──────────────┘       │
│       │              │               │                 │
│       └──────────────┴───────────────┘                 │
│                      ↓                                 │
│  并行阶段 2:                                            │
│  ┌──────────────┐  ┌──────────────┐                   │
│  │ Task 4       │  │ Task 5       │                   │
│  │ Build Web    │  │ Build Mobile │                   │
│  │ Bash #2      │  │ Bash #3      │                   │
│  └──────────────┘  └──────────────┘                   │
│       │                   │                            │
│       └───────────────────┘                            │
│                   ↓                                    │
│  最终阶段:                                              │
│  ┌──────────────────────────────────────┐              │
│  │ Task 6: 打包发布 (Bash #2)           │              │
│  │ npm run package                      │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  总时间: 8 分钟                                         │
│  (串行需要: 25 分钟)                                    │
│  提速: 3.1倍 ⚡                                         │
│                                                         │
└────────────────────────────────────────────────────────┘

代码实现:

// 任务定义
const tasks = [
  {
    id: "0",
    name: "安装依赖",
    command: "npm install",
    depends_on: []
  },
  {
    id: "1",
    name: "Lint",
    command: "npm run lint",
    depends_on: ["0"]
  },
  {
    id: "2",
    name: "Type Check",
    command: "npm run typecheck",
    depends_on: ["0"]
  },
  {
    id: "3",
    name: "Unit Tests",
    command: "npm test",
    depends_on: ["0"]
  },
  {
    id: "4",
    name: "Build Web",
    command: "npm run build:web",
    depends_on: ["1", "2", "3"]
  },
  {
    id: "5",
    name: "Build Mobile",
    command: "npm run build:mobile",
    depends_on: ["1", "2", "3"]
  },
  {
    id: "6",
    name: "Package",
    command: "npm run package",
    depends_on: ["4", "5"]
  }
];

// Claude 自动并行化
User: "执行完整的构建流水线"

Claude: [分析依赖图，创建并行任务]
  - Task 0 立即开始
  - Task 0 完成后，Tasks 1-3 并行开始
  - Tasks 1-3 全部完成后，Tasks 4-5 并行开始
  - Tasks 4-5 完成后，Task 6 开始
```

#### 模式 2: 监控 + 开发

```
后台监控 + 前台开发:

┌────────────────────────────────────────────────────────┐
│  Developer Workflow                                    │
├────────────────────────────────────────────────────────┤
│                                                         │
│  后台任务 (持续运行):                                   │
│  ┌──────────────────────────────────────┐              │
│  │ Task 1: 开发服务器 (Bash #1)         │              │
│  │ npm run dev                          │              │
│  │ (后台运行，Ctrl+B)                   │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  ┌──────────────────────────────────────┐              │
│  │ Task 2: 监控测试 (Bash #2)           │              │
│  │ npm run test:watch                   │              │
│  │ (后台运行，Ctrl+B)                   │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  ┌──────────────────────────────────────┐              │
│  │ Task 3: Tail 日志 (Bash #3)          │              │
│  │ tail -f logs/app.log                 │              │
│  │ (后台运行，Ctrl+B)                   │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  前台任务 (交互式):                                     │
│  ┌──────────────────────────────────────┐              │
│  │ Main Agent: 响应用户请求             │              │
│  │                                      │              │
│  │ User: "实现用户个人资料页面"         │              │
│  │   ↓                                  │              │
│  │ General #1: 实现功能                 │              │
│  │   ├─ 创建 Profile.tsx                │              │
│  │   ├─ 创建 API 端点                   │              │
│  │   └─ 更新路由                        │              │
│  │                                      │              │
│  │ (实时看到开发服务器热更新)           │              │
│  │ (自动运行相关测试)                   │              │
│  │ (日志中看到 API 调用)                │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  优势:                                                  │
│  ✅ 实时反馈                                            │
│  ✅ 自动化测试                                          │
│  ✅ 完整的可观测性                                      │
│  ✅ 不中断工作流                                        │
│                                                         │
└────────────────────────────────────────────────────────┘
```

---

## 4.5 实战场景

### 场景 1: 大规模代码迁移

```
任务: 将 100+ 个组件从 JavaScript 迁移到 TypeScript

策略: 并行处理 + 批量任务

┌────────────────────────────────────────┐
│ Step 1: 分析和分组                     │
├────────────────────────────────────────┤
│ Explore Subagent:                      │
│   ├─ 找到所有 .js/.jsx 组件 (124 个)  │
│   ├─ 分析依赖关系                      │
│   └─ 按依赖层级分组:                   │
│       - 叶子组件 (无依赖): 85 个       │
│       - 中间组件: 30 个                │
│       - 根组件: 9 个                   │
└────────────────────────────────────────┘
                  ↓
┌────────────────────────────────────────┐
│ Step 2: 批量迁移叶子组件 (并行)       │
├────────────────────────────────────────┤
│ 创建 7 个并行任务:                     │
│                                        │
│ General #1: 迁移组件 1-12              │
│ General #2: 迁移组件 13-24             │
│ General #3: 迁移组件 25-36             │
│ General #4: 迁移组件 37-48             │
│ General #5: 迁移组件 49-60             │
│ General #6: 迁移组件 61-72             │
│ General #7: 迁移组件 73-85             │
│                                        │
│ 每个子代理执行:                        │
│   对于每个组件:                        │
│     1. 重命名 .js → .ts                │
│     2. 添加类型注解                    │
│     3. 修复类型错误                    │
│     4. 运行 tsc --noEmit 验证          │
│     5. 运行单元测试                    │
│                                        │
│ 预计时间: ~15 分钟                     │
│ (串行需要: ~90 分钟)                   │
└────────────────────────────────────────┘
                  ↓
┌────────────────────────────────────────┐
│ Step 3: 迁移中间组件 (并行)           │
├────────────────────────────────────────┤
│ 等待 Step 2 完成后开始                 │
│                                        │
│ General #1: 迁移组件 1-5               │
│ General #2: 迁移组件 6-10              │
│ ...                                    │
│                                        │
│ 预计时间: ~8 分钟                      │
└────────────────────────────────────────┘
                  ↓
┌────────────────────────────────────────┐
│ Step 4: 迁移根组件 (串行)             │
├────────────────────────────────────────┤
│ 需要仔细处理，逐个迁移                 │
│                                        │
│ General #1: 迁移 App.js                │
│ General #1: 迁移 Router.js             │
│ ...                                    │
│                                        │
│ 预计时间: ~12 分钟                     │
└────────────────────────────────────────┘
                  ↓
┌────────────────────────────────────────┐
│ Step 5: 最终验证                       │
├────────────────────────────────────────┤
│ Bash #1: 运行完整的类型检查            │
│ Bash #2: 运行所有测试                  │
│ Bash #3: 构建生产版本                  │
│                                        │
│ 预计时间: ~5 分钟                      │
└────────────────────────────────────────┘

总时间: ~40 分钟
串行总时间: ~220 分钟
提速: 5.5 倍 ⚡

实际执行:

User: "将所有 JS 组件迁移到 TypeScript"

Claude: [启动 Plan Mode]
  "分析了 124 个组件
   生成了迁移计划:
   - 4 个并行阶段
   - 预计 40 分钟完成

   是否开始? [Y/n]"

User: "Y"

Claude: [创建任务图，开始执行]
  "✓ 阶段 1 完成: 85 个叶子组件 (15分钟)
   ⏳ 阶段 2 进行中: 中间组件 (3/30 完成)

   所有任务在后台运行
   你可以继续其他工作"
```

### 场景 2: CI/CD 流水线

```
完整的 CI/CD 流程:

┌────────────────────────────────────────────────────────┐
│  Continuous Integration & Deployment                   │
├────────────────────────────────────────────────────────┤
│                                                         │
│  触发: Git push to main branch                          │
│                                                         │
│  Stage 1: 代码质量检查 (并行)                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Lint     │  │ Format   │  │ Security │             │
│  │ Bash #1  │  │ Bash #2  │  │ Bash #3  │             │
│  │ 2 min    │  │ 1 min    │  │ 3 min    │             │
│  └──────────┘  └──────────┘  └──────────┘             │
│       │              │              │                  │
│       └──────────────┴──────────────┘                  │
│                      ↓                                 │
│  Stage 2: 测试 (并行)                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Unit     │  │Integration│ │ E2E      │             │
│  │ Bash #1  │  │ Bash #2  │  │ Bash #3  │             │
│  │ 5 min    │  │ 8 min    │  │ 12 min   │             │
│  └──────────┘  └──────────┘  └──────────┘             │
│       │              │              │                  │
│       └──────────────┴──────────────┘                  │
│                      ↓                                 │
│  Stage 3: 构建 (并行)                                   │
│  ┌──────────────┐  ┌──────────────┐                   │
│  │ Build Web    │  │ Build API    │                   │
│  │ Bash #1      │  │ Bash #2      │                   │
│  │ 4 min        │  │ 3 min        │                   │
│  └──────────────┘  └──────────────┘                   │
│       │                   │                            │
│       └───────────────────┘                            │
│                   ↓                                    │
│  Stage 4: 部署 (串行)                                   │
│  ┌──────────────────────────────────────┐              │
│  │ Deploy to Staging (Bash #1)          │              │
│  │ 2 min                                │              │
│  └──────────────────────────────────────┘              │
│                   ↓                                    │
│  ┌──────────────────────────────────────┐              │
│  │ Smoke Tests (Bash #2)                │              │
│  │ 3 min                                │              │
│  └──────────────────────────────────────┘              │
│                   ↓                                    │
│  ┌──────────────────────────────────────┐              │
│  │ Deploy to Production (Bash #1)       │              │
│  │ 需要手动批准                         │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  总时间: ~30 分钟                                       │
│  (串行需要: ~60 分钟)                                   │
│                                                         │
└────────────────────────────────────────────────────────┘

自动化脚本:

# .claude/workflows/ci-cd.yaml
name: "CI/CD Pipeline"

on:
  push:
    branches: [main]

stages:
  - name: "Quality Checks"
    parallel: true
    tasks:
      - name: "Lint"
        agent: bash
        command: "npm run lint"

      - name: "Format Check"
        agent: bash
        command: "npm run format:check"

      - name: "Security Scan"
        agent: bash
        command: "npm audit && npm run security:scan"

  - name: "Tests"
    parallel: true
    depends_on: ["Quality Checks"]
    tasks:
      - name: "Unit Tests"
        agent: bash
        command: "npm run test:unit"

      - name: "Integration Tests"
        agent: bash
        command: "npm run test:integration"

      - name: "E2E Tests"
        agent: bash
        command: "npm run test:e2e"
        background: true  # 后台运行

  - name: "Build"
    parallel: true
    depends_on: ["Tests"]
    tasks:
      - name: "Build Web"
        agent: bash
        command: "npm run build:web"

      - name: "Build API"
        agent: bash
        command: "npm run build:api"

  - name: "Deploy Staging"
    depends_on: ["Build"]
    tasks:
      - name: "Deploy"
        agent: bash
        command: "./scripts/deploy-staging.sh"

      - name: "Smoke Tests"
        agent: bash
        command: "npm run test:smoke -- --env=staging"

  - name: "Deploy Production"
    depends_on: ["Deploy Staging"]
    requires_approval: true
    tasks:
      - name: "Deploy"
        agent: bash
        command: "./scripts/deploy-production.sh"

触发流程:

User: "运行 CI/CD 流水线"

Claude: [加载 workflow，创建任务图]
  "创建了 11 个任务
   4 个阶段 (部分并行)
   预计 30 分钟完成

   开始执行..."

  [15 分钟后]
  "✓ Stage 1 完成: 所有质量检查通过
   ✓ Stage 2 完成: 所有测试通过 (234/234)
   ⏳ Stage 3 进行中: 构建 (Web 完成, API 95%)

   所有任务在后台运行
   你会在需要批准时收到通知"

  [28 分钟后]
  "✓ Staging 部署成功
   Smoke tests 通过

   ⏸  等待批准: 部署到生产环境

   Staging URL: https://staging.example.com
   请验证后批准部署

   批准? [Y/n/详情]"

User: "Y"

Claude:
  "✓ 开始生产部署...
   ✓ 部署完成

   Production URL: https://example.com

   CI/CD 流程完成 ✓
   总时间: 32 分钟"
```

---

## 下一步

- 学习 [高级特性](./05-advanced-features.md)，掌握任务系统和 Skills
- 查看 [最佳实践](./08-best-practices.md)，优化异步工作流
- 参考 [Hook 系统](./06-hook-system.md)，自动化任务管理

---

**参考资源**:
- [Claude Code Async Workflows](https://claudefa.st/blog/guide/agents/async-workflows)
- [Claude Code Tasks Update](https://venturebeat.com/orchestration/claude-codes-tasks-update-lets-agents-work-longer-and-coordinate-across/)
- [任务管理实践](https://claudefa.st/blog/guide/development/task-management)
