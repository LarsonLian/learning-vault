---
name: orchestrate-workflow
description: 编排完整的自动化开发工作流，从需求分析到交付
version: 1.0.0
---

# Orchestrate Workflow Skill

## 何时使用

当用户请求完整的自动化开发流程时使用此skill：
- "基于需求文档开发XXX系统"
- "自动完成从需求到交付的全流程"
- "帮我实现XXX功能，包括需求分析、架构设计、测试、开发"

## 概述

这个skill编排整个软件开发生命周期，协调8个阶段的执行：

```
需求分析 → 架构设计 → 测试用例 → 任务拆分
→ 并行开发 → 部署测试 → 问题修复 → 交付验证
```

## 工作流程

### Phase 0: 初始化

```xml
<step>确认输入和创建任务列表</step>
```

**必须执行**:
1. 确认需求文档路径
2. 确认项目根目录
3. 确认目标编程语言（Python/JavaScript/Java）

**创建TodoWrite任务列表**:
```python
TodoWrite([
    {"content": "需求分析", "status": "pending", "activeForm": "需求分析中"},
    {"content": "架构设计", "status": "pending", "activeForm": "架构设计中"},
    {"content": "测试用例生成", "status": "pending", "activeForm": "测试用例生成中"},
    {"content": "任务拆分", "status": "pending", "activeForm": "任务拆分中"},
    {"content": "并行开发", "status": "pending", "activeForm": "并行开发中"},
    {"content": "部署测试", "status": "pending", "activeForm": "部署测试中"},
    {"content": "问题修复", "status": "pending", "activeForm": "问题修复中"},
    {"content": "交付验证", "status": "pending", "activeForm": "交付验证中"}
])
```

**创建状态文件**:
```python
Write("WORKFLOW_STATE.json", {
    "project_id": generate_project_id(),
    "started_at": now(),
    "current_phase": "requirements_analysis",
    "phases": {}
})
```

---

### Phase 1: 需求分析

```xml
<step>启动 Requirements Analyst subagent</step>
```

**更新状态**:
```python
TodoWrite([
    {"content": "需求分析", "status": "in_progress", ...},
    ...
])
```

**启动subagent**:
```python
requirements_result = Task(
    subagent_type="requirements-analyst",
    description="分析需求文档",
    prompt=f"""
你是需求分析专家。

任务：分析需求文档并生成结构化需求清单。

输入文档：{requirement_doc_path}

执行步骤：
1. 使用 Read 工具读取需求文档
2. 提取功能性需求（FR-001, FR-002...）
3. 提取非功能性需求（性能、安全、可用性）
4. 识别技术约束和依赖
5. 生成 requirements-analysis.json
6. 生成 REQUIREMENTS_ANALYSIS.md

必须生成两个文件：
- requirements-analysis.json（机器可读）
- REQUIREMENTS_ANALYSIS.md（人类可读）
    """
)
```

**验证输出**:
```python
if not exists("requirements-analysis.json"):
    raise Error("需求分析失败：未生成requirements-analysis.json")

# 更新状态
TodoWrite([
    {"content": "需求分析", "status": "completed", ...},
    {"content": "架构设计", "status": "in_progress", ...},
    ...
])

update_workflow_state("requirements_analysis", "completed",
                      output="requirements-analysis.json")
```

---

### Phase 2: 架构设计

```xml
<step>启动 Architect subagent</step>
```

**启动subagent**:
```python
architecture_result = Task(
    subagent_type="architect",
    description="设计系统架构",
    prompt=f"""
你是系统架构师。

任务：基于需求分析设计系统架构。

输入：
- requirements-analysis.json

执行步骤：
1. 读取 requirements-analysis.json
2. 使用 Skill("brainstorming") 进行架构设计头脑风暴
3. 使用 Skill("research:technical") 进行技术调研
4. 使用 Skill("research:options") 对比技术选项
5. 使用 Skill("consider:swot") 进行SWOT分析
6. 设计系统分层架构
7. 设计API规范（OpenAPI）
8. 设计数据库Schema（SQL DDL）
9. 设计安全架构
10. 设计部署架构（Docker）

必须生成：
- ARCHITECTURE.md
- api-spec.yaml
- schema.sql
- docker-compose.yml
- TECH_STACK.md
    """
)
```

