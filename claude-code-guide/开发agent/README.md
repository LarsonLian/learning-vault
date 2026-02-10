# 自动化开发Agent系统

完整的软件开发自动化系统，从需求分析到代码交付的全流程自动化。

## 📁 目录结构

```
开发agent/
├── README.md                    # 本文件
├── subagents/                   # 7个专门的subagent
│   ├── requirements-analyst.subagent.md
│   ├── architect.subagent.md
│   ├── test-designer.subagent.md
│   ├── task-planner.subagent.md
│   ├── developer.subagent.md
│   ├── qa-engineer.subagent.md
│   └── debugger.subagent.md
└── skills/                      # 主控编排skill
    └── orchestrate-workflow/
        └── SKILL.md
```

## 🎯 系统架构

### 1个Skill + 7个Subagent

```
用户会话
    │
    ├─ Skill: /orchestrate-workflow ← 主控编排（在当前会话执行）
    │   │
    │   ├─ Task → requirements-analyst subagent
    │   ├─ Task → architect subagent
    │   ├─ Task → test-designer subagent
    │   ├─ Task → task-planner subagent
    │   ├─ Task → developer subagent (x3, parallel)
    │   ├─ Task → qa-engineer subagent
    │   └─ Task → debugger subagent
    │
    └─ TodoWrite (全局任务列表，用户可见)
```

## 🚀 安装方法

### 1. 安装Subagents

```bash
# 复制所有subagent配置到Claude Code目录
cp subagents/*.subagent.md ~/.claude/subagents/
```

验证安装：
```bash
ls ~/.claude/subagents/
# 应该看到7个 .subagent.md 文件
```

### 2. 安装Skill

```bash
# 复制orchestrate-workflow skill
cp -r skills/orchestrate-workflow ~/.claude/skills/
```

验证安装：
```bash
ls ~/.claude/skills/orchestrate-workflow/
# 应该看到 SKILL.md
```

## 📋 7个Subagent说明

### 1. Requirements Analyst（需求分析专家）

**职责**: 解析需求文档并生成结构化需求清单

**输入**: `docs/requirements.md`

**输出**:
- `requirements-analysis.json` - 机器可读的结构化需求
- `REQUIREMENTS_ANALYSIS.md` - 人类可读的分析报告

**使用的Skills**: `research:deep-dive`, `consider:first-principles`

---

### 2. Architect（系统架构师）

**职责**: 技术选型和系统架构设计

**输入**: `requirements-analysis.json`

**输出**:
- `ARCHITECTURE.md` - 完整架构文档
- `api-spec.yaml` - OpenAPI 3.0规范
- `schema.sql` - 数据库DDL
- `docker-compose.yml` - Docker配置

**使用的Skills**: `research:technical`, `brainstorming`, `consider:swot`

---

### 3. Test Designer（测试工程师）

**职责**: 生成完整的测试套件（单元/集成/E2E）

**输入**:
- `requirements-analysis.json`
- `ARCHITECTURE.md`
- `api-spec.yaml`

**输出**:
- `tests/unit/` - 单元测试（60%）
- `tests/integration/` - 集成测试（30%）
- `tests/e2e/` - E2E测试（10%）
- `pytest.ini` - 测试配置
- `TEST_PLAN.md` - 测试计划

---

### 4. Task Planner（任务规划专家）

**职责**: 将架构拆分为可执行的开发任务

**输入**: `ARCHITECTURE.md`, `api-spec.yaml`

**输出**:
- `tasks.json` - 结构化任务列表
- `DEVELOPMENT_PLAN.md` - 开发计划
- `dependency_graph.mermaid` - 依赖图

**使用的Skills**: `create-plan`, `consider:pareto`

---

### 5. Developer（软件工程师）⭐

**职责**: 在隔离的git worktree中使用TDD实现代码

**工作流**:
1. `Skill("using-git-worktrees")` - 创建 `.worktrees/task-{id}/`
2. `Skill("test-driven-development")` - RED-GREEN-REFACTOR
3. `Skill("finishing-a-development-branch")` - 提交并清理

**特点**: 可以启动多个实例并行工作！

**使用的Skills**: `using-git-worktrees`, `test-driven-development`, `finishing-a-development-branch`

---

### 6. QA Engineer（QA工程师）

**职责**: 部署代码并执行完整测试套件

**执行的测试**:
```bash
pytest tests/unit/ -v --cov=app
pytest tests/integration/ -v
pytest tests/e2e/ -v
pylint app/
mypy app/
bandit -r app/
```

