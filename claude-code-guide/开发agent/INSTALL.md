# 自动化开发Agent系统 - 安装指南

## 📦 快速安装

### 一键安装（推荐）

```bash
cd /Users/mlamp/workspace/deepminer/claude-code-guide/开发agent

# 安装所有subagents
cp subagents/*.subagent.md ~/.claude/subagents/

# 安装skill
cp -r skills/orchestrate-workflow ~/.claude/skills/

echo "✅ 安装完成！"
```

### 验证安装

```bash
# 验证subagents
ls ~/.claude/subagents/*.subagent.md
# 应该看到7个文件：
# - requirements-analyst.subagent.md
# - architect.subagent.md
# - test-designer.subagent.md
# - task-planner.subagent.md
# - developer.subagent.md
# - qa-engineer.subagent.md
# - debugger.subagent.md

# 验证skill
ls ~/.claude/skills/orchestrate-workflow/
# 应该看到 SKILL.md
```

---

## 📋 组件说明

### 7个Subagent

| Subagent | 文件大小 | 职责 |
|----------|---------|------|
| requirements-analyst | 8.3KB | 需求分析 |
| architect | 15KB | 架构设计 |
| test-designer | 20KB | 测试生成 |
| task-planner | 1.2KB | 任务拆分 |
| developer | 2.0KB | 代码实现 |
| qa-engineer | 2.1KB | 测试执行 |
| debugger | 2.6KB | 问题修复 |

**总计**: ~51KB

### 1个Skill

| Skill | 文件大小 | 职责 |
|-------|---------|------|
| orchestrate-workflow | 23KB | 主控编排 |

---

## 🔧 依赖要求

### 必需的Skills

确保以下skills已安装（来自superpowers和taches-cc-resources插件）：

#### 核心Skills（必需）⭐⭐⭐

```bash
# 检查是否已安装
claude-code --list-skills | grep -E "using-git-worktrees|test-driven-development|finishing-a-development-branch"
```

- **using-git-worktrees** - Git worktrees并行开发（最重要！）
- **test-driven-development** - TDD开发流程
- **finishing-a-development-branch** - 完成分支并清理

#### 编排Skills

- **dispatching-parallel-agents** - 并行agent调度
- **subagent-driven-development** - 子agent驱动开发

#### 规划Skills

- **create-plan** - 创建分层计划（任务拆分使用）

#### 调试Skills

- **debug-like-expert** - 专家级调试
- **systematic-debugging** - 系统化调试

#### 调研Skills

- **research:technical** - 技术调研
- **research:options** - 对比技术选项
- **research:deep-dive** - 深度调研

#### 思考框架Skills

- **brainstorming** - 头脑风暴
- **consider:swot** - SWOT分析
- **consider:5-whys** - 5Why根因分析
- **consider:pareto** - 帕累托分析
- **consider:first-principles** - 第一性原理

#### 质量保证Skills

- **verification-before-completion** - 完成前验证
- **requesting-code-review** - 请求代码审查

### 如何安装依赖Skills

如果缺少某些skills，需要安装对应的插件：

```bash
# 安装superpowers插件（包含git-worktrees等核心skills）
claude-code install superpowers

# 安装taches-cc-resources插件（包含debugging、research等skills）
claude-code install taches-cc-resources
```

---

## ✅ 安装验证

### 步骤1: 检查Subagents

```bash
# 应该看到7个subagent文件
ls -1 ~/.claude/subagents/*.subagent.md | wc -l
# 输出应该 >= 7
```

### 步骤2: 检查Skill

```bash
# 检查orchestrate-workflow skill
cat ~/.claude/skills/orchestrate-workflow/SKILL.md | head -5
# 应该看到：
# ---
# name: orchestrate-workflow
# description: 编排完整的自动化开发工作流
```

### 步骤3: 测试调用

在Claude Code中测试：

```
用户: 列出所有可用的subagents

Claude应该能识别：
- requirements-analyst
- architect
- test-designer
- task-planner
- developer
- qa-engineer
- debugger
```

---

## 🚀 快速开始

### 示例1: 完整开发流程

```
用户: "基于 docs/requirements.md 开发用户管理系统"

Claude会自动：
1. 调用 orchestrate-workflow skill
2. 依次执行9个阶段
3. 输出可运行的代码和完整文档
```

### 示例2: 单独使用某个Agent

```python
# 只做需求分析
Task(
    subagent_type="requirements-analyst",
    prompt="分析 docs/requirements.md"
)

# 只做架构设计
Task(
    subagent_type="architect",
    prompt="基于 requirements-analysis.json 设计架构"
)
```

---

## 🛠️ 故障排除

### 问题1: Subagent未找到

**错误信息**:
```
Error: Subagent 'requirements-analyst' not found
```

**解决方法**:
```bash
# 确认文件是否存在
ls ~/.claude/subagents/requirements-analyst.subagent.md

# 如果不存在，重新复制
cp subagents/requirements-analyst.subagent.md ~/.claude/subagents/
```

### 问题2: Skill未识别

**错误信息**:
```
Error: Skill 'orchestrate-workflow' not found
```

**解决方法**:
```bash
# 确认skill目录
ls ~/.claude/skills/orchestrate-workflow/SKILL.md

# 如果不存在，重新复制
cp -r skills/orchestrate-workflow ~/.claude/skills/
```

### 问题3: 依赖Skills缺失

**错误信息**:
```
Error: Skill 'using-git-worktrees' not found
```

**解决方法**:
```bash
# 安装superpowers插件
claude-code install superpowers

# 或手动检查插件
ls ~/.claude/plugins/cache/superpowers-marketplace/
```

### 问题4: Git Worktree创建失败

**错误信息**:
```
fatal: '.worktrees/task-T001' already exists
```

**解决方法**:
```bash
# 清理过期的worktree
cd /path/to/project
git worktree prune

# 删除残留目录
rm -rf .worktrees/task-T001

# 确保.worktrees在.gitignore中
echo ".worktrees/" >> .gitignore
```

---

## 📝 配置检查清单

安装完成后，确认以下各项：

- [ ] 7个subagent文件在 `~/.claude/subagents/`
- [ ] orchestrate-workflow skill在 `~/.claude/skills/`
- [ ] `using-git-worktrees` skill已安装（superpowers插件）
- [ ] `test-driven-development` skill已安装
- [ ] `debug-like-expert` skill已安装
- [ ] `create-plan` skill已安装（taches-cc-resources插件）
- [ ] Git版本 >= 2.5（支持worktrees）
- [ ] 项目有 `.gitignore` 且包含 `.worktrees/`

---

## 🔄 更新安装

如果配置文件有更新：

```bash
# 重新复制所有文件（会覆盖）
cd /Users/mlamp/workspace/deepminer/claude-code-guide/开发agent

# 更新subagents
cp subagents/*.subagent.md ~/.claude/subagents/

# 更新skill
cp -r skills/orchestrate-workflow ~/.claude/skills/

echo "✅ 更新完成！"
```

---

## 📚 下一步

安装完成后：

1. 阅读 [README.md](README.md) 了解系统架构
2. 查看 `/docs/claude-code/` 下的详细文档
3. 在测试项目中验证完整流程
4. 根据实际使用反馈优化配置

---

## 💡 提示

- **首次使用**: 建议在测试项目中先试用，熟悉流程
- **Git Worktrees**: 确保理解git worktrees的工作原理
- **并行开发**: 小心处理任务依赖，避免并行冲突
- **测试覆盖率**: 默认目标≥85%，可在配置中调整

---

**安装遇到问题？** 查看项目文档或提issue。