**验证输出**:
```python
required_files = [
    "ARCHITECTURE.md",
    "api-spec.yaml",
    "schema.sql"
]

for file in required_files:
    if not exists(file):
        raise Error(f"架构设计失败：缺少{file}")

update_todo("架构设计", "completed")
update_workflow_state("architecture_design", "completed")
```

---

### Phase 3: 测试用例生成

```xml
<step>启动 Test Designer subagent</step>
```

**启动subagent**:
```python
test_result = Task(
    subagent_type="test-designer",
    description="生成测试套件",
    prompt="""
你是测试工程师。

任务：生成完整的测试套件。

输入：
- requirements-analysis.json
- ARCHITECTURE.md
- api-spec.yaml

执行步骤：
1. 读取需求和架构文档
2. 生成单元测试（tests/unit/）
   - 为每个业务逻辑函数生成测试
   - 测试正常路径、边界条件、异常处理
3. 生成集成测试（tests/integration/）
   - 测试API端点
   - 测试数据库交互
4. 生成E2E测试（tests/e2e/）
   - 测试完整业务流程
5. 生成测试数据（tests/fixtures.py）
6. 配置测试框架（pytest.ini）
7. 生成 TEST_PLAN.md

测试金字塔分配：
- 单元测试：60%
- 集成测试：30%
- E2E测试：10%

目标覆盖率：≥85%
    """
)
```

**验证输出**:
```python
if not exists("tests/unit/") or not exists("tests/integration/"):
    raise Error("测试用例生成失败")

# 统计测试数量
unit_tests = count_test_files("tests/unit/")
integration_tests = count_test_files("tests/integration/")
e2e_tests = count_test_files("tests/e2e/")

print(f"✓ 生成 {unit_tests} 个单元测试")
print(f"✓ 生成 {integration_tests} 个集成测试")
print(f"✓ 生成 {e2e_tests} 个E2E测试")

update_todo("测试用例生成", "completed")
```

---

### Phase 4: 任务拆分

```xml
<step>启动 Task Planner subagent</step>
```

**启动subagent**:
```python
tasks_result = Task(
    subagent_type="task-planner",
    description="拆分开发任务",
    prompt="""
你是任务规划专家。

任务：将架构拆分为可执行的开发任务。

输入：
- ARCHITECTURE.md
- api-spec.yaml

执行步骤：
1. 读取架构设计
2. 使用 Skill("create-plan") 创建分层开发计划
3. 识别模块和组件
4. 拆分可独立开发的任务
5. 分析任务间依赖关系（构建DAG）
6. 识别可并行执行的任务组
7. 使用 Skill("consider:pareto") 进行优先级分析

输出格式：
- tasks.json（结构化任务列表）
- DEVELOPMENT_PLAN.md（开发计划文档）
- dependency_graph.mermaid（依赖图）

任务定义必须包含：
- task_id（T-001, T-002...）
- title
- description
- files_to_create
- files_to_modify
- dependencies
- can_parallel（是否可并行）
    """
)
```

**读取并分析任务**:
```python
tasks = read_json("tasks.json")
parallel_groups = tasks["parallel_groups"]

print(f"✓ 拆分为 {len(tasks['phases'])} 个阶段")
print(f"✓ 共 {count_all_tasks(tasks)} 个任务")
print(f"✓ {len(parallel_groups)} 组可并行执行")

update_todo("任务拆分", "completed")
```

---

### Phase 5: 并行开发 ⭐

```xml
<step>使用 dispatching-parallel-agents skill 启动多个 Developer subagent</step>
```

**读取可并行的任务组**:
```python
tasks = read_json("tasks.json")
parallel_tasks = tasks["parallel_groups"][0]  # 第一组并行任务

print(f"📋 准备并行执行 {len(parallel_tasks)} 个任务")
```