**输出**:
- `TEST_REPORT.md` - 测试执行报告
- `reports/` - 覆盖率、Linter、安全扫描报告

**使用的Skills**: `verification-before-completion`

---

### 7. Debugger（调试专家）

**职责**: 系统化分析和修复代码缺陷

**输入**: `TEST_REPORT.md`（失败的测试）

**调试流程**:
1. `Skill("debug-like-expert")` - 专家级调试
2. `Skill("systematic-debugging")` - 系统化调试
3. `Skill("consider:5-whys")` - 根因分析
4. 修复代码
5. 回归测试

**输出**:
- 修复后的代码
- `DEBUG_REPORT.md` - 修复报告

**使用的Skills**: `debug-like-expert`, `systematic-debugging`, `consider:5-whys`

---

## 🎯 /orchestrate-workflow Skill

主控编排skill，协调整个开发流程的9个阶段：

```
Phase 0: 初始化 → 创建TodoWrite任务列表
Phase 1: 需求分析 → Requirements Analyst
Phase 2: 架构设计 → Architect
Phase 3: 测试用例 → Test Designer
Phase 4: 任务拆分 → Task Planner
Phase 5: 并行开发 → 3x Developer (parallel)
Phase 6: 合并分支 → git merge
Phase 7: 部署测试 → QA Engineer
Phase 8: 问题修复 → Debugger (if needed)
Phase 9: 交付验证 → Code Review
```

### 使用方法

```
用户: "基于 docs/requirements.md 开发用户管理系统"

Claude会自动调用: Skill("orchestrate-workflow")

执行完整的9个阶段，最终交付：
✓ 可运行的代码
✓ 完整的测试套件（覆盖率≥85%）
✓ 完整文档
✓ Docker部署配置
```

## 💡 使用示例

### 示例1: 完整开发流程

```
用户: "基于 docs/user-management-requirements.md 开发系统"

Claude:
├─ Phase 1: Requirements Analyst ✓
│   └─ 输出: requirements-analysis.json (15个FR, 8个NFR)
│
├─ Phase 2: Architect ✓
│   └─ 输出: ARCHITECTURE.md, api-spec.yaml, schema.sql
│
├─ Phase 3: Test Designer ✓
│   └─ 输出: tests/ (60个单元测试, 25个集成测试, 8个E2E)
│
├─ Phase 4: Task Planner ✓
│   └─ 输出: tasks.json (8个任务, 3个可并行)
│
├─ Phase 5: 并行开发 ✓
│   ├─ Developer #1: T-001, T-002 (worktree: .worktrees/task-T001/)
│   ├─ Developer #2: T-003, T-004 (worktree: .worktrees/task-T002/)
│   └─ Developer #3: T-005, T-006 (worktree: .worktrees/task-T003/)
│
├─ Phase 6: 合并分支 ✓
│   └─ git merge feature/task-{T001,T002,T003}
│
├─ Phase 7: QA Engineer ✓
│   └─ 测试结果: 90 passed, 3 failed
│
├─ Phase 8: Debugger ✓
│   └─ 修复3个bug
│
├─ Phase 7 (retry): QA Engineer ✓
│   └─ 测试结果: 93 passed ✓
│
└─ Phase 9: 交付验证 ✓
    └─ 覆盖率: 92%, Pylint: 8.7/10

🎉 开发完成！总耗时: 72分钟
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

# 并行启动3个Developer
Task(subagent_type="developer", prompt="实现T-001", run_in_background=True)
Task(subagent_type="developer", prompt="实现T-002", run_in_background=True)
Task(subagent_type="developer", prompt="实现T-003", run_in_background=True)
```

## 🔥 核心特性

### 1. Git Worktrees并行开发 ⭐⭐⭐

Developer Agent使用git worktrees实现真正的并行开发：

```
主仓库 /workspace/project/
└── .worktrees/
    ├── task-T001/  (Developer #1)
    ├── task-T002/  (Developer #2)
    └── task-T003/  (Developer #3)
```

**优势**:
- ✅ 3个agent同时工作，互不干扰
- ✅ 各自独立的分支和测试环境
- ✅ 最后统一合并，检测集成问题

### 2. TDD驱动开发

所有代码实现都遵循TDD流程：
1. **RED**: 写测试（失败）
2. **GREEN**: 实现代码（通过）
3. **REFACTOR**: 重构优化

### 3. 多层质量保证

