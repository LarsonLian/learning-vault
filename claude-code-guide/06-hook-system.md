# 06. Hook 系统（工作流自动化）

## 目录
- [6.1 Hook 类型和触发时机](#61-hook-类型和触发时机)
- [6.2 Hook 配置](#62-hook-配置)
- [6.3 实际应用场景](#63-实际应用场景)
- [6.4 高级 Hook 模式](#64-高级-hook-模式)

---

## 6.1 Hook 类型和触发时机

### Hook 完整列表

```
┌─────────────────────────────────────────────────────────┐
│              Claude Code Hook System                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  会话生命周期 (Session Lifecycle)                         │
│  ┌────────────────────────────────────────────────┐     │
│  │ OnSessionStart                                 │     │
│  │ 触发: Claude Code 启动时                       │     │
│  │ 用途:                                          │     │
│  │   - 加载项目上下文                             │     │
│  │   - 环境检查                                   │     │
│  │   - 显示待办事项                               │     │
│  │                                                │     │
│  │ OnSessionEnd                                   │     │
│  │ 触发: 会话结束时                               │     │
│  │ 用途:                                          │     │
│  │   - 清理临时文件                               │     │
│  │   - 保存会话摘要                               │     │
│  │   - 发送通知                                   │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  工具执行 (Tool Execution)                               │
│  ┌────────────────────────────────────────────────┐     │
│  │ PreToolUse                                     │     │
│  │ 触发: 任何工具执行前                           │     │
│  │ 用途:                                          │     │
│  │   - 权限验证                                   │     │
│  │   - 参数检查                                   │     │
│  │   - 审计日志                                   │     │
│  │   - 上下文注入                                 │     │
│  │                                                │     │
│  │ PostToolUse                                    │     │
│  │ 触发: 工具执行成功后                           │     │
│  │ 用途:                                          │     │
│  │   - 结果验证                                   │     │
│  │   - 自动化测试                                 │     │
│  │   - 性能记录                                   │     │
│  │   - 通知发送                                   │     │
│  │                                                │     │
│  │ ToolError                                      │     │
│  │ 触发: 工具执行失败时                           │     │
│  │ 用途:                                          │     │
│  │   - 错误处理                                   │     │
│  │   - 自动重试                                   │     │
│  │   - 告警通知                                   │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  文件操作 (File Operations)                              │
│  ┌────────────────────────────────────────────────┐     │
│  │ PreFileWrite                                   │     │
│  │ 触发: 文件写入前                               │     │
│  │ 用途:                                          │     │
│  │   - 代码格式化 (Prettier)                      │     │
│  │   - Lint 检查                                  │     │
│  │   - 备份原文件                                 │     │
│  │                                                │     │
│  │ PostFileWrite                                  │     │
│  │ 触发: 文件写入后                               │     │
│  │ 用途:                                          │     │
│  │   - 自动运行测试                               │     │
│  │   - Git 自动提交                               │     │
│  │   - 热更新触发                                 │     │
│  │                                                │     │
│  │ FileConflict                                   │     │
│  │ 触发: 文件冲突时                               │     │
│  │ 用途:                                          │     │
│  │   - 冲突解决策略                               │     │
│  │   - 手动确认提示                               │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  命令执行 (Command Execution)                            │
│  ┌────────────────────────────────────────────────┐     │
│  │ PreBashRun                                     │     │
│  │ 触发: Bash 命令执行前                          │     │
│  │ 用途:                                          │     │
│  │   - 危险命令拦截                               │     │
│  │   - 参数注入检测                               │     │
│  │   - 权限检查                                   │     │
│  │                                                │     │
│  │ PostBashRun                                    │     │
│  │ 触发: Bash 命令执行后                          │     │
│  │ 用途:                                          │     │
│  │   - 命令审计                                   │     │
│  │   - 性能监控                                   │     │
│  │   - 结果缓存                                   │     │
│  │                                                │     │
│  │ BashError                                      │     │
│  │ 触发: 命令执行失败                             │     │
│  │ 用途:                                          │     │
│  │   - 错误诊断                                   │     │
│  │   - 自动修复尝试                               │     │
│  │   - 告警通知                                   │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  任务管理 (Task Management)                              │
│  ┌────────────────────────────────────────────────┐     │
│  │ TaskCreated                                    │     │
│  │ 触发: 创建新任务时                             │     │
│  │                                                │     │
│  │ TaskStarted                                    │     │
│  │ 触发: 任务开始执行                             │     │
│  │                                                │     │
│  │ TaskCompleted                                  │     │
│  │ 触发: 任务成功完成                             │     │
│  │ 用途:                                          │     │
│  │   - Slack 通知                                 │     │
│  │   - 更新项目看板                               │     │
│  │   - 触发下一阶段                               │     │
│  │                                                │     │
│  │ TaskFailed                                     │     │
│  │ 触发: 任务失败                                 │     │
│  │ 用途:                                          │     │
│  │   - 错误通知                                   │     │
│  │   - 自动创建issue                              │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  权限事件 (Permission Events)                            │
│  ┌────────────────────────────────────────────────┐     │
│  │ PermissionRequired                             │     │
│  │ 触发: 需要权限确认时                           │     │
│  │ 用途:                                          │     │
│  │   - 自动批准策略                               │     │
│  │   - 审批流程                                   │     │
│  │   - 权限日志                                   │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Hook 执行流程

```
工具调用示例 (Write tool):

User: "创建 README.md"
    ↓
Claude: 决定使用 Write tool
    ↓
┌──────────────────────────────────┐
│ 1. PreToolUse Hook               │
│ ├─ 检查文件是否存在              │
│ ├─ 验证权限                      │
│ └─ 记录审计日志                  │
│ 结果: ALLOW/DENY/MODIFY          │
└──────────────────────────────────┘
    │ (if ALLOW)
    ↓
┌──────────────────────────────────┐
│ 2. PreFileWrite Hook             │
│ ├─ 运行 Prettier 格式化          │
│ ├─ Lint 检查                     │
│ └─ 备份现有文件                  │
│ 结果: CONTINUE/BLOCK             │
└──────────────────────────────────┘
    │ (if CONTINUE)
    ↓
┌──────────────────────────────────┐
│ 3. 实际写入文件                  │
│ Write tool 执行                  │
└──────────────────────────────────┘
    │
    ↓
┌──────────────────────────────────┐
│ 4. PostFileWrite Hook            │
│ ├─ 运行相关测试                  │
│ ├─ Git add 文件                  │
│ └─ 触发热更新                    │
└──────────────────────────────────┘
    │
    ↓
┌──────────────────────────────────┐
│ 5. PostToolUse Hook              │
│ ├─ 更新文件缓存                  │
│ ├─ 发送完成通知                  │
│ └─ 记录性能指标                  │
└──────────────────────────────────┘
    │
    ↓
返回结果给 Claude
```

---

## 6.2 Hook 配置

### 配置文件结构

```
Hook 配置的三层优先级:

1. Local Project (.claude/hooks.json)      ← 最高优先级
   └─ 项目特定的 hook

2. Project Root (.claude-config.json)
   └─ 团队共享的 hook (通过 Git)

3. User Global (~/.claude/config.json)     ← 最低优先级
   └─ 用户全局的 hook
```

### 基础配置示例

```json
// .claude/hooks.json

{
  "hooks": {
    // 会话钩子
    "OnSessionStart": {
      "enabled": true,
      "command": "./scripts/session-start.sh",
      "description": "加载项目上下文和检查环境"
    },

    "OnSessionEnd": {
      "enabled": true,
      "command": "./scripts/session-end.sh",
      "description": "清理和保存会话"
    },

    // 文件操作钩子
    "PreFileWrite": {
      "enabled": true,
      "rules": [
        {
          "pattern": "*.{js,ts,jsx,tsx}",
          "command": "prettier --write {file}",
          "failureAction": "warn",
          "description": "自动格式化 JS/TS 文件"
        },
        {
          "pattern": "*.{js,ts}",
          "command": "eslint --fix {file}",
          "failureAction": "block",
          "description": "Lint 检查并修复"
        },
        {
          "pattern": "*.py",
          "command": "black --check {file}",
          "failureAction": "block",
          "description": "Python 代码格式检查"
        }
      ]
    },

    "PostFileWrite": {
      "enabled": true,
      "rules": [
        {
          "pattern": "src/**/*.test.{js,ts}",
          "command": "npm test -- {file}",
          "async": true,
          "description": "运行相关测试"
        },
        {
          "pattern": "**/*.{js,ts,tsx}",
          "command": "npm run typecheck",
          "async": true,
          "description": "类型检查"
        }
      ]
    },

    // 命令执行钩子
    "PreBashRun": {
      "enabled": true,
      "rules": [
        {
          "pattern": "rm -rf *",
          "action": "block",
          "message": "🛑 禁止使用 rm -rf *",
          "description": "防止意外删除"
        },
        {
          "pattern": "git push --force*",
          "action": "ask",
          "message": "⚠️  强制推送需要确认",
          "description": "防止覆盖远程历史"
        },
        {
          "pattern": "npm install *",
          "command": "./scripts/check-package.sh {args}",
          "description": "检查包的安全性"
        }
      ]
    },

    "PostBashRun": {
      "enabled": true,
      "command": "echo '[{timestamp}] {command}' >> .claude/audit.log",
      "description": "审计所有命令"
    },

    // 任务钩子
    "TaskCompleted": {
      "enabled": true,
      "command": "./scripts/notify-slack.sh '{task_id}' '{subject}' 'completed'",
      "description": "Slack 通知"
    },

    "TaskFailed": {
      "enabled": true,
      "command": "./scripts/create-issue.sh '{task_id}' '{error}'",
      "description": "自动创建 GitHub Issue"
    }
  },

  // Hook 全局配置
  "hookConfig": {
    "timeout": 30000,           // 默认超时 30秒
    "retryOnFailure": false,    // 失败后不重试
    "captureOutput": true,      // 捕获输出
    "env": {                    // 环境变量
      "CLAUDE_SESSION_ID": "{session_id}",
      "CLAUDE_PROJECT_ROOT": "{project_root}"
    }
  }
}
```

### 高级配置选项

```json
{
  "hooks": {
    "PreFileWrite": {
      "enabled": true,
      "rules": [
        {
          "pattern": "src/**/*.ts",

          // 命令配置
          "command": "eslint --fix {file}",
          "workingDirectory": "{project_root}",
          "timeout": 10000,
          "env": {
            "NODE_ENV": "development"
          },

          // 失败处理
          "failureAction": "block",  // block | warn | ignore | ask
          "onFailure": {
            "command": "./scripts/handle-lint-error.sh {file}",
            "retryCount": 2,
            "retryDelay": 1000
          },

          // 条件执行
          "conditions": {
            "fileSize": "<1MB",      // 文件大小限制
            "gitStatus": "modified",  // 仅修改的文件
            "timeRange": "09:00-18:00"  // 工作时间
          },

          // 异步执行
          "async": false,  // 同步执行（阻塞）
          "background": false,  // 后台运行

          // 缓存
          "cache": {
            "enabled": true,
            "ttl": 3600,  // 1小时
            "key": "{file_hash}"
          },

          // 元数据
          "description": "ESLint 检查和自动修复",
          "tags": ["quality", "linting"],
          "priority": 10  // 高优先级
        }
      ]
    },

    // 自定义 Hook
    "CustomHook": {
      "enabled": true,
      "trigger": {
        "event": "FileWrite",
        "filter": {
          "pattern": "package.json",
          "operation": "write"
        }
      },
      "actions": [
        {
          "name": "检测依赖变化",
          "command": "git diff package.json"
        },
        {
          "name": "更新 lock 文件",
          "command": "npm install",
          "condition": "diff_exists"
        },
        {
          "name": "安全审计",
          "command": "npm audit",
          "async": true
        }
      ]
    }
  }
}
```

---

## 6.3 实际应用场景

### 场景 1: 代码质量保证

```json
// .claude/hooks.json
{
  "hooks": {
    "PreFileWrite": {
      "enabled": true,
      "rules": [
        {
          "pattern": "**/*.{js,ts,jsx,tsx}",
          "command": "prettier --check {file}",
          "failureAction": "block",
          "description": "强制代码格式化"
        },
        {
          "pattern": "**/*.{js,ts}",
          "command": "eslint {file}",
          "failureAction": "block",
          "description": "Lint 检查"
        }
      ]
    },

    "PostFileWrite": {
      "enabled": true,
      "rules": [
        {
          "pattern": "src/**/*.{js,ts}",
          "command": "npm run test:related -- {file}",
          "async": true,
          "failureAction": "warn",
          "description": "运行相关测试"
        }
      ]
    }
  }
}
```

**效果**:
```
User: "创建 src/utils/helper.ts"

Claude: [尝试写入文件]
    ↓
PreFileWrite Hook:
  ├─ ✓ Prettier 格式检查通过
  └─ ✓ ESLint 检查通过
    ↓
Write File ✓
    ↓
PostFileWrite Hook:
  └─ ⏳ 运行相关测试 (后台)
    ↓
Claude: "文件已创建: src/utils/helper.ts
         ✓ 代码格式正确
         ✓ Lint 检查通过
         ⏳ 测试正在后台运行..."

[30秒后]
Claude: "✓ 所有相关测试通过 (12/12)"
```

### 场景 2: 安全防护

```json
{
  "hooks": {
    "PreBashRun": {
      "enabled": true,
      "rules": [
        {
          "pattern": "rm -rf *",
          "action": "block",
          "message": "🛑 禁止使用 rm -rf *\n建议: 指定具体目录",
          "description": "防止意外删除"
        },
        {
          "pattern": "chmod 777 *",
          "action": "block",
          "message": "🛑 禁止 chmod 777\n安全风险: 过于宽松的权限",
          "description": "防止权限滥用"
        },
        {
          "pattern": "git push --force*",
          "action": "ask",
          "message": "⚠️  强制推送会覆盖远程历史\n确认要继续吗?",
          "description": "防止覆盖历史"
        },
        {
          "pattern": "curl * | bash",
          "action": "block",
          "message": "🛑 禁止管道执行未知脚本\n安全风险: 代码注入",
          "description": "防止代码注入"
        },
        {
          "pattern": "sudo *",
          "action": "ask",
          "message": "需要 sudo 权限\n命令: {command}\n确认执行?",
          "description": "sudo 命令需要确认"
        }
      ]
    },

    "PostBashRun": {
      "enabled": true,
      "command": "./scripts/audit-command.sh '{command}' '{exit_code}' '{user}'",
      "description": "审计所有命令"
    }
  }
}
```

**效果**:
```
User: "删除 node_modules"

Claude: [准备执行 rm -rf node_modules]
    ↓
PreBashRun Hook:
  检测到: rm -rf *
  ↓
❌ 命令被拦截

Claude: "🛑 命令被安全策略阻止
         命令: rm -rf node_modules
         原因: 匹配危险模式 'rm -rf *'

         建议替代方案:
         1. rm -rf ./node_modules/  (添加 ./ 前缀)
         2. 使用更安全的 rimraf:
            npx rimraf node_modules

         使用哪种方案? [1/2]"
```

### 场景 3: 自动化测试

```json
{
  "hooks": {
    "PostFileWrite": {
      "enabled": true,
      "rules": [
        {
          "pattern": "src/**/*.{js,ts}",
          "command": "npm run test:changed",
          "async": true,
          "description": "测试修改的文件"
        },
        {
          "pattern": "src/**/*.test.{js,ts}",
          "command": "npm test -- {file}",
          "async": false,
          "failureAction": "warn",
          "description": "立即运行新测试"
        }
      ]
    },

    "TaskCompleted": {
      "enabled": true,
      "rules": [
        {
          "filter": {
            "tags": ["feature"]
          },
          "command": "npm run test:integration",
          "async": true,
          "description": "功能完成后运行集成测试"
        }
      ]
    }
  }
}
```

**工作流**:
```
User: "实现用户登录功能"

Claude: [完成实现]
  ├─ Write: src/auth/login.ts
  ├─ Write: src/auth/login.test.ts
  └─ TaskUpdate: status = completed

Hooks 触发:

PostFileWrite (login.ts):
  └─ ⏳ npm run test:changed (后台)

PostFileWrite (login.test.ts):
  └─ ⏳ npm test -- login.test.ts (前台)
      ✓ 5 tests passed

TaskCompleted (feature tag):
  └─ ⏳ npm run test:integration (后台)

Claude: "✓ 登录功能已实现
         ✓ 单元测试通过 (5/5)
         ⏳ 集成测试运行中..."

[2分钟后]
Claude: "✓ 集成测试通过 (12/12)
         所有测试完成 ✓"
```

### 场景 4: Git 工作流自动化

```json
{
  "hooks": {
    "PostFileWrite": {
      "enabled": true,
      "rules": [
        {
          "pattern": "src/**/*",
          "command": "git add {file}",
          "description": "自动 stage 修改的文件"
        }
      ]
    },

    "TaskCompleted": {
      "enabled": true,
      "command": "./scripts/auto-commit.sh '{task_id}' '{subject}'",
      "description": "任务完成后自动提交"
    },

    "OnSessionEnd": {
      "enabled": true,
      "command": "./scripts/session-summary.sh",
      "description": "生成会话摘要并提交"
    }
  }
}
```

**自动提交脚本** (`scripts/auto-commit.sh`):
```bash
#!/bin/bash
set -e

TASK_ID=$1
SUBJECT=$2

# 检查是否有暂存的更改
if [[ -z $(git diff --cached --name-only) ]]; then
  echo "没有文件需要提交"
  exit 0
fi

# 生成提交消息
COMMIT_MSG=$(cat <<EOF
feat: ${SUBJECT}

Task ID: ${TASK_ID}
Files changed: $(git diff --cached --name-only | wc -l)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)

# 提交
git commit -m "$COMMIT_MSG"

echo "✓ 自动提交完成"
echo "✓ 提交消息:"
echo "$COMMIT_MSG"

# 可选: 自动推送到远程分支
# git push origin HEAD
```

**效果**:
```
User: "实现用户仪表板"

Claude: [完成多个文件的修改]
    ├─ Write: src/pages/Dashboard.tsx
    ├─ Write: src/components/UserStats.tsx
    ├─ Write: src/api/dashboard.ts
    └─ TaskUpdate: completed

每个文件写入后:
  PostFileWrite Hook:
    git add <file> ✓

任务完成后:
  TaskCompleted Hook:
    ./scripts/auto-commit.sh 执行

Git 操作:
  ✓ 3 files staged
  ✓ Commit created:
    "feat: 实现用户仪表板

     Task ID: task-123
     Files changed: 3

     Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"

Claude: "✓ 任务完成
         ✓ 文件已自动提交
         ✓ Commit: a3f2b1c"
```

---

## 6.4 高级 Hook 模式

### 模式 1: Hook 链

```json
{
  "hooks": {
    "PreFileWrite": {
      "enabled": true,
      "chain": [
        {
          "name": "backup",
          "command": "cp {file} {file}.backup",
          "description": "备份原文件"
        },
        {
          "name": "format",
          "command": "prettier --write {file}",
          "description": "格式化"
        },
        {
          "name": "lint",
          "command": "eslint --fix {file}",
          "description": "Lint 并修复",
          "onFailure": "rollback"  // 失败时回滚
        },
        {
          "name": "verify",
          "command": "node --check {file}",
          "description": "语法检查",
          "onFailure": "rollback"
        }
      ],
      "onChainFailure": {
        "rollback": {
          "command": "mv {file}.backup {file}",
          "description": "恢复备份"
        },
        "cleanup": {
          "command": "rm -f {file}.backup"
        }
      }
    }
  }
}
```

### 模式 2: 条件 Hook

```json
{
  "hooks": {
    "PostFileWrite": {
      "enabled": true,
      "rules": [
        {
          "pattern": "**/*.ts",
          "conditions": {
            // 仅在工作时间运行慢速检查
            "timeRange": "09:00-18:00",
            "command": "npm run test:full",
            "description": "完整测试套件"
          },
          "else": {
            // 非工作时间只运行快速检查
            "command": "npm run test:quick",
            "description": "快速测试"
          }
        },
        {
          "pattern": "src/**/*.ts",
          "conditions": {
            // 根据文件大小决定策略
            "fileSize": ">100KB",
            "command": "echo '大文件，跳过某些检查'",
            "skip": ["complexity-check"]
          }
        }
      ]
    }
  }
}
```

### 模式 3: 上下文感知 Hook

```json
{
  "hooks": {
    "PreBashRun": {
      "enabled": true,
      "contextAware": true,
      "rules": [
        {
          "pattern": "npm install *",
          "checks": [
            {
              "name": "package-lock-check",
              "script": "./scripts/check-lock.sh",
              "description": "检查 package-lock.json 状态"
            },
            {
              "name": "disk-space-check",
              "script": "df -h | grep '/$' | awk '{print $5}' | sed 's/%//'",
              "threshold": 90,
              "message": "磁盘空间不足 (<10% 可用)",
              "action": "block"
            },
            {
              "name": "network-check",
              "script": "ping -c 1 registry.npmjs.org",
              "timeout": 5000,
              "message": "无法连接到 npm registry",
              "action": "warn"
            }
          ]
        }
      ]
    }
  }
}
```

### 模式 4: 动态 Hook 生成

```javascript
// .claude/hooks.js (动态配置)

module.exports = {
  hooks: {
    PreFileWrite: {
      enabled: true,
      // 动态生成规则
      rules: generateRulesForFileTypes([
        'js', 'ts', 'jsx', 'tsx',
        'py', 'rb', 'go', 'rust'
      ])
    }
  }
};

function generateRulesForFileTypes(types) {
  return types.map(type => ({
    pattern: `**/*.${type}`,
    command: getFormatterForType(type),
    failureAction: 'block',
    description: `Format ${type.toUpperCase()} files`
  }));
}

function getFormatterForType(type) {
  const formatters = {
    'js': 'prettier --write {file}',
    'ts': 'prettier --write {file}',
    'jsx': 'prettier --write {file}',
    'tsx': 'prettier --write {file}',
    'py': 'black {file}',
    'rb': 'rubocop -a {file}',
    'go': 'gofmt -w {file}',
    'rust': 'rustfmt {file}'
  };
  return formatters[type] || 'cat {file}';
}
```

### 模式 5: Hook 监控和度量

```json
{
  "hooks": {
    "PostToolUse": {
      "enabled": true,
      "metrics": {
        "enabled": true,
        "storage": ".claude/metrics/",
        "collect": [
          "tool_name",
          "execution_time",
          "success",
          "file_path",
          "timestamp"
        ]
      },
      "command": "./scripts/collect-metrics.sh '{tool}' '{duration}' '{success}'"
    }
  },

  "monitoring": {
    "enabled": true,
    "thresholds": {
      "tool_execution_time": {
        "warn": 5000,   // 5秒
        "alert": 10000  // 10秒
      },
      "hook_execution_time": {
        "warn": 2000,
        "alert": 5000
      },
      "failure_rate": {
        "warn": 0.1,    // 10%
        "alert": 0.2    // 20%
      }
    },
    "alerts": {
      "slack": {
        "webhook": "${SLACK_WEBHOOK_URL}",
        "channel": "#claude-code-alerts"
      },
      "email": {
        "to": "team@example.com",
        "subject": "Claude Code Hook Alert"
      }
    }
  }
}
```

**度量收集脚本** (`scripts/collect-metrics.sh`):
```bash
#!/bin/bash

TOOL=$1
DURATION=$2
SUCCESS=$3

METRICS_FILE=".claude/metrics/$(date +%Y-%m).json"

# 创建度量条目
METRIC=$(cat <<EOF
{
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "tool": "$TOOL",
  "duration": $DURATION,
  "success": $SUCCESS
}
EOF
)

# 追加到度量文件
echo "$METRIC" >> "$METRICS_FILE"

# 检查阈值
if [ "$DURATION" -gt 5000 ]; then
  echo "⚠️  警告: 工具执行时间过长 (${DURATION}ms)"
fi
```

---

## 下一步

- 学习 [MCP 集成](./07-mcp-integration.md)，扩展外部工具
- 查看 [最佳实践](./08-best-practices.md)，优化 Hook 配置
- 参考 [参考资源](./09-references.md)，深入学习

---

**参考资源**:
- [Hooks Reference - Claude Code Docs](https://code.claude.com/docs/en/hooks)
- [Claude Code Hooks 实践指南](https://www.datacamp.com/tutorial/claude-code-hooks)
- [工作流自动化最佳实践](https://claudefa.st/blog/guide/workflows/automation)