**使用 dispatching-parallel-agents skill**:
```python
Skill("dispatching-parallel-agents", args=f"""
任务：并行启动 {len(parallel_tasks)} 个Developer agent

任务列表：
{format_task_list(parallel_tasks)}

每个Developer agent的指令：
你是软件工程师，负责实现一个具体的开发任务。

工作流程：
1. **必须首先使用 Skill("using-git-worktrees")**
   - 这会创建隔离的工作区 .worktrees/task-{{task_id}}/
   - 创建独立分支 feature/task-{{task_id}}
   - 安装依赖
   - 运行基线测试

2. **使用 Skill("test-driven-development") 实现代码**
   - RED: 先写测试（失败）
   - GREEN: 实现代码（通过）
   - REFACTOR: 重构优化

3. **运行代码质量检查**
   - Python: black, pylint, mypy
   - JavaScript: prettier, eslint

4. **使用 Skill("finishing-a-development-branch")**
   - 确认所有测试通过
   - 提交代码
   - 清理worktree
   - 返回主仓库

重要：你只需关注自己的任务，不用担心其他agent的工作。
""")
```

**等待所有开发完成**:
```python
# dispatching-parallel-agents skill会自动等待所有agent完成
print("✅ 所有并行开发完成")

update_todo("并行开发", "completed")
```

---

### Phase 6: 合并分支

```xml
<step>合并所有feature分支到main</step>
```

**执行合并**:
```python
bash("git checkout main")

for task_id in parallel_tasks:
    branch_name = f"feature/task-{task_id}"

    result = bash(f'git merge {branch_name} --no-ff -m "Merge {task_id}"')

    if "CONFLICT" in result:
        print(f"⚠️  合并 {branch_name} 时发现冲突")
        # 稍后在Phase 8由Debugger处理
    else:
        print(f"✓ 已合并 {branch_name}")
```

---

### Phase 7: 部署测试

```xml
<step>启动 QA Engineer subagent</step>
```

**启动subagent**:
```python
test_report = Task(
    subagent_type="qa-engineer",
    description="执行完整测试套件",
    prompt="""
你是QA工程师。

任务：部署代码并执行完整测试套件。

执行步骤：
1. 安装依赖
   pip install -r requirements.txt
   pip install -r requirements-dev.txt

2. 启动测试数据库
   docker-compose -f docker-compose.test.yml up -d

3. 运行单元测试
   pytest tests/unit/ -v --cov=app --cov-report=html

4. 运行集成测试
   pytest tests/integration/ -v

5. 运行E2E测试
   pytest tests/e2e/ -v

6. 代码质量检查
   pylint app/ --output-format=json > reports/pylint.json
   mypy app/ --html-report reports/mypy

7. 安全扫描
   bandit -r app/ -f json -o reports/bandit.json

8. 生成测试报告
   生成 TEST_REPORT.md，包含：
   - 测试摘要（通过/失败）
   - 失败的测试详情
   - 覆盖率报告
   - 代码质量评分
   - 安全问题

9. 使用 Skill("verification-before-completion") 验证
    """
)
```

**分析测试结果**:
```python
test_report = read("TEST_REPORT.md")

# 提取失败数量
failures = extract_failure_count(test_report)

if failures > 0:
    print(f"❌ 发现 {failures} 个测试失败")
    update_todo("部署测试", "completed")
    update_todo("问题修复", "in_progress")
    # 进入Phase 8
else:
    print("✅ 所有测试通过！")
    update_todo("部署测试", "completed")
    # 跳到Phase 9
```

---

### Phase 8: 问题修复（如果需要）

```xml
<step>如果有测试失败，启动 Debugger subagent</step>
```

**启动subagent**:
```python
if has_test_failures:
    debug_result = Task(
        subagent_type="debugger",
        description="修复测试失败",
        prompt="""
你是调试专家。

任务：分析并修复所有失败的测试。

输入：TEST_REPORT.md

执行步骤：
1. 读取 TEST_REPORT.md，识别所有失败的测试

2. 对每个失败的测试：
   a. 使用 Skill("debug-like-expert") 进行深度分析
   b. 使用 Skill("systematic-debugging") 系统化调试
      - 重现问题
      - 隔离变量
      - 形成假设
      - 验证假设

   c. 使用 Skill("consider:5-whys") 进行根因分析
      示例：
      - Why 1: 测试失败？→ 未抛出正确异常
      - Why 2: 未抛出异常？→ 未捕获数据库错误
      - Why 3: 未捕获？→ 缺少try-except
      - Why 4: 缺少异常处理？→ 依赖数据库约束
      - Why 5: 依赖约束？→ **根因：缺少业务层校验**

   d. 使用 Edit 工具修复代码

   e. 运行回归测试验证修复
      pytest tests/unit/test_xxx.py -v

3. 生成 DEBUG_REPORT.md，记录：
   - 每个问题的根因
   - 修复方案
   - 修改的文件
   - 测试验证结果

重要：必须找到根本原因，不要只修复症状！
        """
    )

    update_todo("问题修复", "completed")

    # 重新运行测试（回到Phase 7）
    print("🔄 重新运行测试套件...")
    goto Phase 7
```