- 单元测试（每个函数）
- 集成测试（API端点）
- E2E测试（业务流程）
- 代码覆盖率≥85%
- 静态分析（pylint, mypy）
- 安全扫描（bandit）

### 4. Skills充分复用

| Agent | 核心Skills |
|-------|-----------|
| Architect | research:technical, brainstorming, consider:swot |
| Task Planner | **create-plan** ⭐ |
| Developer | **using-git-worktrees**, **test-driven-development** ⭐⭐ |
| Debugger | **debug-like-expert**, **systematic-debugging** ⭐ |
| Orchestrator | dispatching-parallel-agents |

## 📊 性能预估

基于一个中型项目（15个功能需求）：

| Phase | Agent | 预估时间 | 并行能力 |
|-------|-------|---------|---------|
| Phase 1 | Requirements Analyst | 5-10分钟 | - |
| Phase 2 | Architect | 10-15分钟 | - |
| Phase 3 | Test Designer | 8-12分钟 | - |
| Phase 4 | Task Planner | 3-5分钟 | - |
| Phase 5 | 3x Developer | 20-30分钟 | ✅ 并行 |
| Phase 6 | 合并 | 1-2分钟 | - |
| Phase 7 | QA Engineer | 5-8分钟 | - |
| Phase 8 | Debugger | 5-10分钟 | 可选 |
| Phase 9 | Code Review | 3-5分钟 | - |

**总计**: ~60-100分钟（单线程）→ **~40-60分钟（并行）**

## 🔧 技术要求

### 必需的现有Skills

确保以下skills已安装（来自superpowers和taches-cc-resources）：

- `using-git-worktrees` ⭐⭐⭐
- `test-driven-development`
- `finishing-a-development-branch`
- `dispatching-parallel-agents`
- `create-plan`
- `debug-like-expert`
- `systematic-debugging`
- `research:technical`
- `brainstorming`
- `consider:swot`
- `consider:5-whys`
- `consider:pareto`
- `verification-before-completion`
- `requesting-code-review`

### Git要求

- Git 2.5+ (支持worktrees)
- 确保 `.worktrees/` 在 `.gitignore` 中

### 测试框架

根据编程语言：
- **Python**: pytest, pylint, mypy, bandit, black
- **JavaScript**: jest, eslint, prettier
- **Java**: JUnit, Checkstyle

## 📚 相关文档

项目文档位于 `/docs/claude-code/`:

1. `autonomous-dev-agent-architecture.md` - 整体架构设计
2. `subagents-detailed-specs.md` - 8个subagent详细规格
3. `skills-gap-analysis.md` - Skills需求分析
4. `parallel-development-with-worktrees.md` - Git Worktrees详细指南
5. `subagents-summary.md` - 快速参考

## ⚠️ 注意事项

### 1. Git Worktrees是必需的

Developer Agent的并行开发依赖git worktrees，不要在同一工作区并行开发。

### 2. Orchestrator是Skill不是Subagent

主控编排应该通过 `Skill("orchestrate-workflow")` 调用，不要用 `Task(subagent_type="orchestrator")`。

### 3. 每个Phase验证输出

确保每个阶段完成后验证输出文件是否生成。

### 4. 测试失败自动修复

Phase 7发现测试失败会自动进入Phase 8启动Debugger，无需手动干预。

## 🐛 故障排除

### 问题1: Subagent未找到

```
错误: Subagent 'requirements-analyst' not found
解决: 确保所有 .subagent.md 文件已复制到 ~/.claude/subagents/
```

### 问题2: Skill未识别

```
错误: Skill 'orchestrate-workflow' not found
解决: 确保 orchestrate-workflow/ 目录已复制到 ~/.claude/skills/
```

### 问题3: Git Worktree创建失败

```
错误: fatal: '.worktrees/task-T001' already exists
解决: git worktree prune && rm -rf .worktrees/task-T001
```

### 问题4: 依赖Skills缺失

```
错误: Skill 'using-git-worktrees' not found
解决: 安装 superpowers 插件，确保所有必需skills已安装
```

## 🤝 贡献

如果改进了某个subagent或skill，请：

1. 修改对应的 `.subagent.md` 或 `SKILL.md`
2. 更新这个README
3. 提交变更

## 📝 版本历史

- **v1.0.0** (2024-02-10)
  - 初始版本
  - 7个subagent + 1个skill
  - 支持完整的开发流程自动化
  - Git Worktrees并行开发

## 📄 许可

MIT License

---

**这是真正的软件开发自动化！** 🚀

从需求文档到可运行的代码，全流程自动化，效率提升3倍！