---

### Phase 9: 交付验证

```xml
<step>最终验证和代码审查</step>
```

**运行最终验证**:
```python
# 使用 verification-before-completion skill
Skill("verification-before-completion", args="""
验证项目：
1. 所有测试通过
2. 覆盖率≥85%
3. 无安全问题（Bandit扫描）
4. 代码质量达标（Pylint ≥8.0）
5. 类型检查通过（Mypy）
6. 文档完整（ARCHITECTURE.md, API_DOCS.md）
""")
```

**请求代码审查**:
```python
Skill("requesting-code-review", args="""
审查范围：
- 所有新增代码
- 测试覆盖率
- 架构设计合理性
- API设计规范性
- 安全性
- 性能
""")
```

**更新状态**:
```python
TodoWrite([
    {"content": "需求分析", "status": "completed", ...},
    {"content": "架构设计", "status": "completed", ...},
    {"content": "测试用例生成", "status": "completed", ...},
    {"content": "任务拆分", "status": "completed", ...},
    {"content": "并行开发", "status": "completed", ...},
    {"content": "部署测试", "status": "completed", ...},
    {"content": "问题修复", "status": "completed", ...},
    {"content": "交付验证", "status": "completed", ...}
])

update_workflow_state("all", "completed")
```

**生成交付报告**:
```python
Write("DELIVERY_REPORT.md", f"""
# 项目交付报告

**项目名称**: {project_name}
**交付时间**: {now()}
**总耗时**: {calculate_duration()}

## 交付清单

### 1. 代码
- ✅ 源代码（app/）
- ✅ 测试代码（tests/）
- ✅ 配置文件（docker-compose.yml）

### 2. 文档
- ✅ 需求分析（REQUIREMENTS_ANALYSIS.md）
- ✅ 架构设计（ARCHITECTURE.md）
- ✅ API文档（api-spec.yaml）
- ✅ 测试计划（TEST_PLAN.md）
- ✅ 开发计划（DEVELOPMENT_PLAN.md）

### 3. 质量指标
- ✅ 测试数量: {total_tests} 个
- ✅ 测试通过率: 100%
- ✅ 代码覆盖率: {coverage}%
- ✅ Pylint评分: {pylint_score}/10
- ✅ 安全问题: 0个

### 4. 部署
- ✅ Docker配置完整
- ✅ 数据库迁移脚本
- ✅ 环境配置示例

## 下一步建议
1. 在生产环境部署
2. 配置CI/CD流水线
3. 设置监控和告警
4. 准备运维文档
""")
```

**输出总结**:
```
🎉 开发流程完成！

交付成果：
✓ 可运行的代码
✓ {total_tests} 个测试（全部通过）
✓ 测试覆盖率: {coverage}%
✓ 完整文档
✓ Docker部署配置

耗时统计：
- 需求分析: 5分钟
- 架构设计: 12分钟
- 测试用例: 10分钟
- 任务拆分: 3分钟
- 并行开发: 25分钟（3个agent并行）
- 部署测试: 8分钟
- 问题修复: 6分钟
- 交付验证: 3分钟
总计: 72分钟

查看完整报告: DELIVERY_REPORT.md
```

---

## 状态管理

### TodoWrite任务列表

在整个流程中使用TodoWrite跟踪进度，用户随时可见：

```python
TodoWrite([
    {"content": "需求分析", "status": "completed", "activeForm": "需求分析中"},
    {"content": "架构设计", "status": "completed", "activeForm": "架构设计中"},
    {"content": "测试用例生成", "status": "completed", "activeForm": "测试用例生成中"},
    {"content": "任务拆分", "status": "completed", "activeForm": "任务拆分中"},
    {"content": "并行开发", "status": "in_progress", "activeForm": "并行开发中"},
    {"content": "部署测试", "status": "pending", "activeForm": "部署测试中"},
    {"content": "问题修复", "status": "pending", "activeForm": "问题修复中"},
    {"content": "交付验证", "status": "pending", "activeForm": "交付验证中"}
])
```

### WORKFLOW_STATE.json

持久化状态文件：

```json
{
  "project_id": "proj-20240115-001",
  "started_at": "2024-01-15T10:00:00Z",
  "current_phase": "parallel_development",
  "phases": {
    "requirements_analysis": {
      "status": "completed",
      "started_at": "2024-01-15T10:00:00Z",
      "completed_at": "2024-01-15T10:05:00Z",
      "duration_minutes": 5,
      "output": ["requirements-analysis.json", "REQUIREMENTS_ANALYSIS.md"]
    },
    "architecture_design": {
      "status": "completed",
      "duration_minutes": 12,
      "output": ["ARCHITECTURE.md", "api-spec.yaml", "schema.sql"]
    },
    "test_design": {
      "status": "completed",
      "duration_minutes": 10,
      "output": ["tests/"]
    },
    "task_planning": {
      "status": "completed",
      "duration_minutes": 3,
      "output": ["tasks.json", "DEVELOPMENT_PLAN.md"]
    },
    "parallel_development": {
      "status": "in_progress",
      "started_at": "2024-01-15T10:30:00Z",
      "developer_agents": [
        {"agent_id": "dev-1", "tasks": ["T-001", "T-002"], "status": "running"},
        {"agent_id": "dev-2", "tasks": ["T-003", "T-004"], "status": "running"},
        {"agent_id": "dev-3", "tasks": ["T-005", "T-006"], "status": "completed"}
      ]
    }
  }
}
```

---

## 错误处理和重试策略

### 重试配置

```python
retry_strategies = {
    "requirements_analysis": {
        "max_retries": 2,
        "on_failure": "ask_user_for_clarification",
        "retry_delay": 0
    },
    "architecture_design": {
        "max_retries": 2,
        "on_failure": "auto_retry",
        "retry_delay": 0
    },
    "code_implementation": {
        "max_retries": 3,
        "on_failure": "launch_debugger",
        "retry_delay": 0
    },
    "test_execution": {
        "max_retries": 2,
        "on_failure": "launch_debugger",
        "retry_delay": 0
    }
}
```

### 失败处理

```python
def handle_phase_failure(phase_name, error):
    if phase_name == "requirements_analysis":
        # 需求不清晰，询问用户
        AskUserQuestion([{
            "question": "需求分析遇到问题，请澄清以下信息：",
            "header": "需求澄清",
            "options": [...]
        }])

    elif phase_name in ["test_execution", "code_implementation"]:
        # 代码或测试问题，启动Debugger
        Task(subagent_type="debugger", ...)

    else:
        # 其他问题，自动重试
        retry_phase(phase_name)
```

---

## 用户交互

### 关键决策点

在以下时机使用 `AskUserQuestion` 咨询用户：

1. **架构设计阶段**:
   ```python
   AskUserQuestion([{
       "question": "我们有两个技术选型方案，你倾向哪个？",
       "header": "技术选型",
       "options": [
           {"label": "FastAPI + PostgreSQL（推荐）", "description": "高性能，现代"},
           {"label": "Flask + MySQL", "description": "成熟稳定"}
       ]
   }])
   ```

2. **测试失败后**:
   ```python
   if failures > 10:
       AskUserQuestion([{
           "question": "发现较多测试失败，是继续修复还是先审查代码？",
           "header": "处理策略",
           "options": [
               {"label": "继续修复（推荐）", "description": "让Debugger逐个修复"},
               {"label": "暂停审查", "description": "人工介入检查"}
           ]
       }])
   ```

---

## 工具使用

此skill可使用以下工具：

- **Task** - 启动subagent（核心工具）
- **TodoWrite** - 管理任务列表
- **Read/Write** - 读写状态文件
- **Bash** - Git操作、文件操作
- **AskUserQuestion** - 用户交互
- **Skill** - 调用其他skills
  - `dispatching-parallel-agents` - 并行调度
  - `verification-before-completion` - 完成验证
  - `requesting-code-review` - 代码审查

---

## 输出清单

完整流程会生成以下产出：

### 文档
- requirements-analysis.json
- REQUIREMENTS_ANALYSIS.md
- ARCHITECTURE.md
- api-spec.yaml
- schema.sql
- TEST_PLAN.md
- DEVELOPMENT_PLAN.md
- tasks.json
- TEST_REPORT.md
- DEBUG_REPORT.md (如果有bug修复)
- DELIVERY_REPORT.md

### 代码
- app/ (源代码)
- tests/ (测试代码)
- docker-compose.yml
- requirements.txt
- pytest.ini 或 jest.config.js

### 报告
- reports/coverage/ (覆盖率报告)
- reports/pylint.json
- reports/mypy/
- reports/bandit.json

---

## 使用示例

### 示例1: 完整开发流程

```
用户: "基于 docs/user-management-requirements.md 开发用户管理系统"

Claude: 我将使用 orchestrate-workflow skill 完成完整开发流程。

[调用 Skill("orchestrate-workflow")]

Phase 1: 需求分析 ✓
- 识别到 15 个功能需求
- 识别到 8 个非功能需求
- 生成 requirements-analysis.json

Phase 2: 架构设计 ✓
- 技术栈：FastAPI + PostgreSQL + Redis
- 生成 ARCHITECTURE.md, api-spec.yaml, schema.sql

Phase 3: 测试用例生成 ✓
- 生成 60 个单元测试
- 生成 25 个集成测试
- 生成 8 个E2E测试

Phase 4: 任务拆分 ✓
- 拆分为 8 个开发任务
- 识别 3 个任务可并行执行

Phase 5: 并行开发 ✓
- Developer #1: T-001, T-002 (完成)
- Developer #2: T-003, T-004 (完成)
- Developer #3: T-005, T-006 (完成)

Phase 6: 合并分支 ✓
- 合并所有feature分支到main

Phase 7: 部署测试 ✓
- 93 tests: 90 passed, 3 failed

Phase 8: 问题修复 ✓
- 修复 3 个测试失败
- 重新测试：93 passed ✓

Phase 9: 交付验证 ✓
- 测试覆盖率: 92%
- 代码质量: Pylint 8.7/10
- 安全扫描: 0 issues

🎉 开发完成！
总耗时: 72分钟
查看: DELIVERY_REPORT.md
```

### 示例2: 只执行部分阶段

用户也可以只使用某个subagent：

```
用户: "只做需求分析，不要开发"

Claude:
Task(
    subagent_type="requirements-analyst",
    prompt="读取 docs/requirements.md 并生成需求分析"
)
```

---

## 最佳实践

### 1. 始终创建TodoWrite任务列表

在Phase 0必须创建全局任务列表，让用户知道进度。

### 2. 验证每个Phase的输出

每个Phase完成后必须验证输出文件是否存在。

### 3. 使用Git Worktrees并行开发

Phase 5必须使用 `dispatching-parallel-agents` skill，它会自动为每个Developer创建worktree。

### 4. 测试失败自动修复

Phase 7发现测试失败，自动进入Phase 8启动Debugger，不要手动干预。

### 5. 状态持久化

维护 `WORKFLOW_STATE.json`，确保流程可恢复。

---

## 注意事项

### 不要跳过Phase

必须按顺序执行所有Phase，除非用户明确要求只执行某些阶段。

### 不要同时运行多个Orchestrator

这个skill应该在主会话中运行，不要用Task启动多个。

### Git Worktrees是必需的

Phase 5的并行开发必须使用git worktrees，否则会导致代码冲突。

### 失败重试有限制

每个Phase最多重试2-3次，超过则提示用户介入。

---

## 与其他Skills的配合

此skill会调用其他skills：

| Phase | 调用的Skill |
|-------|-----------|
| Phase 5 | `dispatching-parallel-agents` - 并行调度Developer |
| Developer内部 | `using-git-worktrees`, `test-driven-development`, `finishing-a-development-branch` |
| Phase 9 | `verification-before-completion`, `requesting-code-review` |

---

## 总结

`orchestrate-workflow` skill是整个自动化开发系统的大脑，它：

1. ✅ 在主会话中运行（不是subagent）
2. ✅ 使用Task tool调度7个专门的subagent
3. ✅ 使用TodoWrite管理全局进度
4. ✅ 使用Git Worktrees实现并行开发
5. ✅ 自动处理测试失败和bug修复
6. ✅ 生成完整的交付文档

**这是真正的软件开发自动化！** 🚀
