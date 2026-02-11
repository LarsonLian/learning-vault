# LangGraph 核心概念与工程实践指南

> 构建复杂 AI Agent 工作流的完整技术参考

**目标读者:** 从初学者到中级开发者
**核心场景:** 多步骤任务、决策树、人机协作的 AI Agent 工作流
**贯穿案例:** 智能项目管理助手

---

## 第一篇:基础知识篇

建立对 LangGraph 的正确心智模型

### 第 1 章:LangGraph 是什么

#### 1.1 问题背景:为什么需要 LangGraph?

**LLM 应用的演进之路**

在 LLM 应用开发的早期阶段,我们通常从简单的单次调用开始:

```
用户输入 → LLM → 输出结果
```

这种模式适用于简单的问答、文本生成等场景。但当我们尝试构建更复杂的应用时,很快就会遇到局限:

1. **缺乏记忆**: 每次调用都是独立的,无法记住之前的对话或状态
2. **无法决策**: 不能根据输出结果动态选择下一步操作
3. **难以循环**: 无法实现"生成→评估→改进"这样的迭代流程
4. **工具调用受限**: 虽然可以让 LLM 调用工具,但缺乏对调用流程的精细控制

**从 Chain 到 Graph 的必然性**

为了解决单次调用的局限,LangChain 引入了 Chain 的概念,将多个 LLM 调用串联起来:

```
输入 → LLM调用1 → LLM调用2 → LLM调用3 → 输出
```

Chain 虽然实现了基本的串联,但仍然存在本质局限:

| 局限 | 具体表现 | 影响 |
|------|---------|------|
| **线性结构** | 只能按预定顺序执行 | 无法实现条件分支和动态路由 |
| **无状态** | 步骤间靠参数传递 | 难以管理复杂的上下文信息 |
| **不支持循环** | 无法返回到之前的步骤 | 无法实现迭代优化和自我修正 |
| **人机协作困难** | 缺乏中断和恢复机制 | 无法在关键点插入人工审批 |
| **难以调试** | 缺少执行历史 | 问题排查困难,无法回溯状态 |

**真实场景的挑战**

考虑一个智能客服场景:

```
用户咨询 → 理解意图 →
  ├─ 简单问题 → 直接回答
  ├─ 复杂问题 → 查询知识库 → 生成答案 → 质量检查 →
  │                                        ├─ 通过 → 返回
  │                                        └─ 不通过 → 重新生成
  └─ 高风险操作 → 生成方案 → 人工审批 → 执行
```

这样的流程需要:
- **状态管理**: 记住用户信息、历史消息、当前进度
- **条件路由**: 根据意图分类跳转到不同分支
- **循环迭代**: 答案质量不佳时重新生成
- **人机协作**: 高风险操作需要人工介入
- **可恢复性**: 系统重启后能从中断点继续

Chain 无法优雅地实现这些需求,这就是为什么我们需要 LangGraph。

#### 1.2 LangGraph 的定位

**核心定义**

LangGraph 是一个**低级编排框架**,用于构建、管理和部署**长时间运行的、有状态的 AI Agent**。

让我们拆解这个定义:

- **低级编排框架**: 不是高级抽象,而是提供底层控制能力,你可以精确定义每个节点的行为和连接关系
- **长时间运行**: 支持跨多天、多次会话的工作流,不局限于单次请求-响应
- **有状态**: 内置状态管理机制,自动持久化和恢复
- **AI Agent**: 面向 Agent 场景优化,原生支持 LLM、工具调用、人机协作

**核心抽象:有向状态图**

LangGraph 将 Agent 工作流建模为**有向状态图**(Directed Stateful Graph):

```
图 = 节点(Nodes) + 边(Edges) + 状态(State)

节点: 执行具体任务的函数(LLM调用、工具执行、数据处理等)
边: 定义节点间的连接关系(顺序边、条件边)
状态: 在节点间流转的数据结构,记录工作流的完整上下文
```

这个抽象的优势:

1. **可视化**: 工作流结构一目了然,易于理解和沟通
2. **模块化**: 每个节点独立实现,易于测试和复用
3. **灵活性**: 支持任意复杂的控制流(分支、循环、并行)
4. **可追溯**: 状态演进有完整历史,便于调试和审计

**与传统工作流引擎的定位差异**

LangGraph 不是通用工作流引擎(如 Airflow、Temporal),而是**LLM 原生的 Agent 编排框架**:

| 维度 | 传统工作流引擎 | LangGraph |
|------|--------------|-----------|
| **设计目标** | 数据管道、批处理任务 | AI Agent、人机协作工作流 |
| **状态管理** | 通常是任务元数据 | 完整的对话历史、上下文、LLM 输出 |
| **动态性** | 图结构预定义,运行时固定 | 支持 LLM 动态决策路由 |
| **可观测性** | 任务执行日志 | 状态演进历史,支持时间旅行调试 |
| **人机协作** | 通过外部审批系统 | 内置 Interrupt 机制,原生支持 |
| **LLM 集成** | 需要自己封装 | 原生集成,提供工具调用、消息管理等 |

**定位总结**

LangGraph 定位在**可控的 Agent 编排层**:

- 比简单 Chain 更强大:支持复杂控制流、状态管理、人机协作
- 比通用工作流引擎更专注:为 LLM 场景深度优化
- 比全自动 Agent 更可控:开发者精确定义每个决策点

#### 1.3 核心价值

LangGraph 的三大核心价值:

**1. 可控的循环与迭代**

在 LangGraph 中,循环是一等公民:

```
生成初稿 → 质量评估 →
  ├─ 质量合格 → 结束
  └─ 质量不合格 → 改进建议 → 重新生成(返回"生成初稿")
```

实现方式:

```python
# 伪代码
state = {
  draft: "",
  quality_score: 0,
  iteration_count: 0
}

def generate_draft(state):
  draft = LLM.generate(...)
  return {draft: draft, iteration_count: state.iteration_count + 1}

def evaluate_quality(state):
  score = LLM.evaluate(state.draft)
  return {quality_score: score}

def should_continue(state):
  if state.quality_score > 0.8 or state.iteration_count >= 3:
    return "END"
  else:
    return "generate_draft"  # 循环回去

# 构建图
graph.add_node("generate", generate_draft)
graph.add_node("evaluate", evaluate_quality)
graph.add_conditional_edge("evaluate", should_continue)
```

关键价值:
- **有界循环**: 可设置最大迭代次数,避免无限循环
- **状态累积**: 每次迭代的结果都记录在状态中
- **条件终止**: 灵活的终止条件判断

**2. 持久化状态与恢复**

LangGraph 内置 Checkpointer 机制,自动保存每个节点执行后的状态:

```
执行节点A → 保存检查点1 → 执行节点B → 保存检查点2 → ...
```

核心能力:

| 能力 | 说明 | 应用场景 |
|------|------|---------|
| **断点续传** | 系统崩溃后从最后一个检查点恢复 | 长时间运行的任务 |
| **时间旅行** | 回到任意历史检查点查看状态 | 调试、问题分析 |
| **执行历史** | 完整记录状态演进过程 | 审计、回溯分析 |
| **多会话管理** | 每个用户独立的执行线程 | 多用户并发场景 |

存储后端:

```python
# 伪代码
# 开发/测试:内存存储
checkpointer = InMemorySaver()

# 生产环境:持久化存储
checkpointer = SQLiteSaver("checkpoints.db")
# 或
checkpointer = PostgresSaver(connection_string)

graph = graph_builder.compile(checkpointer=checkpointer)
```

**3. 原生的人机协作(Human-in-the-Loop)**

LangGraph 提供 `interrupt_before` 和 `interrupt_after` 机制,让 Agent 在关键点暂停,等待人类输入:

```
准备操作 → interrupt(等待审批) →
  ├─ 人类批准 → 执行操作
  └─ 人类拒绝 → 取消操作
```

实现方式:

```python
# 伪代码
# 在"执行删除"节点前中断
graph = graph_builder.compile(
  checkpointer=checkpointer,
  interrupt_before=["execute_deletion"]
)

# 执行到删除节点前会自动暂停
result = graph.invoke({...}, config=config)
# result.status = "interrupted"

# 获取当前状态,展示给用户
current_state = graph.get_state(config)

# 用户审批后,继续执行
graph.invoke(None, config=config)  # 从中断点继续
```

人机协作的典型场景:
- **风险操作审批**: 删除数据、发送邮件、执行支付
- **不确定性决策**: LLM 置信度低时,让人类选择
- **创意性工作**: 设计方案生成后,让人类修改和确认
- **异常处理**: 遇到预期外的情况,请求人工介入

#### 1.4 适用场景 vs 不适用场景

**✅ 适用场景**

| 场景类型 | 具体示例 | 为什么适合 LangGraph |
|---------|---------|-------------------|
| **多步骤推理** | 复杂问题分解、逐步求解 | 需要状态在多步骤间传递 |
| **迭代优化** | 文章写作→评审→改进→再评审 | 需要循环和质量门控 |
| **条件分支** | 客服:简单问题直接答、复杂问题转人工 | 需要动态路由 |
| **工具编排** | 调研助手:搜索→提取→分析→总结 | 需要管理多工具调用流程 |
| **人机协作** | 法律文书生成→律师审核→修改→定稿 | 需要 interrupt 机制 |
| **长时间运行** | 项目管理:创建任务→分解→分配→跟踪(跨多天) | 需要持久化状态 |
| **状态机** | 订单处理:待支付→待发货→待收货→已完成 | 天然映射到状态图 |

**❌ 不适用场景**

| 场景类型 | 为什么不适合 | 推荐方案 |
|---------|------------|----------|
| **简单单次调用** | LangGraph 太重,引入不必要的复杂度 | 直接调用 LLM API 或使用简单 Chain |
| **纯数据转换管道** | 不需要 LLM 动态决策,只需固定的数据流 | Airflow、Prefect 等数据管道工具 |
| **高性能批处理** | LangGraph 优化目标是灵活性而非吞吐量 | Spark、Flink 等批处理框架 |
| **实时低延迟** | Checkpointer 持久化会引入延迟 | 内存计算、流处理框架 |
| **完全自主 Agent** | 你需要 LangGraph 提供的可控性 | ReAct Agent、AutoGPT 等 |

**判断标准**

问自己这些问题:

1. **是否需要多步骤?** 如果是单次 LLM 调用就能完成,不需要 LangGraph
2. **是否需要状态管理?** 如果步骤间需要共享复杂的上下文,使用 LangGraph
3. **是否需要条件分支或循环?** 如果控制流是动态的,使用 LangGraph
4. **是否需要人工介入?** 如果关键点需要人类决策,使用 LangGraph
5. **是否需要可恢复性?** 如果任务可能长时间运行且不能丢失进度,使用 LangGraph

**最佳实践建议**

- **从简单开始**: 先用简单 Chain 实现,确实遇到局限时再迁移到 LangGraph
- **渐进式采用**: 可以在 LangGraph 节点中调用现有的 Chain
- **混合使用**: 简单部分用 Chain,复杂部分用 LangGraph,不是非此即彼

**小结**

LangGraph 不是银弹,它解决的是**复杂 Agent 工作流的可控编排**问题。理解它的适用场景,可以帮助你在合适的时候选择合适的工具,避免过度设计或功能不足。

---

### 第 2 章:核心抽象

LangGraph 的设计围绕四个核心抽象展开:StateGraph(状态图)、State(状态)、Node(节点)和 Edge(边)。理解这四个概念之间的关系,是掌握 LangGraph 的关键。

#### 2.1 StateGraph:有状态的有向图

**概念**

StateGraph 是 LangGraph 的顶层抽象,代表一个**有向图**(Directed Graph),图中的每个节点都可以读取和更新共享的状态。

```
StateGraph = 图结构 + 状态管理

图结构: 定义节点和边的连接关系
状态管理: 自动在节点间传递和更新状态
```

**与普通有向图的区别**

| 特性 | 普通有向图 | StateGraph |
|------|----------|------------|
| **数据传递** | 通过边传递参数 | 通过共享状态传递 |
| **节点输入** | 前置节点的输出 | 当前完整状态 |
| **节点输出** | 传递给后续节点 | 状态更新(部分或全部字段) |
| **历史追踪** | 通常不保留 | 自动保存每个状态版本 |

**简单示例:三节点图**

```python
# 伪代码示例
# 定义状态结构
class State(TypedDict):
  user_input: str
  processed_data: str
  final_output: str

# 创建StateGraph
graph_builder = StateGraph(State)

# 添加三个节点
graph_builder.add_node("collect", collect_input)
graph_builder.add_node("process", process_data)
graph_builder.add_node("respond", generate_response)

# 添加边,定义执行顺序
graph_builder.add_edge(START, "collect")
graph_builder.add_edge("collect", "process")
graph_builder.add_edge("process", "respond")
graph_builder.add_edge("respond", END)

# 编译成可执行的图
graph = graph_builder.compile()
```

执行流程:

```
START → collect(读取State,更新user_input) →
        process(读取user_input,更新processed_data) →
        respond(读取processed_data,更新final_output) →
        END
```

关键点:
- 每个节点都能访问完整的 State
- 节点只需返回要更新的字段,不需要返回整个状态
- StateGraph 负责合并更新到当前状态

#### 2.2 State:Agent 的记忆结构

**概念**

State 是在节点间流转的数据结构,代表 Agent 的**完整记忆**。它通常用 TypedDict 定义,明确每个字段的类型。

```python
# 伪代码
class AgentState(TypedDict):
  messages: list[Message]      # 对话历史
  current_task: str             # 当前任务描述
  task_status: str              # 任务状态
  tools_used: list[str]         # 已使用的工具
  iteration_count: int          # 迭代次数
```

**状态更新策略**

LangGraph 提供三种状态更新策略:

**1. 覆盖模式(Override)** - 默认行为

```python
# 默认情况下,新值覆盖旧值
class State(TypedDict):
  counter: int
  status: str

# 节点返回 {"counter": 5}
# State中的counter字段被直接替换为5
```

**2. 累加模式(Add)** - 使用 Annotated + reducer

```python
# 使用operator.add作为reducer
from typing import Annotated
from operator import add

class State(TypedDict):
  items: Annotated[list, add]  # 列表累加

# 初始 state = {"items": [1, 2]}
# 节点返回 {"items": [3, 4]}
# 结果 state = {"items": [1, 2, 3, 4]}  # 自动合并
```

**3. 自定义 reducer** - 完全自定义合并逻辑

```python
# 伪代码
def custom_merge_messages(existing: list, new: list) -> list:
  """自定义消息合并:去重 + 限制数量"""
  all_messages = existing + new
  # 去重(基于消息ID)
  unique = deduplicate_by_id(all_messages)
  # 只保留最近100条
  return unique[-100:]

class State(TypedDict):
  messages: Annotated[list[Message], custom_merge_messages]
```

**内置 reducer:add_messages**

LangGraph 为对话场景提供了专用的 `add_messages` reducer:

```python
from langgraph.graph.message import add_messages

class State(TypedDict):
  messages: Annotated[list[Message], add_messages]

# add_messages 的特殊能力:
# 1. 自动将字典转换为Message对象
# 2. 根据消息ID更新已有消息(而不是追加)
# 3. 正确处理tool_calls和tool_results
```

**状态访问模式**

节点函数的输入是只读的状态副本:

```python
def my_node(state: State) -> dict:
  # state 是当前状态的快照
  # 读取状态
  current_count = state["counter"]

  # 不要直接修改 state!
  # state["counter"] += 1  # ❌ 错误!

  # 返回更新
  return {"counter": current_count + 1}  # ✅ 正确
```

#### 2.3 Node:执行单元

**概念**

Node(节点)是图中的执行单元,本质上是一个**函数**,负责执行具体的任务。

**节点函数签名**

```python
def node_function(state: StateType) -> dict | StateType:
  """
  输入: 当前完整状态
  输出: 状态更新(字典)
  """
  # 1. 读取状态中需要的信息
  user_input = state["user_input"]

  # 2. 执行任务(LLM调用、工具执行、数据处理等)
  result = do_something(user_input)

  # 3. 返回状态更新
  return {"processed_result": result}
```

**节点的典型实现模式**

**模式1:LLM 调用节点**

```python
def llm_node(state: State) -> dict:
  """调用LLM生成响应"""
  messages = state["messages"]
  response = llm.invoke(messages)
  return {"messages": [response]}  # 追加到消息列表
```

**模式2:工具执行节点**

```python
def tool_node(state: State) -> dict:
  """执行工具调用"""
  tool_call = state["pending_tool_call"]
  tool_result = execute_tool(tool_call.name, tool_call.args)
  return {
    "tool_results": [tool_result],
    "pending_tool_call": None
  }
```

**模式3:决策节点**

```python
def decision_node(state: State) -> dict:
  """分析并做出决策"""
  complexity = analyze_complexity(state["task"])
  return {
    "task_complexity": complexity,
    "route_decision": "complex" if complexity > 0.7 else "simple"
  }
```

**模式4:聚合节点**

```python
def aggregator_node(state: State) -> dict:
  """聚合并行任务的结果"""
  results = state["parallel_results"]
  summary = combine_results(results)
  return {"final_summary": summary}
```

**节点的关键特性**

1. **无状态**: 节点函数本身不保存状态,所有上下文来自输入的 state
2. **幂等性**: 同样的输入应该产生同样的输出(LLM调用除外)
3. **单一职责**: 每个节点只做一件事,保持简单
4. **可测试**: 易于单独测试,只需构造 state 输入

#### 2.4 Edge:连接规则

Edge(边)定义节点之间的连接关系,决定执行流程。LangGraph 支持两种边:

**1. 普通边(Normal Edge)** - 固定的连接

```python
# 伪代码
graph_builder.add_edge("node_a", "node_b")
# 表示:node_a执行完后,总是执行node_b
```

执行流程确定:

```
node_a → node_b → node_c
```

**2. 条件边(Conditional Edge)** - 动态的路由

```python
# 伪代码
def router_function(state: State) -> str:
  """根据状态决定下一个节点"""
  if state["confidence"] > 0.9:
    return "high_confidence_handler"
  elif state["confidence"] > 0.5:
    return "medium_confidence_handler"
  else:
    return "low_confidence_handler"

graph_builder.add_conditional_edges(
  "decision_node",           # 来源节点
  router_function,           # 路由函数
  {                          # 路由映射
    "high_confidence_handler": "execute_directly",
    "medium_confidence_handler": "request_review",
    "low_confidence_handler": "ask_user"
  }
)
```

执行流程动态:

```
decision_node →
  ├─ 高置信度 → execute_directly
  ├─ 中置信度 → request_review
  └─ 低置信度 → ask_user
```

**特殊节点:START 和 END**

```python
from langgraph.graph import START, END

# START: 图的入口点
graph_builder.add_edge(START, "first_node")

# END: 图的出口点
graph_builder.add_edge("last_node", END)
```

**条件边的高级用法**

```python
# 实现循环:条件边指向之前的节点
def should_continue(state: State) -> str:
  if state["iteration_count"] < 3:
    return "generate_again"  # 回到之前的节点
  else:
    return "finalize"        # 继续向前

graph_builder.add_conditional_edges(
  "evaluate",
  should_continue,
  {
    "generate_again": "generate",  # 循环回去
    "finalize": "finalize"         # 结束循环
  }
)
```

流程图:

```
generate → evaluate →
    ↑         ↓
    └─────────┘ (循环,如果iteration_count < 3)
           OR
           ↓
        finalize (结束循环)
```

#### 2.5 核心抽象之间的关系

**关系图**

```
┌─────────────────────────────────────────┐
│            StateGraph                   │  顶层容器
│  ┌─────────────────────────────────┐   │
│  │         State                   │   │  共享记忆
│  │  ┌──────────────────────────┐   │   │
│  │  │ messages: list           │   │   │
│  │  │ task_status: str         │   │   │
│  │  │ iteration_count: int     │   │   │
│  │  └──────────────────────────┘   │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌────────┐  Edge   ┌────────┐         │
│  │ Node A │ ──────→ │ Node B │         │  节点和边
│  └────────┘         └────────┘         │
│      ↓                   ↓              │
│   读取State           读取State         │
│   返回更新            返回更新          │
└─────────────────────────────────────────┘
```

**协同工作流程**

1. **初始化**: 用户提供初始状态 → StateGraph 创建状态快照
2. **节点执行**: 节点读取当前状态 → 执行任务 → 返回状态更新
3. **状态合并**: StateGraph 将节点返回的更新合并到状态中
4. **边路由**: 根据边的定义,决定下一个执行的节点
5. **持久化**: (如果启用 Checkpointer)保存当前状态到存储
6. **重复**: 继续执行下一个节点,直到到达 END

**数据流示例**

```
初始状态: {"messages": [], "count": 0}

↓ 执行 node_1
node_1 返回: {"messages": ["Hello"], "count": 1}
合并后状态: {"messages": ["Hello"], "count": 1}

↓ 执行 node_2
node_2 返回: {"messages": ["World"]}  # 只更新messages
合并后状态: {"messages": ["Hello", "World"], "count": 1}  # count保持不变

↓ 到达 END
最终状态: {"messages": ["Hello", "World"], "count": 1}
```

**关键理解**

1. **State 是中心**: 所有节点都围绕 State 工作,读取它、更新它
2. **Node 是无状态的**: 节点不存储数据,只负责转换
3. **Edge 是控制流**: 边决定执行顺序,但不传递数据
4. **StateGraph 是协调者**: 负责状态管理、节点调度、历史记录

**与传统编程的对比**

| 传统编程 | LangGraph |
|---------|-----------|
| 函数通过参数和返回值传递数据 | 节点通过共享状态传递数据 |
| if-else 控制流程 | 条件边控制流程 |
| 变量保存状态 | State TypedDict 保存状态 |
| 手动管理状态历史 | StateGraph 自动持久化 |

**小结**

LangGraph 的四个核心抽象形成了一个优雅的模型:

- **StateGraph** 是容器,管理整个工作流
- **State** 是记忆,在节点间共享上下文
- **Node** 是执行单元,实现具体逻辑
- **Edge** 是路由,控制执行顺序

理解这四者的关系,就能用 LangGraph 构建任意复杂的 Agent 工作流。

---

### 第 3 章:与传统工作流引擎的区别

理解 LangGraph 与传统工作流引擎(如 Airflow、Temporal、Prefect)的区别,能帮助你在合适的场景选择合适的工具。

#### 3.1 核心设计理念对比

| 维度 | 传统工作流引擎<br/>(Airflow/Temporal) | LangGraph |
|------|--------------------------------|-----------|
| **设计目标** | 数据管道编排、批处理任务调度 | AI Agent 工作流编排 |
| **核心抽象** | DAG(有向无环图) | Stateful Graph(有向状态图) |
| **循环支持** | 不支持(必须是无环图) | 原生支持(条件边可回指) |
| **状态管理** | 任务元数据、XCom(轻量) | 完整的对话历史、LLM 上下文 |
| **动态路由** | 有限支持,需预定义分支 | 完全动态,LLM 可在运行时决策 |
| **执行模式** | 主要是批处理、定时触发 | 交互式、事件驱动、长时间运行 |

#### 3.2 详细功能对比

**状态管理**

```
传统工作流引擎:
- 主要存储任务元数据(状态、开始时间、结束时间等)
- XCom 用于任务间传递小数据(通常< 1MB)
- 不适合存储复杂的对话历史

LangGraph:
- 状态是第一等公民,可以存储任意复杂的数据结构
- 内置对消息列表的优化(add_messages reducer)
- 支持多种存储后端(内存、SQLite、PostgreSQL)
```

**动态性与灵活性**

```
传统工作流引擎:
Task A → Task B →
  ├─ 条件1 → Task C
  └─ 条件2 → Task D
# 所有任务和分支必须预先定义

LangGraph:
Node A → Node B →
  ├─ LLM决策 → 动态选择 Node C/D/E
  └─ 甚至可以动态创建新节点
# 路由逻辑可以由 LLM 在运行时决定
```

**人机协作**

| 特性 | 传统工作流引擎 | LangGraph |
|------|--------------|-----------|
| **审批机制** | 通过外部系统(邮件、Slack等) | 内置 `interrupt` 机制 |
| **状态保存** | 需要额外配置 | 自动保存到检查点 |
| **恢复执行** | 复杂,需要手动触发 | 简单,调用 `invoke` 继续 |
| **上下文传递** | 有限,需要手动序列化 | 完整保留,包括 LLM 上下文 |

**LLM 原生集成**

```
传统工作流引擎:
def llm_task():
  # 需要自己处理:
  # - LLM API 调用
  # - 消息格式化
  # - 工具调用解析
  # - 错误重试
  llm_result = call_openai_api(...)
  return process_result(llm_result)

LangGraph:
def llm_node(state: State) -> dict:
  # 内置支持:
  # - 消息管理(add_messages)
  # - 工具调用(ToolNode)
  # - Streaming
  response = llm.invoke(state["messages"])
  return {"messages": [response]}
```

#### 3.3 适用场景对比

**使用传统工作流引擎的场景:**

1. **数据管道**: ETL 任务、数据同步、报表生成
2. **定时批处理**: 每日数据汇总、定期清理任务
3. **资源密集型任务**: 大规模数据处理、分布式计算
4. **明确的依赖关系**: 任务 A 完成后才能执行任务 B,无循环
5. **团队熟悉度**: 数据工程团队已经在使用 Airflow

**使用 LangGraph 的场景:**

1. **AI Agent 工作流**: 多步骤推理、工具调用、迭代优化
2. **对话系统**: 聊天机器人、智能客服、虚拟助手
3. **需要循环**: 生成→评估→改进的迭代流程
4. **人机协作**: 需要在关键点等待人工审批或输入
5. **动态决策**: 下一步行动由 LLM 根据上下文决定
6. **长时间会话**: 跨多天、多次交互的任务

#### 3.4 混合使用策略

实际项目中,两者可以互补:

```
场景:智能数据分析平台

Airflow 负责:
- 每日数据采集和预处理(批处理)
- 定时触发 LangGraph 工作流

LangGraph 负责:
- 接收用户查询
- 动态生成 SQL
- 迭代优化查询结果
- 生成分析报告

协作方式:
Airflow Task → 触发 LangGraph API → LangGraph 执行 Agent 流程 → 返回结果 → Airflow 后续任务
```

#### 3.5 LangGraph 的独特性总结

LangGraph 相比传统工作流引擎的核心优势:

1. **LLM 原生**: 为 LLM 场景深度优化,不需要额外封装
2. **状态丰富**: 可以轻松管理复杂的对话历史和上下文
3. **支持循环**: 天然支持迭代式工作流
4. **可观测性**: 时间旅行调试,完整的状态演进历史
5. **人机协作**: 内置 interrupt 机制,无缝集成
6. **动态性**: LLM 可以在运行时决定路由

**不要用 LangGraph 做的事:**

- ❌ 大规模批处理数据(每天处理TB级数据)
- ❌ 纯 ETL 任务(数据库同步、数据清洗)
- ❌ 需要极致性能的场景(毫秒级响应)
- ❌ 简单的定时任务(每小时发一封邮件)

---

### 第 4 章:快速上手示例

通过一个完整的示例,快速体验 LangGraph 的核心概念。

#### 4.1 示例目标

构建一个**智能消息处理器**,实现以下功能:

1. 收集用户输入
2. 分析消息情感(正面/负面/中性)
3. 根据情感选择不同的回复策略
4. 生成最终回复

这个简单示例涵盖:
- ✅ 状态管理(消息、情感、回复)
- ✅ 条件路由(根据情感分支)
- ✅ 多节点协作

#### 4.2 完整代码

```python
# 伪代码:智能消息处理器

from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END

# 步骤1:定义状态
class MessageState(TypedDict):
    user_input: str          # 用户输入
    sentiment: str           # 情感分析结果
    response: str            # 最终回复

# 步骤2:定义节点函数

def collect_input(state: MessageState) -> dict:
    """收集用户输入(在实际应用中,这会从API获取)"""
    # 这里简化为直接使用state中的user_input
    print(f"收到消息: {state['user_input']}")
    return {}  # 不更新状态,只是打印

def analyze_sentiment(state: MessageState) -> dict:
    """分析消息情感"""
    user_input = state["user_input"]

    # 简化的情感分析(实际会调用LLM或ML模型)
    if "好" in user_input or "谢谢" in user_input:
        sentiment = "positive"
    elif "差" in user_input or "问题" in user_input:
        sentiment = "negative"
    else:
        sentiment = "neutral"

    print(f"情感分析结果: {sentiment}")
    return {"sentiment": sentiment}

def generate_positive_response(state: MessageState) -> dict:
    """生成正面回复"""
    response = f"很高兴您满意!我们会继续努力。"
    print(f"正面回复: {response}")
    return {"response": response}

def generate_negative_response(state: MessageState) -> dict:
    """生成负面回复"""
    response = f"非常抱歉给您带来不便,我们会立即处理您的问题。"
    print(f"负面回复: {response}")
    return {"response": response}

def generate_neutral_response(state: MessageState) -> dict:
    """生成中性回复"""
    response = f"感谢您的反馈,有什么我可以帮您的吗?"
    print(f"中性回复: {response}")
    return {"response": response}

# 步骤3:定义路由函数

def route_by_sentiment(state: MessageState) -> Literal["positive", "negative", "neutral"]:
    """根据情感路由到不同的回复节点"""
    return state["sentiment"]

# 步骤4:构建图

graph_builder = StateGraph(MessageState)

# 添加节点
graph_builder.add_node("collect", collect_input)
graph_builder.add_node("analyze", analyze_sentiment)
graph_builder.add_node("positive_response", generate_positive_response)
graph_builder.add_node("negative_response", generate_negative_response)
graph_builder.add_node("neutral_response", generate_neutral_response)

# 添加边
graph_builder.add_edge(START, "collect")
graph_builder.add_edge("collect", "analyze")

# 添加条件边:根据情感路由
graph_builder.add_conditional_edges(
    "analyze",                    # 源节点
    route_by_sentiment,           # 路由函数
    {                             # 路由映射
        "positive": "positive_response",
        "negative": "negative_response",
        "neutral": "neutral_response"
    }
)

# 所有回复节点都指向END
graph_builder.add_edge("positive_response", END)
graph_builder.add_edge("negative_response", END)
graph_builder.add_edge("neutral_response", END)

# 步骤5:编译图
graph = graph_builder.compile()

# 步骤6:运行示例

# 示例1:正面消息
result1 = graph.invoke({
    "user_input": "你们的服务真好,谢谢!",
    "sentiment": "",
    "response": ""
})
print(f"\n最终状态: {result1}\n")

# 示例2:负面消息
result2 = graph.invoke({
    "user_input": "有问题,体验很差",
    "sentiment": "",
    "response": ""
})
print(f"\n最终状态: {result2}\n")

# 示例3:中性消息
result3 = graph.invoke({
    "user_input": "你好",
    "sentiment": "",
    "response": ""
})
print(f"\n最终状态: {result3}\n")
```

#### 4.3 逐行解释

**定义状态 (State)**

```python
class MessageState(TypedDict):
    user_input: str
    sentiment: str
    response: str
```

- 使用 `TypedDict` 明确定义状态结构
- 每个字段都有明确的类型
- 这个状态会在所有节点间共享

**定义节点函数**

```python
def analyze_sentiment(state: MessageState) -> dict:
    user_input = state["user_input"]  # 读取状态
    sentiment = ...                   # 执行分析
    return {"sentiment": sentiment}   # 返回状态更新
```

- 节点函数输入是完整的 `state`
- 可以读取 `state` 中的任何字段
- 返回一个字典,只包含要更新的字段
- 不需要返回整个状态

**定义路由函数**

```python
def route_by_sentiment(state: MessageState) -> str:
    return state["sentiment"]  # 返回节点名称
```

- 路由函数也接收完整的 `state`
- 返回值是字符串,对应下一个节点的名称
- LangGraph 会根据返回值选择下一个节点

**构建图结构**

```python
graph_builder = StateGraph(MessageState)
graph_builder.add_node("analyze", analyze_sentiment)
graph_builder.add_edge(START, "collect")
graph_builder.add_conditional_edges("analyze", route_by_sentiment, {...})
```

- `StateGraph(MessageState)`: 创建图构建器,指定状态类型
- `add_node(名称, 函数)`: 添加节点
- `add_edge(源, 目标)`: 添加固定边
- `add_conditional_edges(源, 路由函数, 映射)`: 添加条件边

**编译和执行**

```python
graph = graph_builder.compile()
result = graph.invoke(initial_state)
```

- `compile()`: 将构建器编译成可执行的图
- `invoke(initial_state)`: 执行图,传入初始状态
- 返回最终状态

#### 4.4 运行与输出

**执行流程可视化**

```
示例1:正面消息 "你们的服务真好,谢谢!"

START
  ↓
collect (打印:收到消息)
  ↓
analyze (分析情感 → "positive")
  ↓
[条件路由] sentiment == "positive"
  ↓
positive_response (生成:很高兴您满意!)
  ↓
END

最终状态:
{
  "user_input": "你们的服务真好,谢谢!",
  "sentiment": "positive",
  "response": "很高兴您满意!我们会继续努力。"
}
```

```
示例2:负面消息 "有问题,体验很差"

START → collect → analyze → [路由:negative] → negative_response → END

最终状态:
{
  "user_input": "有问题,体验很差",
  "sentiment": "negative",
  "response": "非常抱歉给您带来不便,我们会立即处理您的问题。"
}
```

**状态演进追踪**

```
初始状态:
{"user_input": "谢谢", "sentiment": "", "response": ""}

经过 collect 节点:
{"user_input": "谢谢", "sentiment": "", "response": ""}
(无变化,只打印)

经过 analyze 节点:
{"user_input": "谢谢", "sentiment": "positive", "response": ""}
(更新了 sentiment)

经过 positive_response 节点:
{"user_input": "谢谢", "sentiment": "positive", "response": "很高兴您满意!..."}
(更新了 response)

到达 END,返回最终状态
```

#### 4.5 关键学习点

通过这个示例,你应该理解:

1. **状态驱动**: 整个工作流围绕 `State` 展开,节点读取和更新状态
2. **节点独立**: 每个节点只负责一件事,保持简单
3. **条件路由**: 通过路由函数实现动态分支
4. **声明式构建**: 先定义节点和边,最后编译执行
5. **部分更新**: 节点只需返回要更新的字段,不是整个状态

#### 4.6 扩展思考

基于这个基础示例,可以轻松扩展:

**添加 LLM 调用**

```python
def analyze_sentiment(state: MessageState) -> dict:
    # 用 LLM 替代简单的关键词匹配
    prompt = f"分析以下消息的情感(positive/negative/neutral): {state['user_input']}"
    sentiment = llm.invoke(prompt)
    return {"sentiment": sentiment}
```

**添加循环迭代**

```python
# 如果回复质量不好,重新生成
def should_regenerate(state: MessageState) -> str:
    quality_score = evaluate_quality(state["response"])
    if quality_score < 0.7:
        return "analyze"  # 回到分析节点
    else:
        return END
```

**添加持久化**

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
graph = graph_builder.compile(checkpointer=checkpointer)

# 带线程ID执行,支持恢复
config = {"configurable": {"thread_id": "user_123"}}
result = graph.invoke(initial_state, config=config)
```

**小结**

这个快速上手示例展示了 LangGraph 的核心工作方式。虽然简单,但已经包含了状态管理、条件路由等关键概念。在后续章节中,我们会深入学习更高级的特性。

---

## 第二篇:核心能力深入

每章遵循"概念→实战→最佳实践"三段式结构

### 第 5 章:状态管理 - 设计你的 Agent 记忆

状态(State)是 LangGraph 的核心,它决定了 Agent 能记住什么、如何记住、记住多久。良好的状态设计是构建可靠 Agent 的基础。

#### 5.1 概念讲解

##### 5.1.1 State Schema 设计原则

State Schema 定义了 Agent 的"记忆结构"。设计时需要考虑:

**原则1:包含所有必要信息**

```python
# ❌ 不好:信息不足
class State(TypedDict):
    messages: list

# ✅ 好:包含完整上下文
class State(TypedDict):
    messages: list[Message]       # 对话历史
    current_task: str             # 当前任务
    user_context: dict            # 用户信息
    iteration_count: int          # 迭代次数
```

**原则2:避免冗余字段**

```python
# ❌ 不好:同一信息的多种表示
class State(TypedDict):
    task_description: str
    task_title: str               # 冗余
    task_summary: str             # 冗余

# ✅ 好:单一信息源
class State(TypedDict):
    task: Task  # 包含所有任务信息的对象
```

**原则3:合理的字段类型**

```python
# 基础类型
class State(TypedDict):
    count: int
    score: float
    is_approved: bool
    status: Literal["pending", "approved", "rejected"]

# 复合类型
class State(TypedDict):
    messages: list[Message]
    metadata: dict[str, any]
    tools_used: set[str]

# 自定义类型
class TaskInfo(TypedDict):
    title: str
    priority: int
    deadline: str

class State(TypedDict):
    tasks: list[TaskInfo]
```

**原则4:扁平化优于嵌套**

```python
# ❌ 避免:深层嵌套
class State(TypedDict):
    project: dict  # {"team": {"members": [{"name": "...", "role": "..."}]}}

# ✅ 推荐:扁平结构
class State(TypedDict):
    project_name: str
    team_members: list[dict]  # [{"name": "...", "role": "..."}]
```

##### 5.1.2 Reducer 函数详解

Reducer 函数控制状态字段的更新策略。LangGraph 提供三种模式:

**模式1:覆盖模式(默认)**

```python
class State(TypedDict):
    counter: int  # 默认行为:新值覆盖旧值

# 节点返回 {"counter": 5}
# 状态更新: counter = 5 (直接覆盖)
```

适用场景:
- 标量值(int, float, str, bool)
- 状态标志位
- 最新的单一值

**模式2:累加模式(使用 operator.add)**

```python
from typing import Annotated
from operator import add

class State(TypedDict):
    items: Annotated[list, add]

# 初始: items = [1, 2]
# 节点返回: {"items": [3, 4]}
# 结果: items = [1, 2, 3, 4]  # 自动合并
```

适用场景:
- 列表累积
- 集合合并
- 字符串拼接

**模式3:自定义 Reducer**

```python
def merge_with_deduplication(
    existing: list[str],
    new: list[str]
) -> list[str]:
    """合并列表并去重"""
    return list(set(existing + new))

class State(TypedDict):
    tags: Annotated[list[str], merge_with_deduplication]

# 初始: tags = ["python", "ai"]
# 节点返回: {"tags": ["ai", "ml"]}
# 结果: tags = ["python", "ai", "ml"]  # 去重后
```

**内置 Reducer:add_messages**

专为对话场景设计的高级 reducer:

```python
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

# add_messages 的特殊能力:

# 1. 自动类型转换
node_returns = {"messages": [{"role": "user", "content": "hi"}]}
# 自动转换为 HumanMessage 对象

# 2. 基于ID更新(而不是追加)
# 如果新消息的ID已存在,更新该消息而不是追加
existing_messages = [Message(id="msg1", content="old")]
new_messages = [Message(id="msg1", content="updated")]
# 结果: [Message(id="msg1", content="updated")]  # 更新而非追加

# 3. 正确处理 tool_calls 和 tool_results
# 自动关联工具调用和工具结果
```

**自定义 Reducer 示例:保留最近 N 条**

```python
def keep_recent(n: int):
    """工厂函数:创建保留最近N条的reducer"""
    def reducer(existing: list, new: list) -> list:
        combined = existing + new
        return combined[-n:]  # 只保留最后N个
    return reducer

class State(TypedDict):
    # 只保留最近10条消息
    messages: Annotated[list, keep_recent(10)]
```

##### 5.1.3 状态访问模式

**只读访问**

```python
def my_node(state: State) -> dict:
    # state 是当前状态的不可变副本
    current_count = state["counter"]

    # ❌ 错误:不要直接修改 state
    # state["counter"] += 1

    # ✅ 正确:返回更新
    return {"counter": current_count + 1}
```

**条件更新**

```python
def conditional_update(state: State) -> dict:
    """只在特定条件下更新"""
    if state["score"] > 0.8:
        return {"status": "approved"}
    else:
        return {}  # 返回空字典,不更新任何字段
```

**批量更新**

```python
def batch_update(state: State) -> dict:
    """一次更新多个字段"""
    return {
        "status": "completed",
        "score": 0.95,
        "timestamp": current_time(),
        "result": compute_result(state)
    }
```

#### 5.2 实战片段:项目管理助手的状态设计

让我们设计智能项目管理助手的完整状态结构:

```python
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages
from operator import add

# 任务信息
class Task(TypedDict):
    id: str
    title: str
    description: str
    priority: Literal["high", "medium", "low"]
    status: Literal["pending", "in_progress", "completed"]
    assigned_to: str | None
    estimated_hours: int
    subtasks: list[str]  # 子任务ID列表

# 主状态
class ProjectManagerState(TypedDict):
    # 1. 对话历史 - 使用 add_messages
    messages: Annotated[list, add_messages]

    # 2. 任务列表 - 使用 add 累加
    tasks: Annotated[list[Task], add]

    # 3. 当前上下文 - 覆盖模式
    current_task_id: str | None
    current_operation: Literal["create", "update", "query", "delete"] | None

    # 4. 工作流状态 - 覆盖模式
    workflow_stage: Literal["intake", "analysis", "execution", "review"]
    needs_approval: bool
    approval_context: dict | None

    # 5. 迭代计数 - 覆盖模式
    iteration_count: int
    max_iterations: int

    # 6. 错误处理 - 使用自定义 reducer
    errors: Annotated[list[str], lambda old, new: old[-5:] + new]  # 只保留最近5个错误
```

**状态字段说明**

| 字段 | Reducer | 用途 | 示例值 |
|------|---------|------|--------|
| `messages` | add_messages | 完整对话历史 | [HumanMessage(...), AIMessage(...)] |
| `tasks` | add | 任务累积列表 | [Task(...), Task(...)] |
| `current_task_id` | 覆盖 | 当前操作的任务 | "task_001" |
| `workflow_stage` | 覆盖 | 当前工作流阶段 | "analysis" |
| `needs_approval` | 覆盖 | 是否需要审批 | true |
| `iteration_count` | 覆盖 | 当前迭代次数 | 2 |
| `errors` | 自定义 | 错误历史(最近5条) | ["Error 1", "Error 2"] |

**节点如何使用这个状态**

```python
def create_task_node(state: ProjectManagerState) -> dict:
    """创建任务节点"""
    # 1. 读取对话历史,提取任务信息
    last_message = state["messages"][-1]
    task_info = extract_task_info(last_message)

    # 2. 创建新任务
    new_task = Task(
        id=generate_id(),
        title=task_info["title"],
        priority=task_info.get("priority", "medium"),
        status="pending",
        ...
    )

    # 3. 返回状态更新
    return {
        "tasks": [new_task],              # 追加到任务列表
        "current_task_id": new_task["id"], # 更新当前任务ID
        "workflow_stage": "execution",     # 更新工作流阶段
        "messages": [AIMessage(f"已创建任务: {new_task['title']}")]
    }

def route_by_complexity(state: ProjectManagerState) -> str:
    """根据任务复杂度路由"""
    current_task = find_task(state["tasks"], state["current_task_id"])

    if current_task["priority"] == "high" or current_task["estimated_hours"] > 40:
        return "complex_task_handler"
    else:
        return "simple_task_handler"
```

#### 5.3 最佳实践

##### 5.3.1 保持状态扁平化

**原则**: 避免深层嵌套,优先使用扁平结构

**反例:深层嵌套**

```python
# ❌ 不好:3层嵌套
class State(TypedDict):
    project: dict  # {
                   #   "info": {"name": "...", "id": "..."},
                   #   "team": {
                   #     "members": [{"name": "...", "skills": ["..."]}]
                   #   }
                   # }
```

问题:
- 访问困难: `state["project"]["team"]["members"][0]["skills"]`
- 更新复杂: 需要深拷贝避免意外修改
- 难以追踪变化

**正例:扁平化**

```python
# ✅ 好:扁平结构
class Member(TypedDict):
    name: str
    skills: list[str]

class State(TypedDict):
    project_name: str
    project_id: str
    team_members: list[Member]
```

优势:
- 访问简单: `state["project_name"]`, `state["team_members"]`
- 更新清晰: `return {"project_name": "NewName"}`
- 易于理解

##### 5.3.2 合理使用 Reducer

**原则**: 根据数据特性选择合适的 reducer

**场景1:累积日志/历史**

```python
# ✅ 使用 add
class State(TypedDict):
    logs: Annotated[list[str], add]

# 节点可以持续追加日志
return {"logs": [f"Step {i} completed"]}
```

**场景2:唯一值集合**

```python
# ✅ 使用自定义 reducer 去重
def merge_unique(old: list, new: list) -> list:
    return list(set(old + new))

class State(TypedDict):
    tags: Annotated[list[str], merge_unique]
```

**场景3:最新状态**

```python
# ✅ 使用默认覆盖
class State(TypedDict):
    current_stage: str
    latest_score: float

# 总是使用最新值
return {"current_stage": "review", "latest_score": 0.92}
```

**场景4:有界历史**

```python
# ✅ 使用自定义 reducer 限制大小
def keep_last_n(n: int):
    def reducer(old: list, new: list) -> list:
        return (old + new)[-n:]
    return reducer

class State(TypedDict):
    recent_messages: Annotated[list, keep_last_n(100)]
```

##### 5.3.3 状态版本化策略

**原则**: 在状态结构可能变化时,提供平滑的迁移路径

**策略1:版本号字段**

```python
class State(TypedDict):
    schema_version: int  # 当前状态结构版本
    # ... 其他字段

def migrate_state(state: dict) -> dict:
    """迁移旧版本状态到新版本"""
    version = state.get("schema_version", 1)

    if version == 1:
        # v1 -> v2: 添加新字段
        state["new_field"] = default_value
        state["schema_version"] = 2

    if version == 2:
        # v2 -> v3: 重命名字段
        state["renamed_field"] = state.pop("old_field")
        state["schema_version"] = 3

    return state
```

**策略2:可选字段**

```python
from typing import NotRequired

class State(TypedDict):
    required_field: str
    optional_field: NotRequired[str]  # 新版本添加的字段

# 兼容旧状态
def node(state: State) -> dict:
    optional_value = state.get("optional_field", "default")
    # ...
```

**策略3:向后兼容的 Reducer**

```python
def backward_compatible_add(old: list | None, new: list) -> list:
    """兼容 None 值的 add reducer"""
    if old is None:
        return new
    return old + new

class State(TypedDict):
    items: Annotated[list | None, backward_compatible_add]
```

**小结**

良好的状态管理:
- ✅ 使用 TypedDict 明确定义结构
- ✅ 选择合适的 reducer 策略
- ✅ 保持扁平化,避免深层嵌套
- ✅ 只存储必要信息,避免冗余
- ✅ 考虑版本演进和兼容性

状态是 Agent 的记忆,设计得当能让 Agent 更智能、更可靠。

---

### 第 6 章:条件路由 - 让 Agent 做决策

条件路由(Conditional Routing)是 LangGraph 实现动态控制流的核心机制。通过条件边,Agent 可以根据状态动态选择下一步行动,实现真正的智能决策。

#### 6.1 概念讲解

##### 6.1.1 条件边原理

**普通边 vs 条件边**

```python
# 普通边:固定路径
graph.add_edge("node_a", "node_b")
# node_a 总是执行 node_b

# 条件边:动态路径
graph.add_conditional_edges(
    "node_a",           # 源节点
    routing_function,   # 路由函数
    {                   # 路由映射
        "path1": "node_b",
        "path2": "node_c",
        "path3": "node_d"
    }
)
# node_a 根据routing_function的返回值选择下一个节点
```

**工作机制**

```
1. node_a 执行完成 → 产生新状态
2. LangGraph 调用 routing_function(新状态)
3. routing_function 返回字符串(如 "path1")
4. LangGraph 查找映射表 {"path1": "node_b"}
5. 执行 node_b
```

**可视化**

```
node_a → [routing_function决策] →
    ├─ "path1" → node_b
    ├─ "path2" → node_c
    └─ "path3" → node_d
```

##### 6.1.2 路由函数详解

路由函数的标准签名:

```python
def routing_function(state: StateType) -> str:
    """
    输入: 当前完整状态
    输出: 下一个节点的名称(字符串)
    """
    # 基于状态做决策
    if condition1(state):
        return "node_name_1"
    elif condition2(state):
        return "node_name_2"
    else:
        return "node_name_3"
```

**路由函数的三种模式**

**模式1:基于状态字段**

```python
def route_by_status(state: State) -> str:
    """根据状态字段路由"""
    status = state["task_status"]

    if status == "simple":
        return "auto_handler"
    elif status == "complex":
        return "manual_review"
    else:
        return "error_handler"
```

**模式2:基于计算逻辑**

```python
def route_by_score(state: State) -> str:
    """根据计算结果路由"""
    score = calculate_confidence(state["data"])

    if score > 0.9:
        return "high_confidence"
    elif score > 0.6:
        return "medium_confidence"
    else:
        return "low_confidence"
```

**模式3:基于 LLM 决策**

```python
def route_by_llm(state: State) -> str:
    """让 LLM 决定路由"""
    prompt = f"""
    根据以下信息,决定下一步操作:
    {state["context"]}

    回复以下之一:
    - "need_more_info"
    - "ready_to_execute"
    - "escalate_to_human"
    """

    decision = llm.invoke(prompt)
    return decision.strip()
```

##### 6.1.3 END 节点和图终止

**END 的作用**

```python
from langgraph.graph import END

# END 是特殊的终止节点
graph.add_edge("final_node", END)
# 执行到此,图结束,返回最终状态
```

**使用 END 的三种方式**

**方式1:普通边指向 END**

```python
graph.add_edge("conclude", END)
# conclude 执行后,图终止
```

**方式2:条件边指向 END**

```python
def should_continue(state: State) -> str:
    if state["is_done"]:
        return "END"
    else:
        return "continue_processing"

graph.add_conditional_edges(
    "check_completion",
    should_continue,
    {
        "END": END,
        "continue_processing": "next_step"
    }
)
```

**方式3:路由函数直接返回 END**

```python
from langgraph.graph import END

def smart_router(state: State) -> str | END:
    if all_done(state):
        return END  # 直接返回 END 对象
    else:
        return "next_node"
```

#### 6.2 实战片段:任务分类路由器

让我们实现一个完整的任务分类系统:

```python
from typing import Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

# 状态定义
class TaskState(TypedDict):
    task_description: str
    task_complexity: Literal["simple", "medium", "complex"] | None
    estimated_hours: int | None
    requires_approval: bool
    result: str | None

# 节点1:分析任务复杂度
def analyze_complexity(state: TaskState) -> dict:
    """分析任务复杂度"""
    description = state["task_description"]

    # 简化的复杂度分析(实际会用 LLM)
    word_count = len(description.split())

    if word_count < 10:
        complexity = "simple"
        hours = 2
        approval = False
    elif word_count < 30:
        complexity = "medium"
        hours = 8
        approval = False
    else:
        complexity = "complex"
        hours = 40
        approval = True

    return {
        "task_complexity": complexity,
        "estimated_hours": hours,
        "requires_approval": approval
    }

# 路由函数:根据复杂度路由
def route_by_complexity(state: TaskState) -> str:
    """根据任务复杂度决定下一步"""
    complexity = state["task_complexity"]

    if complexity == "simple":
        return "auto_execute"
    elif complexity == "medium":
        return "assign_to_team"
    else:  # complex
        if state["requires_approval"]:
            return "request_approval"
        else:
            return "decompose_task"

# 节点2a:自动执行简单任务
def auto_execute(state: TaskState) -> dict:
    """直接执行简单任务"""
    result = f"自动执行: {state['task_description']}"
    return {"result": result}

# 节点2b:分配中等任务
def assign_to_team(state: TaskState) -> dict:
    """分配给团队"""
    result = f"已分配给团队,预计{state['estimated_hours']}小时"
    return {"result": result}

# 节点2c:请求审批
def request_approval(state: TaskState) -> dict:
    """请求审批"""
    result = f"等待审批:复杂任务,预计{state['estimated_hours']}小时"
    return {"result": result}

# 节点2d:分解任务
def decompose_task(state: TaskState) -> dict:
    """分解复杂任务"""
    result = f"任务已分解为子任务"
    return {"result": result}

# 构建图
builder = StateGraph(TaskState)

# 添加节点
builder.add_node("analyze", analyze_complexity)
builder.add_node("auto_execute", auto_execute)
builder.add_node("assign_to_team", assign_to_team)
builder.add_node("request_approval", request_approval)
builder.add_node("decompose_task", decompose_task)

# 添加边
builder.add_edge(START, "analyze")

# 添加条件边:核心路由逻辑
builder.add_conditional_edges(
    "analyze",              # 源节点
    route_by_complexity,    # 路由函数
    {                       # 路由映射
        "auto_execute": "auto_execute",
        "assign_to_team": "assign_to_team",
        "request_approval": "request_approval",
        "decompose_task": "decompose_task"
    }
)

# 所有处理节点都指向 END
builder.add_edge("auto_execute", END)
builder.add_edge("assign_to_team", END)
builder.add_edge("request_approval", END)
builder.add_edge("decompose_task", END)

# 编译
graph = builder.compile()

# 测试不同复杂度的任务
tasks = [
    "修复bug",                                  # 简单
    "优化数据库查询性能",                       # 中等
    "重构整个用户认证系统,支持多种登录方式"    # 复杂
]

for task in tasks:
    result = graph.invoke({
        "task_description": task,
        "task_complexity": None,
        "estimated_hours": None,
        "requires_approval": False,
        "result": None
    })
    print(f"\n任务: {task}")
    print(f"复杂度: {result['task_complexity']}")
    print(f"处理: {result['result']}")
```

**执行流程示例**

```
输入: "修复bug"
↓
analyze (分析) → complexity="simple", hours=2
↓
route_by_complexity (路由) → 返回 "auto_execute"
↓
auto_execute (执行) → result="自动执行: 修复bug"
↓
END

最终状态:
{
  "task_description": "修复bug",
  "task_complexity": "simple",
  "estimated_hours": 2,
  "requires_approval": false,
  "result": "自动执行: 修复bug"
}
```

#### 6.3 最佳实践

##### 6.3.1 路由逻辑应该简单明确

**原则**: 路由函数应该易于理解,决策清晰

**反例:复杂的嵌套逻辑**

```python
# ❌ 不好:逻辑复杂,难以维护
def complex_router(state: State) -> str:
    if state["type"] == "A":
        if state["score"] > 0.8:
            if state["user_level"] == "premium":
                return "path1"
            else:
                return "path2" if state["count"] > 5 else "path3"
        else:
            return "path4"
    elif state["type"] == "B":
        # ... 更多嵌套
```

**正例:清晰的决策树**

```python
# ✅ 好:逻辑清晰,每个分支明确
def clear_router(state: State) -> str:
    # 先检查类型
    if state["type"] == "urgent":
        return "urgent_handler"

    # 再检查分数
    if state["score"] > 0.8:
        return "high_score_path"

    # 最后的默认路径
    return "standard_path"
```

**更好:提取决策逻辑**

```python
# ✅ 最好:决策逻辑独立,易于测试
def calculate_priority(state: State) -> str:
    """计算优先级(纯函数,易测试)"""
    if state["is_urgent"]:
        return "high"
    elif state["score"] > 0.8:
        return "medium"
    else:
        return "low"

def route_by_priority(state: State) -> str:
    """路由函数简单调用决策逻辑"""
    priority = calculate_priority(state)

    priority_map = {
        "high": "urgent_handler",
        "medium": "normal_handler",
        "low": "batch_handler"
    }

    return priority_map[priority]
```

##### 6.3.2 处理所有可能的分支

**原则**: 确保路由函数覆盖所有可能的状态,避免死路径

**反例:遗漏分支**

```python
# ❌ 危险:如果 status 不是这三个值之一,会出错
def incomplete_router(state: State) -> str:
    if state["status"] == "ready":
        return "execute"
    elif state["status"] == "pending":
        return "wait"
    elif state["status"] == "failed":
        return "retry"
    # 缺少 else 分支!
```

**正例:总是有默认分支**

```python
# ✅ 安全:总是有 fallback
def complete_router(state: State) -> str:
    status = state["status"]

    if status == "ready":
        return "execute"
    elif status == "pending":
        return "wait"
    elif status == "failed":
        return "retry"
    else:
        # 默认分支:记录未知状态并终止
        log_warning(f"Unknown status: {status}")
        return END
```

**更好:使用枚举和完全匹配**

```python
from typing import Literal

# ✅ 最好:类型系统保证完整性
def type_safe_router(state: State) -> str:
    status: Literal["ready", "pending", "failed", "unknown"] = state["status"]

    match status:
        case "ready":
            return "execute"
        case "pending":
            return "wait"
        case "failed":
            return "retry"
        case "unknown":
            return END
```

**测试路由完整性**

```python
# 单元测试:确保所有状态都有路由
def test_router_completeness():
    test_cases = [
        ({"status": "ready"}, "execute"),
        ({"status": "pending"}, "wait"),
        ({"status": "failed"}, "retry"),
        ({"status": "invalid"}, END),  # 测试边界情况
    ]

    for state, expected in test_cases:
        result = complete_router(state)
        assert result == expected
```

##### 6.3.3 使用枚举避免字符串硬编码

**原则**: 使用常量或枚举定义节点名称,避免拼写错误

**反例:字符串硬编码**

```python
# ❌ 危险:容易拼写错误
def bad_router(state: State) -> str:
    if state["score"] > 0.8:
        return "high_priority_handler"  # 这里拼写正确吗?

builder.add_node("high_pririty_handler", handler)  # 拼写错误!
# 运行时才会发现错误
```

**正例:使用常量**

```python
# ✅ 好:使用常量
class NodeNames:
    HIGH_PRIORITY = "high_priority_handler"
    MEDIUM_PRIORITY = "medium_priority_handler"
    LOW_PRIORITY = "low_priority_handler"

def good_router(state: State) -> str:
    if state["score"] > 0.8:
        return NodeNames.HIGH_PRIORITY  # IDE会自动补全

builder.add_node(NodeNames.HIGH_PRIORITY, handler)  # 一致性保证
```

**更好:使用枚举**

```python
# ✅ 最好:使用枚举
from enum import Enum

class NodeName(str, Enum):
    HIGH_PRIORITY = "high_priority_handler"
    MEDIUM_PRIORITY = "medium_priority_handler"
    LOW_PRIORITY = "low_priority_handler"

def best_router(state: State) -> NodeName:
    """类型安全的路由函数"""
    if state["score"] > 0.8:
        return NodeName.HIGH_PRIORITY  # 类型安全
    elif state["score"] > 0.5:
        return NodeName.MEDIUM_PRIORITY
    else:
        return NodeName.LOW_PRIORITY

# 构建图时也使用枚举
builder.add_node(NodeName.HIGH_PRIORITY, high_priority_handler)
builder.add_conditional_edges(
    "analyze",
    best_router,
    {
        NodeName.HIGH_PRIORITY: NodeName.HIGH_PRIORITY.value,
        NodeName.MEDIUM_PRIORITY: NodeName.MEDIUM_PRIORITY.value,
        NodeName.LOW_PRIORITY: NodeName.LOW_PRIORITY.value,
    }
)
```

**小结**

条件路由的最佳实践:
- ✅ 保持路由逻辑简单明确
- ✅ 覆盖所有可能的分支,总是有 else
- ✅ 使用枚举或常量,避免字符串硬编码
- ✅ 独立测试路由函数
- ✅ 考虑边界情况和异常状态

条件路由让 Agent 能够智能决策,是实现复杂工作流的关键。

---

### 第 7 章:并行执行 - 提升效率的关键

并行执行允许多个节点同时运行,是提升 Agent 性能的重要手段。LangGraph 原生支持并行执行,让你轻松实现 fan-out/fan-in 模式。

#### 7.1 概念讲解

##### 7.1.1 并行节点执行机制

**什么是并行执行**

```python
# 串行执行:一个接一个
node_a → node_b → node_c  # 总时间 = 时间A + 时间B + 时间C

# 并行执行:同时进行
node_a → [node_b, node_c, node_d] → node_e
         ↑ 并行执行 ↑
# 总时间 = 时间A + max(时间B, 时间C, 时间D) + 时间E
```

**LangGraph 的并行执行模型**

```
Fan-Out(扇出) → Parallel Execution(并行执行) → Fan-In(扇入)

         node_a
           ↓
    ┌──────┼──────┐
    ↓      ↓      ↓
  node_b node_c node_d  # 并行执行
    └──────┼──────┘
           ↓
        node_e           # 聚合结果
```

**如何触发并行执行**

方法1:多条边从同一个节点出发

```python
builder.add_edge(START, "node_b")
builder.add_edge(START, "node_c")
builder.add_edge(START, "node_d")
# START 后,node_b、node_c、node_d 并行执行
```

方法2:使用 Send API 动态分发(后面详述)

##### 7.1.2 Send API 详解

**Send API 的作用**

Send 允许在运行时动态创建并行任务,适用于:
- 任务数量事先不确定
- 需要对列表中的每个元素执行相同操作
- Map-Reduce 模式

**基本语法**

```python
from langgraph.types import Send

def fan_out_node(state: State) -> list[Send]:
    """动态创建并行任务"""
    tasks = []
    for item in state["items"]:
        # 为每个 item 创建一个任务
        # Send(目标节点名, 传递给该节点的状态)
        tasks.append(Send("process_item", {"item": item}))
    return tasks

# 在图中使用
builder.add_conditional_edges(
    "fan_out_node",
    fan_out_node,  # 返回 Send 列表
)
```

**Send 的工作流程**

```
1. fan_out_node 执行,返回 [Send("process", {"item": 1}),
                          Send("process", {"item": 2}),
                          Send("process", {"item": 3})]

2. LangGraph 并行执行:
   process({"item": 1})  ┐
   process({"item": 2})  ├─ 并行
   process({"item": 3})  ┘

3. 所有并行任务完成后,继续后续节点
```

**完整示例:Map-Reduce 模式**

```python
from typing import Annotated
from operator import add
from langgraph.types import Send

class State(TypedDict):
    items: list[int]
    results: Annotated[list[int], add]  # 累加结果

def map_items(state: State) -> list[Send]:
    """Map:为每个 item 创建并行任务"""
    return [
        Send("process_item", {"item": item})
        for item in state["items"]
    ]

def process_item(state: dict) -> dict:
    """处理单个 item"""
    item = state["item"]
    result = item * 2  # 简单的处理逻辑
    return {"results": [result]}

def reduce_results(state: State) -> dict:
    """Reduce:聚合所有结果"""
    total = sum(state["results"])
    return {"final_result": total}

# 构建图
builder = StateGraph(State)
builder.add_node("map", map_items)
builder.add_node("process_item", process_item)  # 会被并行调用多次
builder.add_node("reduce", reduce_results)

builder.add_edge(START, "map")
builder.add_conditional_edges("map", map_items)  # 动态并行
builder.add_edge("process_item", "reduce")  # 所有并行任务完成后执行
builder.add_edge("reduce", END)
```

##### 7.1.3 并行结果聚合策略

**策略1:使用 Reducer 自动聚合**

```python
class State(TypedDict):
    # 使用 add reducer,自动合并结果
    results: Annotated[list, add]

def parallel_node_1(state: State) -> dict:
    return {"results": ["result1"]}

def parallel_node_2(state: State) -> dict:
    return {"results": ["result2"]}

# 执行后,state["results"] = ["result1", "result2"]
```

**策略2:专门的聚合节点**

```python
class State(TypedDict):
    result_a: str
    result_b: str
    result_c: str
    combined: str

def aggregate(state: State) -> dict:
    """显式聚合"""
    combined = f"{state['result_a']} | {state['result_b']} | {state['result_c']}"
    return {"combined": combined}

# 图结构
#   [node_a, node_b, node_c] → aggregate
```

**策略3:条件聚合**

```python
def smart_aggregate(state: State) -> dict:
    """根据结果质量选择性聚合"""
    valid_results = [
        r for r in state["results"]
        if r["quality_score"] > 0.7
    ]

    if len(valid_results) >= 2:
        return {"final": combine(valid_results)}
    else:
        return {"error": "Not enough valid results"}
```

#### 7.2 实战片段:多任务并行处理

让我们构建一个并行的市场研究助手,同时执行多个研究任务:

```python
from typing import Annotated, TypedDict
from operator import add
from langgraph.types import Send
from langgraph.graph import StateGraph, START, END

# 状态定义
class ResearchState(TypedDict):
    topic: str                              # 研究主题
    research_tasks: list[str]               # 要执行的研究任务
    task_results: Annotated[list[dict], add]  # 累加所有任务结果
    final_report: str                       # 最终报告

# 任务单元状态(用于并行执行)
class TaskState(TypedDict):
    task_name: str
    topic: str

# 节点1:分解研究任务
def decompose_research(state: ResearchState) -> dict:
    """将研究分解为多个并行任务"""
    topic = state["topic"]

    tasks = [
        "competitor_analysis",    # 竞品分析
        "market_trends",          # 市场趋势
        "user_feedback",          # 用户反馈
        "technical_feasibility"   # 技术可行性
    ]

    return {"research_tasks": tasks}

# 节点2:动态创建并行任务
def fan_out_tasks(state: ResearchState) -> list[Send]:
    """为每个研究任务创建并行执行单元"""
    return [
        Send("execute_task", {
            "task_name": task,
            "topic": state["topic"]
        })
        for task in state["research_tasks"]
    ]

# 节点3:执行单个任务(会被并行调用)
def execute_task(state: TaskState) -> dict:
    """执行单个研究任务"""
    task_name = state["task_name"]
    topic = state["topic"]

    # 模拟不同的研究任务
    if task_name == "competitor_analysis":
        result = {
            "task": task_name,
            "findings": f"分析了 {topic} 领域的主要竞争对手",
            "data": ["竞品A", "竞品B", "竞品C"]
        }
    elif task_name == "market_trends":
        result = {
            "task": task_name,
            "findings": f"{topic} 市场呈上升趋势",
            "data": ["趋势1", "趋势2"]
        }
    elif task_name == "user_feedback":
        result = {
            "task": task_name,
            "findings": f"用户对 {topic} 的反馈积极",
            "data": ["反馈1", "反馈2"]
        }
    else:  # technical_feasibility
        result = {
            "task": task_name,
            "findings": f"{topic} 技术上可行",
            "data": ["技术点1", "技术点2"]
        }

    print(f"✓ 完成任务: {task_name}")
    return {"task_results": [result]}

# 节点4:聚合所有结果
def aggregate_results(state: ResearchState) -> dict:
    """将所有并行任务的结果聚合成报告"""
    results = state["task_results"]

    report = f"# {state['topic']} 研究报告\n\n"

    for result in results:
        report += f"## {result['task']}\n"
        report += f"{result['findings']}\n"
        report += f"数据: {', '.join(result['data'])}\n\n"

    return {"final_report": report}

# 构建图
builder = StateGraph(ResearchState)

# 添加节点
builder.add_node("decompose", decompose_research)
builder.add_node("execute_task", execute_task)  # 会被并行调用
builder.add_node("aggregate", aggregate_results)

# 添加边
builder.add_edge(START, "decompose")

# 使用 Send API 实现动态并行
builder.add_conditional_edges("decompose", fan_out_tasks)

# 所有并行任务完成后,执行聚合
builder.add_edge("execute_task", "aggregate")
builder.add_edge("aggregate", END)

# 编译
graph = builder.compile()

# 执行
result = graph.invoke({
    "topic": "AI 代码助手",
    "research_tasks": [],
    "task_results": [],
    "final_report": ""
})

print("\n" + "="*50)
print(result["final_report"])
```

**执行流程**

```
START
  ↓
decompose (分解任务) → research_tasks = [A, B, C, D]
  ↓
fan_out_tasks (创建并行任务)
  ↓
┌─────────────┬─────────────┬─────────────┬─────────────┐
│ execute(A)  │ execute(B)  │ execute(C)  │ execute(D)  │ 并行执行
└─────────────┴─────────────┴─────────────┴─────────────┘
  │               │               │               │
  └───────────────┴───────────────┴───────────────┘
                  ↓
            aggregate (聚合结果)
                  ↓
                 END

最终状态:
{
  "topic": "AI 代码助手",
  "research_tasks": [...],
  "task_results": [
    {"task": "competitor_analysis", ...},
    {"task": "market_trends", ...},
    {"task": "user_feedback", ...},
    {"task": "technical_feasibility", ...}
  ],
  "final_report": "# AI 代码助手 研究报告\n\n..."
}
```

#### 7.3 最佳实践

##### 7.3.1 识别可并行任务的方法

**原则**:并行任务之间应该是独立的,互不依赖

**判断标准**

```
✅ 可以并行:
- 任务A的输出不影响任务B的输入
- 任务之间没有数据依赖
- 任务之间没有顺序要求

❌ 不能并行:
- 任务B需要任务A的结果
- 任务之间有共享的可变状态
- 任务之间有顺序依赖
```

**示例:区分可并行和不可并行**

```python
# ❌ 不能并行:有依赖关系
def task_flow_with_dependency():
    # 必须先生成代码,再测试
    code = generate_code(spec)      # 任务1
    test_result = run_tests(code)   # 任务2(依赖任务1)
    # 这两个任务必须串行

# ✅ 可以并行:独立任务
def task_flow_independent():
    # 这些可以同时进行
    competitor_data = analyze_competitors(topic)   # 任务1
    market_data = research_market(topic)           # 任务2
    user_data = collect_feedback(topic)            # 任务3
    # 这三个任务可以并行
```

**识别并行机会的技巧**

```python
# 技巧1:多个数据源
# ✅ 从不同来源获取数据可以并行
tasks = [
    fetch_from_database(query),
    fetch_from_api(endpoint),
    fetch_from_cache(key)
]

# 技巧2:相同操作应用于不同数据
# ✅ Map 操作天然并行
results = [process(item) for item in items]

# 技巧3:多个独立的 LLM 调用
# ✅ 不同的 LLM 任务可以并行
joke = llm.invoke("Write a joke about cats")
poem = llm.invoke("Write a poem about cats")
story = llm.invoke("Write a story about cats")
```

##### 7.3.2 并发控制策略

**原则**:控制并发数量,避免资源耗尽

**问题:无限制并发**

```python
# ❌ 危险:1000个任务同时执行
items = range(1000)
tasks = [Send("process", {"item": i}) for i in items]
# 可能导致:
# - 内存耗尽
# - API 限流
# - 系统过载
```

**解决方案1:分批处理**

```python
def fan_out_with_batching(state: State) -> list[Send]:
    """分批处理,每批最多10个"""
    items = state["items"]
    batch_size = 10

    # 只处理前batch_size个
    current_batch = items[:batch_size]
    remaining = items[batch_size:]

    tasks = [
        Send("process_item", {"item": item})
        for item in current_batch
    ]

    # 更新状态,记录剩余任务
    return tasks + [{"remaining_items": remaining}]
```

**解决方案2:使用信号量(概念)**

```python
# 伪代码:限制并发数
max_concurrent = 5
semaphore = Semaphore(max_concurrent)

def process_with_limit(item):
    with semaphore:  # 最多5个同时执行
        return process(item)
```

**解决方案3:优先级队列**

```python
def fan_out_with_priority(state: State) -> list[Send]:
    """按优先级分配资源"""
    tasks = state["tasks"]

    # 先处理高优先级任务
    high_priority = [t for t in tasks if t["priority"] == "high"]
    low_priority = [t for t in tasks if t["priority"] == "low"]

    # 高优先级任务并行,低优先级任务排队
    sends = [Send("process", {"task": t}) for t in high_priority[:5]]
    sends += [{"queued_tasks": low_priority}]

    return sends
```

##### 7.3.3 部分失败处理

**原则**:并行任务中,部分失败不应导致全部失败

**策略1:收集成功和失败**

```python
class State(TypedDict):
    successes: Annotated[list, add]
    failures: Annotated[list, add]

def resilient_task(state: TaskState) -> dict:
    """容错的任务执行"""
    try:
        result = risky_operation(state["data"])
        return {"successes": [result]}
    except Exception as e:
        return {"failures": [{
            "task": state["task_name"],
            "error": str(e)
        }]}

def aggregate_with_partial_failure(state: State) -> dict:
    """聚合时处理部分失败"""
    total = len(state["successes"]) + len(state["failures"])
    success_rate = len(state["successes"]) / total

    if success_rate >= 0.7:  # 70%成功就继续
        return {"status": "partial_success", "data": state["successes"]}
    else:
        return {"status": "failed", "errors": state["failures"]}
```

**策略2:重试机制**

```python
def task_with_retry(state: TaskState) -> dict:
    """失败自动重试"""
    max_retries = 3
    attempt = state.get("attempt", 0)

    try:
        result = execute(state["data"])
        return {"results": [result]}
    except Exception as e:
        if attempt < max_retries:
            # 重新发送任务,增加重试计数
            return Send("task_with_retry", {
                **state,
                "attempt": attempt + 1
            })
        else:
            # 达到最大重试次数,记录失败
            return {"failures": [{"error": str(e)}]}
```

**策略3:降级处理**

```python
def task_with_fallback(state: TaskState) -> dict:
    """主任务失败时使用备选方案"""
    try:
        # 尝试首选方法
        result = preferred_method(state["data"])
        return {"results": [result]}
    except Exception:
        try:
            # 降级到备选方法
            result = fallback_method(state["data"])
            return {"results": [result], "used_fallback": True}
        except Exception as e:
            # 两种方法都失败
            return {"failures": [{"error": str(e)}]}
```

**小结**

并行执行的最佳实践:
- ✅ 只并行独立的任务
- ✅ 使用 Send API 实现动态并行
- ✅ 控制并发数量,避免资源耗尽
- ✅ 处理部分失败,保证健壮性
- ✅ 使用 reducer 自动聚合结果

并行执行是提升 Agent 性能的关键,合理使用能显著减少总执行时间。

---

### 第 8 章:人机协作 - Interrupt 与 Approval 模式

人机协作(Human-in-the-Loop)是 AI Agent 的重要能力。LangGraph 通过 `interrupt` 机制,让 Agent 能够在关键点暂停,等待人类输入后继续执行。

#### 8.1 概念讲解

##### 8.1.1 Interrupt 机制详解

**什么是 Interrupt**

Interrupt 允许图在执行过程中**暂停**,等待外部输入(通常是人类),然后**恢复**执行。

```
正常执行流程:
node_a → node_b → node_c → END

带 Interrupt 的流程:
node_a → node_b → [暂停,等待输入] → node_c → END
                  ↑
                  人类提供输入
```

**Interrupt 的两种方式**

**方式1:编译时配置(静态)**

```python
graph = graph_builder.compile(
    checkpointer=checkpointer,           # 必须有 checkpointer
    interrupt_before=["risky_operation"], # 在该节点执行前中断
    # 或
    interrupt_after=["data_collection"]   # 在该节点执行后中断
)
```

**方式2:运行时调用(动态)**

```python
from langgraph.types import interrupt

def approval_node(state: State) -> dict:
    """节点内部调用 interrupt"""
    # 暂停并等待用户输入
    user_decision = interrupt({
        "action": state["pending_action"],
        "question": "是否批准此操作?"
    })

    # user_decision 是用户提供的值
    return {"approved": user_decision}
```

**工作机制**

```
1. 图执行到 interrupt 点
2. LangGraph 保存当前状态到 checkpointer
3. 图暂停,返回中断信息
4. 用户审查并提供输入
5. 用户调用 graph.invoke(Command(resume=user_input), config)
6. 图从中断点恢复执行,使用用户输入
```

##### 8.1.2 Approval 工作流模式

**基本审批模式**

```python
def prepare_action(state: State) -> dict:
    """准备要执行的操作"""
    action = generate_action_plan(state)
    return {"pending_action": action}

def request_approval(state: State) -> dict:
    """请求人工审批"""
    approved = interrupt({
        "action": state["pending_action"],
        "context": state["current_context"],
        "prompt": "请审批此操作"
    })

    return {"approved": approved}

def execute_or_cancel(state: State) -> dict:
    """根据审批结果执行或取消"""
    if state["approved"]:
        result = execute(state["pending_action"])
        return {"result": result, "status": "executed"}
    else:
        return {"result": None, "status": "cancelled"}
```

**多级审批模式**

```python
def route_by_risk(state: State) -> str:
    """根据风险等级路由"""
    risk = assess_risk(state["action"])

    if risk == "low":
        return "auto_execute"  # 低风险,自动执行
    elif risk == "medium":
        return "manager_approval"  # 中风险,经理审批
    else:
        return "director_approval"  # 高风险,总监审批

# 图结构
#   prepare → assess_risk → [条件路由]
#     ├─ low → auto_execute
#     ├─ medium → manager_approval → execute
#     └─ high → director_approval → execute
```

##### 8.1.3 恢复执行的状态管理

**Checkpointer 的作用**

```python
from langgraph.checkpoint.memory import InMemorySaver

# Interrupt 必须配合 checkpointer 使用
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# 每次执行需要提供 thread_id
config = {"configurable": {"thread_id": "user_123"}}

# 第一次执行:运行到 interrupt
result1 = graph.invoke(initial_state, config=config)
# result1["__interrupt__"] 包含中断信息

# 恢复执行:提供用户输入
from langgraph.types import Command
result2 = graph.invoke(Command(resume=user_input), config=config)
# 从中断点继续,使用 user_input
```

**状态完整性**

Interrupt 前后,状态完全保留:

```python
# Interrupt 前的状态
{
  "messages": ["用户消息1", "AI回复1"],
  "current_task": "数据分析",
  "progress": 0.5
}

# 暂停...

# Interrupt 后恢复,状态完整保留
# 加上用户提供的新输入
{
  "messages": ["用户消息1", "AI回复1"],
  "current_task": "数据分析",
  "progress": 0.5,
  "user_feedback": "继续"  # 新添加
}
```

#### 8.2 实战片段:高风险操作审批流程

让我们构建一个完整的审批工作流:

```python
from typing import Literal, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

# 状态定义
class ApprovalState(TypedDict):
    action_type: str                 # 操作类型
    action_details: dict              # 操作详情
    risk_level: Literal["low", "medium", "high"] | None
    requires_approval: bool
    approval_status: Literal["pending", "approved", "rejected"] | None
    execution_result: str | None

# 节点1:准备操作
def prepare_action(state: ApprovalState) -> dict:
    """分析并准备操作"""
    action_type = state["action_type"]
    details = state["action_details"]

    print(f"准备操作: {action_type}")
    print(f"详情: {details}")

    return {}  # 状态已包含操作信息

# 节点2:评估风险
def assess_risk(state: ApprovalState) -> dict:
    """评估操作风险"""
    action_type = state["action_type"]

    # 简化的风险评估
    risk_levels = {
        "read_data": "low",
        "update_data": "medium",
        "delete_data": "high",
        "system_config": "high"
    }

    risk = risk_levels.get(action_type, "medium")
    requires_approval = (risk in ["medium", "high"])

    print(f"风险等级: {risk}")
    print(f"需要审批: {requires_approval}")

    return {
        "risk_level": risk,
        "requires_approval": requires_approval
    }

# 路由函数:决定是否需要审批
def route_by_approval(state: ApprovalState) -> str:
    """根据风险决定路由"""
    if not state["requires_approval"]:
        return "auto_execute"
    else:
        return "request_approval"

# 节点3a:自动执行(低风险)
def auto_execute(state: ApprovalState) -> dict:
    """自动执行低风险操作"""
    result = f"已自动执行: {state['action_type']}"
    print(result)

    return {
        "approval_status": "approved",
        "execution_result": result
    }

# 节点3b:请求审批(高/中风险)
def request_approval(state: ApprovalState) -> dict:
    """暂停并请求人工审批"""
    print("\n" + "="*50)
    print("⚠️  需要人工审批")
    print("="*50)

    # 构造审批请求
    approval_request = {
        "action": state["action_type"],
        "details": state["action_details"],
        "risk": state["risk_level"],
        "question": "是否批准此操作?(输入 'yes' 或 'no')"
    }

    # 调用 interrupt,暂停执行
    user_decision = interrupt(approval_request)

    # 图会在这里暂停,等待用户输入
    # 用户调用 resume 后,user_decision 会是用户提供的值

    # 解析用户决策
    approved = (user_decision == "yes")

    return {
        "approval_status": "approved" if approved else "rejected"
    }

# 节点4:执行或取消
def execute_or_cancel(state: ApprovalState) -> dict:
    """根据审批结果执行或取消"""
    if state["approval_status"] == "approved":
        result = f"✓ 已执行: {state['action_type']}"
        print(result)
        return {"execution_result": result}
    else:
        result = f"✗ 已取消: {state['action_type']}"
        print(result)
        return {"execution_result": result}

# 构建图
builder = StateGraph(ApprovalState)

# 添加节点
builder.add_node("prepare", prepare_action)
builder.add_node("assess", assess_risk)
builder.add_node("auto_execute", auto_execute)
builder.add_node("request_approval", request_approval)
builder.add_node("execute_or_cancel", execute_or_cancel)

# 添加边
builder.add_edge(START, "prepare")
builder.add_edge("prepare", "assess")

# 条件路由:根据风险决定是否需要审批
builder.add_conditional_edges(
    "assess",
    route_by_approval,
    {
        "auto_execute": "auto_execute",
        "request_approval": "request_approval"
    }
)

# 自动执行直接结束
builder.add_edge("auto_execute", END)

# 审批后执行或取消
builder.add_edge("request_approval", "execute_or_cancel")
builder.add_edge("execute_or_cancel", END)

# 编译(必须使用 checkpointer)
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# ========== 测试场景1:低风险操作(自动执行) ==========
print("\n【场景1:低风险操作 - 读取数据】")
config1 = {"configurable": {"thread_id": "request_001"}}

result1 = graph.invoke({
    "action_type": "read_data",
    "action_details": {"table": "users", "limit": 10},
    "risk_level": None,
    "requires_approval": False,
    "approval_status": None,
    "execution_result": None
}, config=config1)

print(f"最终结果: {result1['execution_result']}\n")

# ========== 测试场景2:高风险操作(需要审批) ==========
print("\n【场景2:高风险操作 - 删除数据】")
config2 = {"configurable": {"thread_id": "request_002"}}

# 第一步:执行到 interrupt
result2_step1 = graph.invoke({
    "action_type": "delete_data",
    "action_details": {"table": "orders", "condition": "status='cancelled'"},
    "risk_level": None,
    "requires_approval": False,
    "approval_status": None,
    "execution_result": None
}, config=config2)

# 检查是否中断
if "__interrupt__" in result2_step1:
    print("\n图已暂停,等待审批...")
    print(f"中断信息: {result2_step1['__interrupt__']}")

    # 第二步:用户审批(模拟用户批准)
    print("\n用户决策: 批准")
    result2_step2 = graph.invoke(
        Command(resume="yes"),  # 用户批准
        config=config2
    )

    print(f"最终结果: {result2_step2['execution_result']}")

# ========== 测试场景3:拒绝审批 ==========
print("\n【场景3:高风险操作 - 拒绝审批】")
config3 = {"configurable": {"thread_id": "request_003"}}

result3_step1 = graph.invoke({
    "action_type": "system_config",
    "action_details": {"setting": "max_connections", "value": 1000},
    "risk_level": None,
    "requires_approval": False,
    "approval_status": None,
    "execution_result": None
}, config=config3)

if "__interrupt__" in result3_step1:
    print("\n用户决策: 拒绝")
    result3_step2 = graph.invoke(
        Command(resume="no"),  # 用户拒绝
        config=config3
    )

    print(f"最终结果: {result3_step2['execution_result']}")
```

**执行流程示例**

```
场景2:删除数据(高风险)

START
  ↓
prepare (准备操作) → action_type="delete_data"
  ↓
assess (评估风险) → risk="high", requires_approval=true
  ↓
route_by_approval → 返回 "request_approval"
  ↓
request_approval → interrupt(审批请求)
  ↓
[图暂停,保存状态]
  ↓
[用户审查,决定批准/拒绝]
  ↓
graph.invoke(Command(resume="yes")) → 恢复执行
  ↓
execute_or_cancel → approval_status="approved" → 执行
  ↓
END
```

#### 8.3 最佳实践

##### 8.3.1 何时需要人类介入

**原则**:在不确定性高、影响大的决策点使用人机协作

**适合人工介入的场景**

| 场景类型 | 示例 | 原因 |
|---------|------|------|
| **高风险操作** | 删除数据、修改配置、发送邮件 | 错误代价高 |
| **低置信度决策** | LLM 置信度 < 0.7 的结果 | 结果不可靠 |
| **创意性工作** | 设计方案、营销文案 | 需要人类审美 |
| **伦理敏感** | 涉及隐私、法律的决策 | 需要人类判断 |
| **异常情况** | 遇到训练数据外的情况 | Agent 无法处理 |
| **学习反馈** | 收集人类偏好用于改进 | 持续优化 |

**不适合人工介入的场景**

```python
# ❌ 不要在这些情况下使用 interrupt
- 简单的数据查询
- 确定性的计算
- 已经过充分测试的操作
- 高频率的重复任务
```

**动态决策示例**

```python
def smart_interrupt(state: State) -> dict:
    """根据置信度决定是否中断"""
    confidence = state["llm_confidence"]

    if confidence < 0.6:
        # 置信度低,请求人工确认
        user_input = interrupt({
            "suggestion": state["llm_output"],
            "confidence": confidence,
            "question": "LLM 不确定,请人工确认"
        })
        return {"final_output": user_input}
    else:
        # 置信度高,直接使用 LLM 输出
        return {"final_output": state["llm_output"]}
```

##### 8.3.2 超时处理机制

**原则**:人工审批不应无限期等待

**策略1:在应用层实现超时**

```python
import time
from datetime import datetime, timedelta

def request_approval_with_timeout(state: State) -> dict:
    """带超时的审批请求"""
    # 记录请求时间
    request_time = datetime.now()

    # 请求审批
    decision = interrupt({
        "action": state["action"],
        "timeout": "2小时",
        "default": "auto_reject"
    })

    # 注意:interrupt 会暂停图
    # 恢复后,检查是否超时(在外部应用逻辑中实现)

    return {"approval": decision}

# 在应用层
def monitor_approval(thread_id, timeout_hours=2):
    """监控审批超时"""
    deadline = datetime.now() + timedelta(hours=timeout_hours)

    while datetime.now() < deadline:
        # 检查是否有用户输入
        if check_user_response(thread_id):
            return "approved"
        time.sleep(60)  # 每分钟检查一次

    # 超时,自动拒绝
    graph.invoke(Command(resume="timeout_reject"), config)
```

**策略2:在状态中记录时间**

```python
class State(TypedDict):
    approval_requested_at: str | None
    approval_deadline: str | None

def check_timeout(state: State) -> str:
    """检查是否超时"""
    if state["approval_deadline"]:
        deadline = datetime.fromisoformat(state["approval_deadline"])
        if datetime.now() > deadline:
            return "timeout_handler"

    return "wait_approval"
```

##### 8.3.3 审批上下文的完整性

**原则**:提供足够的上下文,让人类做出明智决策

**反例:上下文不足**

```python
# ❌ 不好:信息不足
approved = interrupt({
    "question": "是否批准?"  # 批准什么?
})
```

**正例:完整上下文**

```python
# ✅ 好:提供完整信息
approved = interrupt({
    # 1. 操作描述
    "action": {
        "type": "delete_records",
        "table": "users",
        "condition": "last_login < '2020-01-01'",
        "estimated_affected": 15234
    },

    # 2. 风险评估
    "risk": {
        "level": "high",
        "concerns": ["不可逆操作", "影响大量数据"],
        "mitigation": "已备份到 backup_users 表"
    },

    # 3. 上下文信息
    "context": {
        "requester": "admin@example.com",
        "reason": "清理过期数据",
        "timestamp": "2024-03-15 10:30:00"
    },

    # 4. 明确的问题
    "question": "是否批准删除 15234 条用户记录?",

    # 5. 预期的回复格式
    "expected_response": "yes/no/defer"
})
```

**提供撤销选项**

```python
def execute_with_undo(state: State) -> dict:
    """执行操作,但提供撤销机会"""
    # 执行操作
    result = execute(state["action"])

    # 再次中断,提供撤销机会
    confirm = interrupt({
        "result": result,
        "question": "操作已执行,是否确认?输入 'undo' 撤销"
    })

    if confirm == "undo":
        rollback(result)
        return {"status": "reverted"}
    else:
        return {"status": "confirmed", "result": result}
```

**小结**

人机协作的最佳实践:
- ✅ 在高风险、不确定的决策点使用
- ✅ 提供完整的上下文信息
- ✅ 实现超时和默认行为
- ✅ 必须使用 checkpointer 保存状态
- ✅ 考虑撤销和回滚机制

人机协作让 Agent 更可控、更可信,是生产环境的必备能力。

---

### 第 9 章:持久化与恢复 - 长时间运行的 Agent

持久化(Persistence)让 Agent 能够跨会话保存状态,支持长时间运行的任务、系统故障恢复、以及强大的调试能力。

#### 9.1 概念讲解

##### 9.1.1 Checkpointer 抽象层

**什么是 Checkpointer**

Checkpointer 是 LangGraph 的持久化抽象层,负责:
- 在每个节点执行后保存状态
- 提供状态查询和恢复接口
- 支持多线程/多用户的独立会话

**基本接口**

```python
from langgraph.checkpoint.base import BaseCheckpointSaver

class MyCheckpointer(BaseCheckpointSaver):
    def put(self, config, checkpoint, metadata):
        """保存检查点"""
        pass

    def get(self, config):
        """获取最新检查点"""
        pass

    def list(self, config):
        """列出所有检查点"""
        pass
```

**内置 Checkpointer 实现**

| 类型 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **MemorySaver** | 开发、测试、演示 | 简单快速,无依赖 | 进程重启后丢失 |
| **SqliteSaver** | 单机应用、原型 | 持久化,无需外部服务 | 不支持分布式 |
| **PostgresSaver** | 生产环境 | 持久化,支持分布式 | 需要数据库服务 |

**使用 Checkpointer**

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

# 开发环境:内存
checkpointer = InMemorySaver()

# 生产环境:SQLite
conn = sqlite3.connect("checkpoints.db")
checkpointer = SqliteSaver(conn)

# 编译时指定
graph = builder.compile(checkpointer=checkpointer)

# 执行时提供 thread_id
config = {"configurable": {"thread_id": "user_123"}}
result = graph.invoke(input_data, config=config)
```

##### 9.1.2 执行历史记录

**检查点的内容**

每个检查点包含:
1. **完整状态**: 当前所有状态字段的值
2. **元数据**: 节点名、时间戳、父检查点ID
3. **配置**: thread_id 等配置信息

**查看执行历史**

```python
# 获取指定线程的所有检查点
history = list(graph.get_state_history(config))

for checkpoint in history:
    print(f"步骤: {checkpoint.metadata['step']}")
    print(f"节点: {checkpoint.metadata['source']}")
    print(f"状态: {checkpoint.values}")
    print(f"时间: {checkpoint.metadata['ts']}")
    print("---")

# 输出示例:
# 步骤: 3
# 节点: analyze
# 状态: {"messages": [...], "score": 0.8}
# 时间: 2024-03-15T10:30:00
```

**检查点的层次结构**

```
checkpoint_0 (初始状态)
  ↓
checkpoint_1 (执行 node_a 后)
  ↓
checkpoint_2 (执行 node_b 后)
  ↓
checkpoint_3 (执行 node_c 后)
  ↓
checkpoint_4 (END)
```

##### 9.1.3 时间旅行调试

**时间旅行(Time Travel)**允许你:
- 回到任意历史检查点查看状态
- 从过去的状态重新执行
- 修改历史状态并查看影响

**查看历史状态**

```python
# 获取所有检查点
checkpoints = list(graph.get_state_history(config))

# 查看第2步的状态
step_2 = checkpoints[2]
print(f"第2步状态: {step_2.values}")

# 查看特定检查点
checkpoint_id = "specific_checkpoint_id"
state = graph.get_state({"checkpoint_id": checkpoint_id})
print(f"检查点状态: {state.values}")
```

**从历史状态恢复执行**

```python
# 从第2步重新执行
step_2_config = checkpoints[2].config

# 继续执行(使用第2步的状态)
result = graph.invoke(None, config=step_2_config)
# 会从第2步之后的节点继续
```

**修改历史并重新执行**

```python
# 获取第2步的检查点
step_2 = checkpoints[2]

# 修改状态
graph.update_state(
    step_2.config,
    {"score": 0.9}  # 修改某个字段
)

# 从修改后的状态继续执行
result = graph.invoke(None, config=step_2.config)
# 会使用修改后的状态继续
```

#### 9.2 实战片段:多天跨度的项目管理流程

让我们构建一个支持长时间运行的项目管理工作流:

```python
from typing import Annotated, Literal, TypedDict
from operator import add
from datetime import datetime
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

# 状态定义
class ProjectState(TypedDict):
    project_id: str
    phase: Literal["planning", "development", "testing", "deployment"]
    tasks: Annotated[list[dict], add]
    current_day: int
    total_days: int
    logs: Annotated[list[str], add]

# 节点1:规划阶段
def planning_phase(state: ProjectState) -> dict:
    """项目规划"""
    log = f"Day {state['current_day']}: 完成项目规划"
    print(log)

    tasks = [
        {"id": 1, "name": "需求分析", "status": "completed"},
        {"id": 2, "name": "架构设计", "status": "completed"}
    ]

    return {
        "phase": "development",
        "tasks": tasks,
        "current_day": state["current_day"] + 1,
        "logs": [log]
    }

# 节点2:开发阶段
def development_phase(state: ProjectState) -> dict:
    """开发阶段(可能跨多天)"""
    day = state["current_day"]
    log = f"Day {day}: 开发进行中..."
    print(log)

    new_tasks = [
        {"id": 3, "name": "前端开发", "status": "in_progress"},
        {"id": 4, "name": "后端开发", "status": "in_progress"}
    ]

    # 模拟多天开发
    if day < 5:
        # 继续开发
        return {
            "current_day": day + 1,
            "logs": [log]
        }
    else:
        # 开发完成,进入测试
        return {
            "phase": "testing",
            "tasks": new_tasks,
            "current_day": day + 1,
            "logs": [log + " (开发完成)"]
        }

# 节点3:测试阶段
def testing_phase(state: ProjectState) -> dict:
    """测试阶段"""
    log = f"Day {state['current_day']}: 执行测试"
    print(log)

    return {
        "phase": "deployment",
        "current_day": state["current_day"] + 1,
        "logs": [log]
    }

# 节点4:部署阶段
def deployment_phase(state: ProjectState) -> dict:
    """部署阶段"""
    log = f"Day {state['current_day']}: 项目已部署"
    print(log)

    return {
        "current_day": state["current_day"] + 1,
        "logs": [log]
    }

# 路由函数
def route_by_phase(state: ProjectState) -> str:
    """根据当前阶段路由"""
    phase = state["phase"]

    if phase == "planning":
        return "planning"
    elif phase == "development":
        return "development"
    elif phase == "testing":
        return "testing"
    elif phase == "deployment":
        return "deployment"
    else:
        return END

# 开发阶段路由
def route_development(state: ProjectState) -> str:
    """开发阶段内部路由"""
    if state["current_day"] < 5:
        return "development"  # 继续开发
    else:
        return "next_phase"   # 进入下一阶段

# 构建图
builder = StateGraph(ProjectState)

# 添加节点
builder.add_node("planning", planning_phase)
builder.add_node("development", development_phase)
builder.add_node("testing", testing_phase)
builder.add_node("deployment", deployment_phase)

# 添加边
builder.add_edge(START, "planning")

# 从 planning 到 development
builder.add_edge("planning", "development")

# development 内部循环或进入 testing
builder.add_conditional_edges(
    "development",
    route_development,
    {
        "development": "development",  # 循环
        "next_phase": "testing"
    }
)

builder.add_edge("testing", "deployment")
builder.add_edge("deployment", END)

# 使用 SQLite 持久化
conn = sqlite3.connect("project.db", check_same_thread=False)
checkpointer = SqliteSaver(conn)
graph = builder.compile(checkpointer=checkpointer)

# ========== 模拟多天执行 ==========

config = {"configurable": {"thread_id": "project_001"}}

# Day 1: 开始项目
print("=== Day 1: 启动项目 ===")
result = graph.invoke({
    "project_id": "project_001",
    "phase": "planning",
    "tasks": [],
    "current_day": 1,
    "total_days": 10,
    "logs": []
}, config=config)

print(f"\n当前阶段: {result['phase']}")
print(f"当前天数: {result['current_day']}\n")

# Day 2-3: 继续执行(模拟系统重启)
print("=== Days 2-3: 系统重启,从检查点恢复 ===")
for day in range(2):
    result = graph.invoke(None, config=config)  # None 表示继续
    print(f"当前阶段: {result['phase']}, Day: {result['current_day']}")

# 查看执行历史
print("\n=== 执行历史 ===")
history = list(graph.get_state_history(config))
print(f"总共 {len(history)} 个检查点")

for i, checkpoint in enumerate(history[:5]):  # 只显示前5个
    print(f"\n检查点 {i}:")
    print(f"  阶段: {checkpoint.values.get('phase', 'N/A')}")
    print(f"  天数: {checkpoint.values.get('current_day', 'N/A')}")
    print(f"  源节点: {checkpoint.metadata.get('source', 'N/A')}")

# 时间旅行:回到 Day 2
print("\n=== 时间旅行:回到 Day 2 ===")
day_2_checkpoint = [c for c in history if c.values.get("current_day") == 2][0]
state_at_day_2 = graph.get_state(day_2_checkpoint.config)
print(f"Day 2 状态: 阶段={state_at_day_2.values['phase']}, 任务数={len(state_at_day_2.values['tasks'])}")

# 从 Day 2 重新执行
print("\n从 Day 2 重新执行...")
result = graph.invoke(None, config=day_2_checkpoint.config)
print(f"重新执行后: {result['phase']}, Day {result['current_day']}")
```

**执行流程**

```
Day 1:
  START → planning → development (Day 2)
  ↓ 保存检查点

Day 2 (系统重启):
  从检查点恢复 → development → development (Day 3)
  ↓ 保存检查点

Day 3:
  从检查点恢复 → development → development (Day 4)
  ↓ 保存检查点

... (继续执行)

时间旅行:
  回到 Day 2 的检查点
  从 Day 2 重新执行
  可以看到不同的执行路径
```

#### 9.3 最佳实践

##### 9.3.1 存储后端选择

**原则**:根据应用需求选择合适的存储后端

**场景1:开发和测试**

```python
from langgraph.checkpoint.memory import InMemorySaver

# ✅ 优点:
# - 无需配置
# - 速度快
# - 适合快速迭代

# ❌ 缺点:
# - 进程重启后丢失
# - 不支持多进程

checkpointer = InMemorySaver()
```

**场景2:单机应用**

```python
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

# ✅ 优点:
# - 持久化
# - 无需外部服务
# - 适合中小型应用

# ❌ 缺点:
# - 不支持分布式
# - 并发性能有限

conn = sqlite3.connect("app.db")
checkpointer = SqliteSaver(conn)
```

**场景3:生产环境/分布式系统**

```python
from langgraph.checkpoint.postgres import PostgresSaver
import psycopg

# ✅ 优点:
# - 持久化
# - 支持分布式
# - 高并发性能
# - 支持备份和复制

# ❌ 缺点:
# - 需要 PostgreSQL 服务
# - 配置相对复杂

conn = psycopg.connect("postgresql://user:pass@host:5432/db")
checkpointer = PostgresSaver(conn)
```

**存储后端对比**

| 特性 | InMemorySaver | SqliteSaver | PostgresSaver |
|------|--------------|-------------|---------------|
| 持久化 | ❌ | ✅ | ✅ |
| 分布式 | ❌ | ❌ | ✅ |
| 性能 | 极快 | 快 | 快 |
| 并发 | 低 | 中 | 高 |
| 运维成本 | 无 | 低 | 中 |

##### 9.3.2 检查点粒度控制

**原则**:不是所有状态更新都需要保存检查点

**策略1:只在关键节点保存**

```python
# 只在重要节点后保存检查点
graph = builder.compile(
    checkpointer=checkpointer,
    # 注意:LangGraph 默认在每个节点后都保存
    # 如果要减少检查点,可以在状态中添加标志
)

def expensive_node(state: State) -> dict:
    """耗时的节点"""
    # 执行大量计算...
    result = heavy_computation()

    # 明确标记需要保存检查点
    return {
        "result": result,
        "_checkpoint": True  # 自定义标志
    }
```

**策略2:采样保存**

```python
class State(TypedDict):
    iteration: int
    data: dict
    checkpoint_every: int  # 每N次迭代保存

def iterative_node(state: State) -> dict:
    """迭代节点"""
    iteration = state["iteration"] + 1

    # 只在特定迭代保存检查点
    should_checkpoint = (iteration % state["checkpoint_every"] == 0)

    return {
        "iteration": iteration,
        "data": process(state["data"]),
        "_save_checkpoint": should_checkpoint
    }
```

**策略3:基于状态大小**

```python
import sys

def smart_checkpoint(state: State) -> dict:
    """根据状态大小决定是否保存"""
    state_size = sys.getsizeof(str(state))

    # 状态太大时跳过检查点
    if state_size > 1_000_000:  # 1MB
        return {"_skip_checkpoint": True}

    return {}
```

##### 9.3.3 状态清理策略

**原则**:避免无限增长的状态历史

**策略1:限制检查点数量**

```python
# 在应用层实现清理
def cleanup_old_checkpoints(thread_id: str, keep_latest: int = 10):
    """保留最新的N个检查点"""
    config = {"configurable": {"thread_id": thread_id}}
    history = list(graph.get_state_history(config))

    if len(history) > keep_latest:
        # 删除旧检查点(伪代码)
        old_checkpoints = history[keep_latest:]
        for checkpoint in old_checkpoints:
            checkpointer.delete(checkpoint.config)

# 定期清理
cleanup_old_checkpoints("user_123", keep_latest=20)
```

**策略2:基于时间的清理**

```python
from datetime import datetime, timedelta

def cleanup_by_age(thread_id: str, max_age_days: int = 7):
    """删除超过N天的检查点"""
    config = {"configurable": {"thread_id": thread_id}}
    history = list(graph.get_state_history(config))

    cutoff = datetime.now() - timedelta(days=max_age_days)

    for checkpoint in history:
        timestamp = checkpoint.metadata.get("ts")
        if timestamp < cutoff:
            checkpointer.delete(checkpoint.config)
```

**策略3:压缩旧检查点**

```python
def compress_old_checkpoints(thread_id: str, threshold_days: int = 3):
    """压缩旧检查点,只保留关键信息"""
    config = {"configurable": {"thread_id": thread_id}}
    history = list(graph.get_state_history(config))

    cutoff = datetime.now() - timedelta(days=threshold_days)

    for checkpoint in history:
        if checkpoint.metadata.get("ts") < cutoff:
            # 只保留摘要
            compressed_state = {
                "summary": summarize(checkpoint.values),
                "metadata": checkpoint.metadata
            }
            checkpointer.update(checkpoint.config, compressed_state)
```

**数据库层面的优化**

```sql
-- 为 PostgreSQL 设置自动清理
CREATE INDEX idx_thread_timestamp ON checkpoints(thread_id, ts);

-- 定期删除旧数据
DELETE FROM checkpoints
WHERE ts < NOW() - INTERVAL '30 days';

-- 或者只保留最新N个
DELETE FROM checkpoints
WHERE id NOT IN (
  SELECT id FROM checkpoints
  WHERE thread_id = 'user_123'
  ORDER BY ts DESC
  LIMIT 20
);
```

**小结**

持久化与恢复的最佳实践:
- ✅ 根据场景选择合适的存储后端
- ✅ 控制检查点粒度,避免过度保存
- ✅ 定期清理旧检查点,控制存储增长
- ✅ 利用时间旅行调试复杂问题
- ✅ 为长时间运行的任务提供可靠性保障

持久化让 Agent 能够跨会话、跨故障运行,是生产级应用的基础设施。

---

## 第三篇:高级特性篇

掌握复杂场景的解决方案

### 第 10 章:子图与模块化

#### 10.1 为什么需要子图?

**问题背景**

随着 Agent 应用复杂度增加,我们会遇到以下挑战:

**挑战1:图结构过于复杂**

```
一个包含 20+ 节点的扁平图:

start → analyze → classify → route1 → route2 → route3 →
validate1 → validate2 → format1 → format2 → review1 →
review2 → approve1 → approve2 → execute1 → execute2 →
notify1 → notify2 → log1 → log2 → end
```

问题:
- 难以理解整体逻辑
- 修改一个部分容易影响其他部分
- 测试困难,无法单独测试某个逻辑块
- 代码复用困难

**挑战2:职责边界不清晰**

```
项目管理 Agent 的图结构:

需求分析 → 任务分解 → 资源分配 → 进度跟踪 →
风险评估 → 报告生成 → 通知发送

每个环节都很复杂,混在一起难以维护
```

**挑战3:团队协作困难**

```
团队 A: 负责需求分析模块
团队 B: 负责任务执行模块
团队 C: 负责报告生成模块

如果在同一个图中开发,容易产生冲突
```

**子图的价值**

SubGraph(子图)是 LangGraph 提供的模块化机制,它允许:

1. **层次化组织**: 将复杂图分解为多个逻辑单元
2. **关注点分离**: 每个子图专注于一个职责
3. **代码复用**: 子图可以在多个父图中复用
4. **独立测试**: 每个子图可以单独测试
5. **团队协作**: 不同团队可以独立开发各自的子图

#### 10.2 子图基础

##### 10.2.1 创建和使用子图

**基本概念**

子图本质上是一个完整的 LangGraph 图,可以作为节点嵌入到另一个图中。

**简单示例:验证模块**

```python
from typing import TypedDict
from langgraph.graph import StateGraph

# 1. 定义子图的状态
class ValidationState(TypedDict):
    content: str
    is_valid: bool
    errors: list[str]

# 2. 创建子图
def create_validation_subgraph():
    """创建一个内容验证子图"""
    builder = StateGraph(ValidationState)

    def check_length(state: ValidationState) -> dict:
        """检查长度"""
        content = state["content"]
        errors = state.get("errors", [])

        if len(content) < 10:
            errors.append("内容太短")
        if len(content) > 1000:
            errors.append("内容太长")

        return {"errors": errors}

    def check_format(state: ValidationState) -> dict:
        """检查格式"""
        content = state["content"]
        errors = state.get("errors", [])

        if not content.strip():
            errors.append("内容为空")

        return {"errors": errors}

    def finalize(state: ValidationState) -> dict:
        """最终判断"""
        is_valid = len(state.get("errors", [])) == 0
        return {"is_valid": is_valid}

    # 构建子图
    builder.add_node("check_length", check_length)
    builder.add_node("check_format", check_format)
    builder.add_node("finalize", finalize)

    builder.set_entry_point("check_length")
    builder.add_edge("check_length", "check_format")
    builder.add_edge("check_format", "finalize")
    builder.set_finish_point("finalize")

    return builder.compile()

# 3. 在父图中使用子图
class MainState(TypedDict):
    user_input: str
    validation_result: bool
    final_output: str

def create_main_graph():
    """创建主图"""
    builder = StateGraph(MainState)

    # 创建子图实例
    validation_subgraph = create_validation_subgraph()

    def prepare_validation(state: MainState) -> dict:
        """准备验证输入"""
        return {
            "content": state["user_input"],
            "errors": []
        }

    def run_validation(state: MainState) -> dict:
        """运行验证子图"""
        # 调用子图
        result = validation_subgraph.invoke({
            "content": state["user_input"],
            "is_valid": False,
            "errors": []
        })

        return {"validation_result": result["is_valid"]}

    def process_result(state: MainState) -> dict:
        """处理验证结果"""
        if state["validation_result"]:
            return {"final_output": "验证通过,继续处理"}
        else:
            return {"final_output": "验证失败,请修改输入"}

    builder.add_node("run_validation", run_validation)
    builder.add_node("process_result", process_result)

    builder.set_entry_point("run_validation")
    builder.add_edge("run_validation", "process_result")
    builder.set_finish_point("process_result")

    return builder.compile()

# 使用
graph = create_main_graph()
result = graph.invoke({
    "user_input": "这是一段测试内容",
    "validation_result": False,
    "final_output": ""
})
print(result["final_output"])
```

##### 10.2.2 父子图状态映射

**核心挑战**

父图和子图通常有不同的状态结构,需要进行状态映射。

**映射策略1:手动映射(推荐)**

```python
class ParentState(TypedDict):
    user_input: str
    processing_data: dict
    result: str

class ChildState(TypedDict):
    input_text: str
    output_text: str

def create_child_graph():
    """子图"""
    builder = StateGraph(ChildState)

    def process(state: ChildState) -> dict:
        return {"output_text": state["input_text"].upper()}

    builder.add_node("process", process)
    builder.set_entry_point("process")
    builder.set_finish_point("process")

    return builder.compile()

def create_parent_graph():
    """父图"""
    builder = StateGraph(ParentState)

    child_graph = create_child_graph()

    def invoke_child(state: ParentState) -> dict:
        """调用子图,手动映射状态"""
        # 父状态 → 子状态
        child_input = {
            "input_text": state["user_input"],
            "output_text": ""
        }

        # 执行子图
        child_output = child_graph.invoke(child_input)

        # 子状态 → 父状态
        return {
            "result": child_output["output_text"],
            "processing_data": {"child_output": child_output}
        }

    builder.add_node("invoke_child", invoke_child)
    builder.set_entry_point("invoke_child")
    builder.set_finish_point("invoke_child")

    return builder.compile()
```

**映射策略2:共享状态结构**

```python
# 定义共享的基础状态
class BaseState(TypedDict):
    id: str
    timestamp: str

class ParentState(BaseState):
    """父图状态,继承基础状态"""
    user_input: str
    result: str

class ChildState(BaseState):
    """子图状态,继承基础状态"""
    data: str

def create_child_graph():
    builder = StateGraph(ChildState)

    def process(state: ChildState) -> dict:
        # 可以访问 id 和 timestamp
        return {
            "data": f"Processed at {state['timestamp']}"
        }

    builder.add_node("process", process)
    builder.set_entry_point("process")
    builder.set_finish_point("process")

    return builder.compile()

def create_parent_graph():
    builder = StateGraph(ParentState)

    child_graph = create_child_graph()

    def invoke_child(state: ParentState) -> dict:
        # 传递共享字段
        child_output = child_graph.invoke({
            "id": state["id"],
            "timestamp": state["timestamp"],
            "data": state["user_input"]
        })

        return {"result": child_output["data"]}

    builder.add_node("invoke_child", invoke_child)
    builder.set_entry_point("invoke_child")
    builder.set_finish_point("invoke_child")

    return builder.compile()
```

**映射策略3:使用转换函数**

```python
def create_state_adapter(parent_to_child, child_to_parent):
    """创建状态适配器"""
    def adapter(child_graph):
        def adapted_node(parent_state):
            # 转换
            child_input = parent_to_child(parent_state)
            # 执行
            child_output = child_graph.invoke(child_input)
            # 转换回来
            return child_to_parent(parent_state, child_output)

        return adapted_node

    return adapter

# 使用
def parent_to_child_mapper(parent_state: ParentState) -> ChildState:
    """父状态 → 子状态"""
    return {
        "input_text": parent_state["user_input"],
        "output_text": ""
    }

def child_to_parent_mapper(parent_state: ParentState, child_state: ChildState) -> dict:
    """子状态 → 父状态更新"""
    return {
        "result": child_state["output_text"]
    }

# 创建适配器
adapter = create_state_adapter(
    parent_to_child_mapper,
    child_to_parent_mapper
)

# 在父图中使用
child_graph = create_child_graph()
adapted_child = adapter(child_graph)

builder.add_node("child", adapted_child)
```

#### 10.3 图组合模式

##### 10.3.1 模式1:串行组合

**场景**: 多个子图按顺序执行

```python
class PipelineState(TypedDict):
    raw_input: str
    preprocessed: str
    analyzed: str
    final_output: str

def create_preprocessing_graph():
    """预处理子图"""
    builder = StateGraph(PipelineState)

    def clean_text(state: PipelineState) -> dict:
        text = state["raw_input"].strip().lower()
        return {"preprocessed": text}

    builder.add_node("clean", clean_text)
    builder.set_entry_point("clean")
    builder.set_finish_point("clean")

    return builder.compile()

def create_analysis_graph():
    """分析子图"""
    builder = StateGraph(PipelineState)

    def analyze(state: PipelineState) -> dict:
        text = state["preprocessed"]
        result = f"分析结果: {len(text)} 字符"
        return {"analyzed": result}

    builder.add_node("analyze", analyze)
    builder.set_entry_point("analyze")
    builder.set_finish_point("analyze")

    return builder.compile()

def create_pipeline_graph():
    """串行流水线"""
    builder = StateGraph(PipelineState)

    preprocess = create_preprocessing_graph()
    analyze = create_analysis_graph()

    def run_preprocess(state: PipelineState) -> dict:
        result = preprocess.invoke(state)
        return {"preprocessed": result["preprocessed"]}

    def run_analyze(state: PipelineState) -> dict:
        result = analyze.invoke(state)
        return {"analyzed": result["analyzed"]}

    def finalize(state: PipelineState) -> dict:
        return {"final_output": state["analyzed"]}

    builder.add_node("preprocess", run_preprocess)
    builder.add_node("analyze", run_analyze)
    builder.add_node("finalize", finalize)

    builder.set_entry_point("preprocess")
    builder.add_edge("preprocess", "analyze")
    builder.add_edge("analyze", "finalize")
    builder.set_finish_point("finalize")

    return builder.compile()

# 使用
graph = create_pipeline_graph()
result = graph.invoke({
    "raw_input": "  Hello World  ",
    "preprocessed": "",
    "analyzed": "",
    "final_output": ""
})
```

**执行流程**:
```
raw_input
   ↓
[预处理子图] → preprocessed
   ↓
[分析子图] → analyzed
   ↓
finalize → final_output
```

##### 10.3.2 模式2:并行组合

**场景**: 多个子图同时执行,聚合结果

```python
from typing import List

class ParallelState(TypedDict):
    input_data: str
    results: List[str]
    final_summary: str

def create_analyzer_a():
    """分析器A:检查长度"""
    builder = StateGraph(ParallelState)

    def analyze(state: ParallelState) -> dict:
        length = len(state["input_data"])
        return {"results": [f"长度: {length}"]}

    builder.add_node("analyze", analyze)
    builder.set_entry_point("analyze")
    builder.set_finish_point("analyze")

    return builder.compile()

def create_analyzer_b():
    """分析器B:检查单词数"""
    builder = StateGraph(ParallelState)

    def analyze(state: ParallelState) -> dict:
        words = len(state["input_data"].split())
        return {"results": [f"单词数: {words}"]}

    builder.add_node("analyze", analyze)
    builder.set_entry_point("analyze")
    builder.set_finish_point("analyze")

    return builder.compile()

def create_parallel_graph():
    """并行执行多个分析器"""
    builder = StateGraph(ParallelState)

    analyzer_a = create_analyzer_a()
    analyzer_b = create_analyzer_b()

    def run_analyzer_a(state: ParallelState) -> dict:
        result = analyzer_a.invoke(state)
        return {"results": state.get("results", []) + result["results"]}

    def run_analyzer_b(state: ParallelState) -> dict:
        result = analyzer_b.invoke(state)
        return {"results": state.get("results", []) + result["results"]}

    def aggregate(state: ParallelState) -> dict:
        summary = " | ".join(state["results"])
        return {"final_summary": summary}

    builder.add_node("analyzer_a", run_analyzer_a)
    builder.add_node("analyzer_b", run_analyzer_b)
    builder.add_node("aggregate", aggregate)

    builder.set_entry_point("analyzer_a")
    builder.add_edge("analyzer_a", "analyzer_b")
    builder.add_edge("analyzer_b", "aggregate")
    builder.set_finish_point("aggregate")

    return builder.compile()

# 注意:真正的并行需要使用 Send API(高级特性)
# 这里的示例是串行执行但逻辑上独立的子图
```

**真正的并行执行**(使用 Send API):

```python
from langgraph.constants import Send

def create_true_parallel_graph():
    """真正的并行执行"""
    builder = StateGraph(ParallelState)

    analyzer_a = create_analyzer_a()
    analyzer_b = create_analyzer_b()

    def fan_out(state: ParallelState):
        """扇出:启动多个并行任务"""
        return [
            Send("analyzer_a", state),
            Send("analyzer_b", state)
        ]

    def run_analyzer_a(state: ParallelState) -> dict:
        result = analyzer_a.invoke(state)
        return {"results": result["results"]}

    def run_analyzer_b(state: ParallelState) -> dict:
        result = analyzer_b.invoke(state)
        return {"results": result["results"]}

    def aggregate(state: ParallelState) -> dict:
        summary = " | ".join(state["results"])
        return {"final_summary": summary}

    builder.add_node("fan_out", fan_out)
    builder.add_node("analyzer_a", run_analyzer_a)
    builder.add_node("analyzer_b", run_analyzer_b)
    builder.add_node("aggregate", aggregate)

    builder.set_entry_point("fan_out")
    # fan_out 返回 Send 对象,自动路由到对应节点
    builder.add_edge("analyzer_a", "aggregate")
    builder.add_edge("analyzer_b", "aggregate")
    builder.set_finish_point("aggregate")

    return builder.compile()
```

##### 10.3.3 模式3:条件组合

**场景**: 根据条件选择不同的子图

```python
class RoutingState(TypedDict):
    task_type: str
    input_data: str
    output: str

def create_simple_processor():
    """简单处理器"""
    builder = StateGraph(RoutingState)

    def process(state: RoutingState) -> dict:
        return {"output": f"简单处理: {state['input_data']}"}

    builder.add_node("process", process)
    builder.set_entry_point("process")
    builder.set_finish_point("process")

    return builder.compile()

def create_complex_processor():
    """复杂处理器"""
    builder = StateGraph(RoutingState)

    def analyze(state: RoutingState) -> dict:
        return {"output": f"复杂分析: {state['input_data']}"}

    builder.add_node("analyze", analyze)
    builder.set_entry_point("analyze")
    builder.set_finish_point("analyze")

    return builder.compile()

def create_routing_graph():
    """条件路由图"""
    builder = StateGraph(RoutingState)

    simple_processor = create_simple_processor()
    complex_processor = create_complex_processor()

    def route_to_processor(state: RoutingState) -> dict:
        """根据任务类型选择处理器"""
        if state["task_type"] == "simple":
            result = simple_processor.invoke(state)
        else:
            result = complex_processor.invoke(state)

        return {"output": result["output"]}

    builder.add_node("route", route_to_processor)
    builder.set_entry_point("route")
    builder.set_finish_point("route")

    return builder.compile()

# 使用
graph = create_routing_graph()

# 简单任务
result1 = graph.invoke({
    "task_type": "simple",
    "input_data": "test",
    "output": ""
})

# 复杂任务
result2 = graph.invoke({
    "task_type": "complex",
    "input_data": "test",
    "output": ""
})
```

##### 10.3.4 模式4:递归组合

**场景**: 子图包含对自身或其他子图的递归调用

```python
class RecursiveState(TypedDict):
    items: List[str]
    current_index: int
    processed_items: List[str]

def create_recursive_processor():
    """递归处理器"""
    builder = StateGraph(RecursiveState)

    def process_item(state: RecursiveState) -> dict:
        """处理当前项"""
        items = state["items"]
        current_index = state["current_index"]
        processed = state.get("processed_items", [])

        if current_index >= len(items):
            # 递归终止
            return {"processed_items": processed}

        # 处理当前项
        current_item = items[current_index]
        processed_item = f"处理: {current_item}"
        processed.append(processed_item)

        return {
            "current_index": current_index + 1,
            "processed_items": processed
        }

    def should_continue(state: RecursiveState) -> str:
        """判断是否继续递归"""
        if state["current_index"] >= len(state["items"]):
            return "done"
        return "continue"

    builder.add_node("process", process_item)
    builder.set_entry_point("process")

    builder.add_conditional_edges(
        "process",
        should_continue,
        {
            "continue": "process",  # 递归回到自己
            "done": END
        }
    )

    return builder.compile()

# 使用
graph = create_recursive_processor()
result = graph.invoke({
    "items": ["A", "B", "C"],
    "current_index": 0,
    "processed_items": []
})
print(result["processed_items"])
# 输出: ['处理: A', '处理: B', '处理: C']
```

#### 10.4 实战案例:模块化的项目管理助手

**场景**: 将项目管理助手分解为多个子图

```python
from typing import TypedDict, List

# ============= 状态定义 =============

class RequirementAnalysisState(TypedDict):
    """需求分析子图状态"""
    raw_requirement: str
    analyzed_requirements: List[dict]
    priority: str

class TaskDecompositionState(TypedDict):
    """任务分解子图状态"""
    requirements: List[dict]
    tasks: List[dict]

class ResourceAllocationState(TypedDict):
    """资源分配子图状态"""
    tasks: List[dict]
    available_resources: List[str]
    allocated_tasks: List[dict]

class ProjectState(TypedDict):
    """主项目状态"""
    user_input: str
    requirements: List[dict]
    tasks: List[dict]
    allocated_tasks: List[dict]
    project_plan: str

# ============= 子图1: 需求分析 =============

def create_requirement_analysis_subgraph():
    """需求分析子图"""
    builder = StateGraph(RequirementAnalysisState)

    def parse_requirement(state: RequirementAnalysisState) -> dict:
        """解析需求"""
        raw = state["raw_requirement"]
        # 简化:实际应该用 LLM
        requirements = [
            {"id": 1, "text": raw, "type": "feature"}
        ]
        return {"analyzed_requirements": requirements}

    def classify_priority(state: RequirementAnalysisState) -> dict:
        """分类优先级"""
        # 简化:实际应该用 LLM
        return {"priority": "high"}

    builder.add_node("parse", parse_requirement)
    builder.add_node("classify", classify_priority)

    builder.set_entry_point("parse")
    builder.add_edge("parse", "classify")
    builder.set_finish_point("classify")

    return builder.compile()

# ============= 子图2: 任务分解 =============

def create_task_decomposition_subgraph():
    """任务分解子图"""
    builder = StateGraph(TaskDecompositionState)

    def decompose_to_tasks(state: TaskDecompositionState) -> dict:
        """分解为任务"""
        requirements = state["requirements"]
        tasks = []

        for req in requirements:
            # 简化:实际应该用 LLM
            tasks.append({
                "id": len(tasks) + 1,
                "requirement_id": req["id"],
                "name": f"实现 {req['text']}",
                "estimated_hours": 8
            })

        return {"tasks": tasks}

    def validate_tasks(state: TaskDecompositionState) -> dict:
        """验证任务"""
        # 验证逻辑
        return {}

    builder.add_node("decompose", decompose_to_tasks)
    builder.add_node("validate", validate_tasks)

    builder.set_entry_point("decompose")
    builder.add_edge("decompose", "validate")
    builder.set_finish_point("validate")

    return builder.compile()

# ============= 子图3: 资源分配 =============

def create_resource_allocation_subgraph():
    """资源分配子图"""
    builder = StateGraph(ResourceAllocationState)

    def allocate_resources(state: ResourceAllocationState) -> dict:
        """分配资源"""
        tasks = state["tasks"]
        resources = state["available_resources"]

        allocated = []
        for i, task in enumerate(tasks):
            resource = resources[i % len(resources)]
            allocated.append({
                **task,
                "assigned_to": resource
            })

        return {"allocated_tasks": allocated}

    builder.add_node("allocate", allocate_resources)
    builder.set_entry_point("allocate")
    builder.set_finish_point("allocate")

    return builder.compile()

# ============= 主图: 组合所有子图 =============

def create_project_management_graph():
    """项目管理主图"""
    builder = StateGraph(ProjectState)

    # 创建子图实例
    requirement_analysis = create_requirement_analysis_subgraph()
    task_decomposition = create_task_decomposition_subgraph()
    resource_allocation = create_resource_allocation_subgraph()

    def analyze_requirements(state: ProjectState) -> dict:
        """运行需求分析子图"""
        result = requirement_analysis.invoke({
            "raw_requirement": state["user_input"],
            "analyzed_requirements": [],
            "priority": ""
        })

        return {"requirements": result["analyzed_requirements"]}

    def decompose_tasks(state: ProjectState) -> dict:
        """运行任务分解子图"""
        result = task_decomposition.invoke({
            "requirements": state["requirements"],
            "tasks": []
        })

        return {"tasks": result["tasks"]}

    def allocate_resources(state: ProjectState) -> dict:
        """运行资源分配子图"""
        result = resource_allocation.invoke({
            "tasks": state["tasks"],
            "available_resources": ["Alice", "Bob", "Charlie"],
            "allocated_tasks": []
        })

        return {"allocated_tasks": result["allocated_tasks"]}

    def generate_plan(state: ProjectState) -> dict:
        """生成项目计划"""
        tasks = state["allocated_tasks"]
        plan_lines = ["项目计划:\n"]

        for task in tasks:
            plan_lines.append(
                f"- {task['name']} "
                f"({task['estimated_hours']}h) "
                f"-> {task['assigned_to']}"
            )

        return {"project_plan": "\n".join(plan_lines)}

    # 构建主图
    builder.add_node("analyze", analyze_requirements)
    builder.add_node("decompose", decompose_tasks)
    builder.add_node("allocate", allocate_resources)
    builder.add_node("generate_plan", generate_plan)

    builder.set_entry_point("analyze")
    builder.add_edge("analyze", "decompose")
    builder.add_edge("decompose", "allocate")
    builder.add_edge("allocate", "generate_plan")
    builder.set_finish_point("generate_plan")

    return builder.compile()

# ============= 使用 =============

graph = create_project_management_graph()

result = graph.invoke({
    "user_input": "开发一个用户登录功能",
    "requirements": [],
    "tasks": [],
    "allocated_tasks": [],
    "project_plan": ""
})

print(result["project_plan"])
```

**输出**:
```
项目计划:

- 实现 开发一个用户登录功能 (8h) -> Alice
```

**架构优势**:

| 方面 | 单一图 | 子图模式 |
|------|--------|----------|
| **可读性** | 节点过多,难以理解 | 层次清晰,易于理解 |
| **可维护性** | 修改影响范围大 | 修改局限在子图内 |
| **可测试性** | 只能整体测试 | 每个子图独立测试 |
| **可复用性** | 难以复用 | 子图可在多处复用 |
| **团队协作** | 容易冲突 | 并行开发不冲突 |

#### 10.5 最佳实践

##### 10.5.1 子图设计原则

**原则1:单一职责**

每个子图应该只做一件事情。

```python
# ❌ 不好:一个子图做太多事情
def create_everything_subgraph():
    """一个子图处理所有逻辑"""
    # 包含:验证、分析、转换、存储...
    # 太复杂,难以维护

# ✅ 好:每个子图专注一个职责
def create_validation_subgraph():
    """只负责验证"""
    pass

def create_analysis_subgraph():
    """只负责分析"""
    pass

def create_transformation_subgraph():
    """只负责转换"""
    pass
```

**原则2:明确的输入输出**

子图的状态应该清晰定义输入和输出。

```python
# ✅ 好:明确的状态定义
class ValidationInput(TypedDict):
    """验证子图的输入"""
    content: str
    rules: List[str]

class ValidationOutput(TypedDict):
    """验证子图的输出"""
    is_valid: bool
    errors: List[str]
    warnings: List[str]

# 在文档中说明
def create_validation_subgraph():
    """
    验证子图

    输入:
        - content: 待验证的内容
        - rules: 验证规则列表

    输出:
        - is_valid: 是否通过验证
        - errors: 错误列表
        - warnings: 警告列表
    """
    pass
```

**原则3:最小化依赖**

子图应该尽量减少对外部状态的依赖。

```python
# ❌ 不好:依赖全局状态
global_config = {}

def create_subgraph():
    def process(state):
        # 依赖全局变量
        setting = global_config["setting"]
        return {}

# ✅ 好:依赖注入
def create_subgraph(config: dict):
    """通过参数传入依赖"""
    def process(state):
        setting = config["setting"]
        return {}

    # 构建图...
    return builder.compile()
```

##### 10.5.2 状态映射策略

**策略1:使用适配器模式**

```python
class StateAdapter:
    """状态适配器"""

    def __init__(self, subgraph, to_child, from_child):
        self.subgraph = subgraph
        self.to_child = to_child
        self.from_child = from_child

    def __call__(self, parent_state):
        # 转换为子图状态
        child_state = self.to_child(parent_state)

        # 执行子图
        result = self.subgraph.invoke(child_state)

        # 转换回父图状态
        return self.from_child(parent_state, result)

# 使用
validation_subgraph = create_validation_subgraph()

adapter = StateAdapter(
    subgraph=validation_subgraph,
    to_child=lambda p: {"content": p["user_input"], "rules": []},
    from_child=lambda p, c: {"is_valid": c["is_valid"]}
)

builder.add_node("validate", adapter)
```

**策略2:使用状态继承**

```python
class BaseState(TypedDict):
    """基础状态"""
    id: str
    timestamp: str

class ParentState(BaseState):
    """继承基础状态"""
    user_input: str

class ChildState(BaseState):
    """继承基础状态"""
    processed_data: str

# 自动共享 id 和 timestamp
```

##### 10.5.3 错误处理

**原则**: 子图的错误应该向上传播

```python
class SubgraphState(TypedDict):
    input: str
    output: str
    error: str  # 错误信息

def create_subgraph():
    builder = StateGraph(SubgraphState)

    def risky_operation(state: SubgraphState) -> dict:
        try:
            result = do_something(state["input"])
            return {"output": result, "error": ""}
        except Exception as e:
            return {"output": "", "error": str(e)}

    builder.add_node("operation", risky_operation)
    builder.set_entry_point("operation")
    builder.set_finish_point("operation")

    return builder.compile()

def create_parent_graph():
    builder = StateGraph(ParentState)

    subgraph = create_subgraph()

    def handle_subgraph(state: ParentState) -> dict:
        result = subgraph.invoke({
            "input": state["data"],
            "output": "",
            "error": ""
        })

        # 检查子图是否出错
        if result["error"]:
            return {"status": "error", "message": result["error"]}

        return {"status": "success", "result": result["output"]}

    builder.add_node("subgraph", handle_subgraph)
    # ...
```

##### 10.5.4 测试策略

**策略1:独立测试子图**

```python
import pytest

def test_validation_subgraph():
    """测试验证子图"""
    graph = create_validation_subgraph()

    # 测试通过情况
    result = graph.invoke({
        "content": "valid content",
        "is_valid": False,
        "errors": []
    })
    assert result["is_valid"] == True
    assert len(result["errors"]) == 0

    # 测试失败情况
    result = graph.invoke({
        "content": "",
        "is_valid": False,
        "errors": []
    })
    assert result["is_valid"] == False
    assert len(result["errors"]) > 0
```

**策略2:使用 Mock 测试父图**

```python
from unittest.mock import Mock

def test_parent_graph_with_mock():
    """使用 Mock 子图测试父图"""

    # 创建 Mock 子图
    mock_subgraph = Mock()
    mock_subgraph.invoke.return_value = {
        "is_valid": True,
        "errors": []
    }

    # 注入 Mock
    def create_parent_graph_for_test(subgraph):
        builder = StateGraph(ParentState)

        def use_subgraph(state):
            result = subgraph.invoke(state)
            return {"validation_result": result["is_valid"]}

        builder.add_node("validate", use_subgraph)
        # ...
        return builder.compile()

    graph = create_parent_graph_for_test(mock_subgraph)

    result = graph.invoke({"user_input": "test"})

    # 验证 Mock 被调用
    mock_subgraph.invoke.assert_called_once()
```

**小结**

子图与模块化的最佳实践:
- ✅ 使用子图将复杂图分解为逻辑模块
- ✅ 每个子图遵循单一职责原则
- ✅ 明确定义子图的输入输出状态
- ✅ 使用适配器模式处理状态映射
- ✅ 子图错误向上传播,父图统一处理
- ✅ 独立测试每个子图,提高测试覆盖率
- ✅ 使用依赖注入增强子图的可测试性

通过合理使用子图,可以构建大规模、易维护的 Agent 应用。

---

### 第 11 章:实战案例 - 智能项目管理助手完整实现

#### 11.1 需求分析

**业务场景**

我们要构建一个智能项目管理助手,能够:

1. **需求理解**: 理解用户的自然语言需求描述
2. **任务分解**: 将需求分解为具体的可执行任务
3. **资源分配**: 根据团队成员的能力和负载分配任务
4. **进度跟踪**: 跟踪任务进度,识别风险
5. **智能调整**: 根据实际情况调整计划
6. **人机协作**: 关键决策点需要人工确认

**功能需求**

| 功能 | 描述 | 优先级 |
|------|------|--------|
| 需求解析 | 从自然语言提取结构化需求 | P0 |
| 任务分解 | 生成任务列表和依赖关系 | P0 |
| 资源分配 | 智能分配任务给团队成员 | P0 |
| 风险评估 | 识别潜在风险 | P1 |
| 人工审批 | 重大决策需要人工确认 | P1 |
| 进度跟踪 | 更新任务状态 | P2 |
| 计划调整 | 根据变化重新规划 | P2 |

**技术需求**

- **状态管理**: 持久化项目状态,支持跨会话
- **错误处理**: 优雅处理 LLM 调用失败、解析错误等
- **可观测性**: 记录执行日志,便于调试
- **可扩展性**: 易于添加新功能

#### 11.2 架构设计

**图结构设计**

```
用户输入
   ↓
需求解析
   ↓
任务分解
   ↓
风险评估 → [高风险?] → 人工审批 → 继续
   ↓                         ↓
  [低风险]                   ↓
   ↓                         ↓
资源分配 ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
   ↓
生成计划
   ↓
[需要调整?] → 重新分配 → 资源分配
   ↓
输出计划
```

**状态设计**

```python
from typing import TypedDict, List, Literal

class Task(TypedDict):
    """任务结构"""
    id: str
    name: str
    description: str
    estimated_hours: float
    dependencies: List[str]
    assigned_to: str
    status: Literal["pending", "in_progress", "completed"]

class TeamMember(TypedDict):
    """团队成员"""
    name: str
    skills: List[str]
    current_load: float  # 当前负载(小时)

class Risk(TypedDict):
    """风险"""
    level: Literal["low", "medium", "high"]
    description: str
    mitigation: str

class ProjectState(TypedDict):
    """项目状态"""
    # 输入
    user_requirement: str

    # 中间结果
    parsed_requirements: List[dict]
    tasks: List[Task]
    risks: List[Risk]
    team_members: List[TeamMember]

    # 控制流
    needs_approval: bool
    approval_status: Literal["pending", "approved", "rejected"]
    needs_reallocation: bool

    # 输出
    project_plan: str
    messages: List[str]  # 日志消息
```

#### 11.3 完整实现

```python
from typing import TypedDict, List, Literal, Annotated
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3
import json

# ============= 状态定义 =============

class Task(TypedDict):
    id: str
    name: str
    description: str
    estimated_hours: float
    dependencies: List[str]
    assigned_to: str
    status: Literal["pending", "in_progress", "completed"]

class TeamMember(TypedDict):
    name: str
    skills: List[str]
    current_load: float

class Risk(TypedDict):
    level: Literal["low", "medium", "high"]
    description: str
    mitigation: str

class ProjectState(TypedDict):
    user_requirement: str
    parsed_requirements: List[dict]
    tasks: List[Task]
    risks: List[Risk]
    team_members: List[TeamMember]
    needs_approval: bool
    approval_status: Literal["pending", "approved", "rejected", ""]
    needs_reallocation: bool
    project_plan: str
    messages: Annotated[List[str], "append"]  # 使用 append reducer

# ============= 工具函数 =============

def log_message(state: ProjectState, message: str) -> dict:
    """添加日志消息"""
    return {"messages": [f"[LOG] {message}"]}

# ============= 节点实现 =============

def parse_requirements(state: ProjectState) -> dict:
    """解析需求"""
    requirement = state["user_requirement"]

    # 简化版本:实际应该调用 LLM
    # llm_response = llm.invoke(f"解析以下需求:\n{requirement}")

    parsed = [
        {
            "id": "REQ-001",
            "description": requirement,
            "priority": "high",
            "estimated_complexity": "medium"
        }
    ]

    return {
        "parsed_requirements": parsed,
        "messages": ["需求解析完成"]
    }

def decompose_tasks(state: ProjectState) -> dict:
    """任务分解"""
    requirements = state["parsed_requirements"]

    # 简化版本:实际应该调用 LLM
    tasks: List[Task] = []

    for req in requirements:
        # 根据需求生成任务
        base_tasks = [
            {
                "id": f"TASK-{len(tasks) + 1:03d}",
                "name": f"设计 {req['description']}",
                "description": "完成设计文档",
                "estimated_hours": 8.0,
                "dependencies": [],
                "assigned_to": "",
                "status": "pending"
            },
            {
                "id": f"TASK-{len(tasks) + 2:03d}",
                "name": f"实现 {req['description']}",
                "description": "编码实现",
                "estimated_hours": 16.0,
                "dependencies": [f"TASK-{len(tasks) + 1:03d}"],
                "assigned_to": "",
                "status": "pending"
            },
            {
                "id": f"TASK-{len(tasks) + 3:03d}",
                "name": f"测试 {req['description']}",
                "description": "编写和执行测试",
                "estimated_hours": 8.0,
                "dependencies": [f"TASK-{len(tasks) + 2:03d}"],
                "assigned_to": "",
                "status": "pending"
            }
        ]

        tasks.extend(base_tasks)

    return {
        "tasks": tasks,
        "messages": [f"任务分解完成,共 {len(tasks)} 个任务"]
    }

def assess_risks(state: ProjectState) -> dict:
    """风险评估"""
    tasks = state["tasks"]

    risks: List[Risk] = []

    # 简单规则:超过 20 小时的任务标记为高风险
    total_hours = sum(t["estimated_hours"] for t in tasks)

    if total_hours > 40:
        risks.append({
            "level": "high",
            "description": f"项目总工时 {total_hours} 小时,可能超出预期",
            "mitigation": "建议分阶段交付"
        })
    elif total_hours > 20:
        risks.append({
            "level": "medium",
            "description": f"项目总工时 {total_hours} 小时,需要关注进度",
            "mitigation": "每周进行进度检查"
        })
    else:
        risks.append({
            "level": "low",
            "description": "项目规模适中",
            "mitigation": "按正常流程执行"
        })

    # 判断是否需要审批
    needs_approval = any(r["level"] == "high" for r in risks)

    return {
        "risks": risks,
        "needs_approval": needs_approval,
        "messages": [f"风险评估完成,风险等级: {risks[0]['level']}"]
    }

def allocate_resources(state: ProjectState) -> dict:
    """资源分配"""
    tasks = state["tasks"]
    team_members = state.get("team_members", [])

    # 如果没有团队成员,使用默认团队
    if not team_members:
        team_members = [
            {
                "name": "Alice",
                "skills": ["backend", "database"],
                "current_load": 0.0
            },
            {
                "name": "Bob",
                "skills": ["frontend", "ui"],
                "current_load": 0.0
            },
            {
                "name": "Charlie",
                "skills": ["testing", "qa"],
                "current_load": 0.0
            }
        ]

    # 简单的负载均衡分配
    allocated_tasks = []
    member_loads = {m["name"]: m["current_load"] for m in team_members}

    for task in tasks:
        # 找到负载最少的成员
        assigned_member = min(member_loads.items(), key=lambda x: x[1])[0]

        # 分配任务
        allocated_task = {**task, "assigned_to": assigned_member}
        allocated_tasks.append(allocated_task)

        # 更新负载
        member_loads[assigned_member] += task["estimated_hours"]

    # 检查是否需要重新分配
    max_load = max(member_loads.values())
    min_load = min(member_loads.values())
    needs_reallocation = (max_load - min_load) > 20  # 负载差超过 20 小时

    return {
        "tasks": allocated_tasks,
        "team_members": [
            {**m, "current_load": member_loads[m["name"]]}
            for m in team_members
        ],
        "needs_reallocation": needs_reallocation,
        "messages": ["资源分配完成"]
    }

def generate_plan(state: ProjectState) -> dict:
    """生成项目计划"""
    tasks = state["tasks"]
    risks = state["risks"]
    team_members = state["team_members"]

    plan_lines = ["# 项目计划\n"]

    # 需求概述
    plan_lines.append("## 需求")
    plan_lines.append(f"{state['user_requirement']}\n")

    # 任务列表
    plan_lines.append("## 任务列表\n")
    for task in tasks:
        deps = ", ".join(task["dependencies"]) if task["dependencies"] else "无"
        plan_lines.append(
            f"- **{task['name']}** "
            f"({task['estimated_hours']}h) "
            f"-> {task['assigned_to']} "
            f"[依赖: {deps}]"
        )

    # 团队负载
    plan_lines.append("\n## 团队负载\n")
    for member in team_members:
        plan_lines.append(f"- {member['name']}: {member['current_load']}h")

    # 风险
    plan_lines.append("\n## 风险评估\n")
    for risk in risks:
        plan_lines.append(
            f"- [{risk['level'].upper()}] {risk['description']}\n"
            f"  缓解措施: {risk['mitigation']}"
        )

    project_plan = "\n".join(plan_lines)

    return {
        "project_plan": project_plan,
        "messages": ["项目计划生成完成"]
    }

# ============= 条件判断 =============

def check_approval_needed(state: ProjectState) -> str:
    """检查是否需要审批"""
    if state["needs_approval"]:
        return "request_approval"
    return "allocate"

def check_approval_status(state: ProjectState) -> str:
    """检查审批状态"""
    status = state.get("approval_status", "")

    if status == "approved":
        return "continue"
    elif status == "rejected":
        return "end"
    else:
        # 等待审批
        return "wait"

def check_reallocation_needed(state: ProjectState) -> str:
    """检查是否需要重新分配"""
    if state.get("needs_reallocation", False):
        return "reallocate"
    return "generate"

# ============= 人机交互节点 =============

def request_approval(state: ProjectState) -> dict:
    """请求人工审批"""
    return {
        "approval_status": "pending",
        "messages": ["等待人工审批..."]
    }

# ============= 构建图 =============

def create_project_management_graph():
    """创建项目管理图"""
    builder = StateGraph(ProjectState)

    # 添加节点
    builder.add_node("parse", parse_requirements)
    builder.add_node("decompose", decompose_tasks)
    builder.add_node("assess_risks", assess_risks)
    builder.add_node("request_approval", request_approval)
    builder.add_node("allocate", allocate_resources)
    builder.add_node("generate", generate_plan)

    # 设置入口
    builder.set_entry_point("parse")

    # 添加边
    builder.add_edge("parse", "decompose")
    builder.add_edge("decompose", "assess_risks")

    # 条件分支:是否需要审批
    builder.add_conditional_edges(
        "assess_risks",
        check_approval_needed,
        {
            "request_approval": "request_approval",
            "allocate": "allocate"
        }
    )

    # 审批后继续
    builder.add_edge("request_approval", "allocate")

    # 条件分支:是否需要重新分配
    builder.add_conditional_edges(
        "allocate",
        check_reallocation_needed,
        {
            "reallocate": "allocate",  # 循环
            "generate": "generate"
        }
    )

    # 结束
    builder.set_finish_point("generate")

    # 添加持久化
    conn = sqlite3.connect(":memory:")  # 使用内存数据库
    checkpointer = SqliteSaver(conn)

    return builder.compile(checkpointer=checkpointer)

# ============= 使用示例 =============

def run_example():
    """运行示例"""
    graph = create_project_management_graph()

    # 初始输入
    initial_state = {
        "user_requirement": "开发一个用户认证系统,包括登录、注册、密码重置功能",
        "parsed_requirements": [],
        "tasks": [],
        "risks": [],
        "team_members": [],
        "needs_approval": False,
        "approval_status": "",
        "needs_reallocation": False,
        "project_plan": "",
        "messages": []
    }

    # 配置
    config = {"configurable": {"thread_id": "project-001"}}

    # 执行
    result = graph.invoke(initial_state, config=config)

    # 打印结果
    print("=== 执行日志 ===")
    for msg in result["messages"]:
        print(msg)

    print("\n=== 项目计划 ===")
    print(result["project_plan"])

    # 如果需要审批
    if result["needs_approval"] and result["approval_status"] == "pending":
        print("\n=== 需要人工审批 ===")
        print("请审批此项目计划(approve/reject):")

        # 模拟人工审批
        approval = "approve"  # 实际应该等待用户输入

        # 继续执行
        updated_state = {
            **result,
            "approval_status": "approved"
        }

        result = graph.invoke(updated_state, config=config)

        print("\n=== 审批后的计划 ===")
        print(result["project_plan"])

if __name__ == "__main__":
    run_example()
```

#### 11.4 功能增强

##### 11.4.1 集成真实的 LLM

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage

llm = ChatOpenAI(model="gpt-4", temperature=0)

def parse_requirements_with_llm(state: ProjectState) -> dict:
    """使用 LLM 解析需求"""
    requirement = state["user_requirement"]

    messages = [
        SystemMessage(content="""你是一个项目管理专家。
将用户需求解析为结构化的需求列表。

输出 JSON 格式:
[
  {
    "id": "REQ-001",
    "description": "需求描述",
    "priority": "high/medium/low",
    "estimated_complexity": "high/medium/low"
  }
]
"""),
        HumanMessage(content=requirement)
    ]

    response = llm.invoke(messages)

    # 解析 LLM 输出
    try:
        parsed = json.loads(response.content)
        return {
            "parsed_requirements": parsed,
            "messages": ["需求解析完成"]
        }
    except json.JSONDecodeError:
        return {
            "parsed_requirements": [],
            "messages": ["需求解析失败,请重新描述需求"]
        }

def decompose_tasks_with_llm(state: ProjectState) -> dict:
    """使用 LLM 分解任务"""
    requirements = state["parsed_requirements"]

    messages = [
        SystemMessage(content="""你是一个项目管理专家。
将需求分解为具体的任务,包括任务名称、描述、预计工时和依赖关系。

输出 JSON 格式:
[
  {
    "id": "TASK-001",
    "name": "任务名称",
    "description": "详细描述",
    "estimated_hours": 8.0,
    "dependencies": [],
    "assigned_to": "",
    "status": "pending"
  }
]
"""),
        HumanMessage(content=json.dumps(requirements, ensure_ascii=False))
    ]

    response = llm.invoke(messages)

    try:
        tasks = json.loads(response.content)
        return {
            "tasks": tasks,
            "messages": [f"任务分解完成,共 {len(tasks)} 个任务"]
        }
    except json.JSONDecodeError:
        return {
            "tasks": [],
            "messages": ["任务分解失败"]
        }
```

##### 11.4.2 添加重试机制

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
def parse_requirements_with_retry(state: ProjectState) -> dict:
    """带重试的需求解析"""
    return parse_requirements_with_llm(state)

def parse_requirements_robust(state: ProjectState) -> dict:
    """健壮的需求解析"""
    try:
        return parse_requirements_with_retry(state)
    except Exception as e:
        # 重试失败后的降级处理
        return {
            "parsed_requirements": [],
            "messages": [f"需求解析失败: {str(e)}"]
        }
```

##### 11.4.3 添加流式输出

```python
def run_with_streaming():
    """流式输出执行过程"""
    graph = create_project_management_graph()

    initial_state = {
        "user_requirement": "开发用户认证系统",
        "parsed_requirements": [],
        "tasks": [],
        "risks": [],
        "team_members": [],
        "needs_approval": False,
        "approval_status": "",
        "needs_reallocation": False,
        "project_plan": "",
        "messages": []
    }

    config = {"configurable": {"thread_id": "project-001"}}

    # 流式执行
    for event in graph.stream(initial_state, config=config):
        # event 是一个字典: {node_name: state_update}
        for node, state_update in event.items():
            print(f"\n[{node}]")

            # 打印日志消息
            if "messages" in state_update:
                for msg in state_update["messages"]:
                    print(f"  {msg}")
```

#### 11.5 测试

##### 11.5.1 单元测试

```python
import pytest

def test_parse_requirements():
    """测试需求解析"""
    state = {
        "user_requirement": "开发登录功能",
        "parsed_requirements": [],
        "messages": []
    }

    result = parse_requirements(state)

    assert len(result["parsed_requirements"]) > 0
    assert result["parsed_requirements"][0]["id"].startswith("REQ-")

def test_decompose_tasks():
    """测试任务分解"""
    state = {
        "parsed_requirements": [
            {
                "id": "REQ-001",
                "description": "登录功能",
                "priority": "high",
                "estimated_complexity": "medium"
            }
        ],
        "tasks": [],
        "messages": []
    }

    result = decompose_tasks(state)

    assert len(result["tasks"]) > 0
    assert all(t["id"].startswith("TASK-") for t in result["tasks"])

def test_assess_risks():
    """测试风险评估"""
    # 高风险场景
    state = {
        "tasks": [
            {"estimated_hours": 50.0}
        ],
        "risks": [],
        "messages": []
    }

    result = assess_risks(state)

    assert len(result["risks"]) > 0
    assert result["needs_approval"] == True

    # 低风险场景
    state["tasks"] = [{"estimated_hours": 10.0}]
    result = assess_risks(state)

    assert result["needs_approval"] == False
```

##### 11.5.2 集成测试

```python
def test_full_workflow():
    """测试完整工作流"""
    graph = create_project_management_graph()

    initial_state = {
        "user_requirement": "开发一个简单的待办事项应用",
        "parsed_requirements": [],
        "tasks": [],
        "risks": [],
        "team_members": [],
        "needs_approval": False,
        "approval_status": "",
        "needs_reallocation": False,
        "project_plan": "",
        "messages": []
    }

    config = {"configurable": {"thread_id": "test-001"}}

    result = graph.invoke(initial_state, config=config)

    # 验证输出
    assert result["project_plan"] != ""
    assert len(result["tasks"]) > 0
    assert len(result["risks"]) > 0
    assert all(t["assigned_to"] != "" for t in result["tasks"])
```

#### 11.6 部署建议

##### 11.6.1 生产环境配置

```python
import os
from langgraph.checkpoint.postgres import PostgresSaver
import psycopg

def create_production_graph():
    """创建生产环境的图"""
    # 使用 PostgreSQL
    conn = psycopg.connect(os.getenv("DATABASE_URL"))
    checkpointer = PostgresSaver(conn)

    builder = StateGraph(ProjectState)
    # ... 添加节点

    return builder.compile(
        checkpointer=checkpointer,
        # 启用调试
        debug=False
    )
```

##### 11.6.2 API 封装

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel

app = FastAPI()

class ProjectRequest(BaseModel):
    requirement: str
    thread_id: str

class ProjectResponse(BaseModel):
    project_plan: str
    status: str
    needs_approval: bool

@app.post("/projects", response_model=ProjectResponse)
async def create_project(request: ProjectRequest):
    """创建项目"""
    graph = create_production_graph()

    initial_state = {
        "user_requirement": request.requirement,
        "parsed_requirements": [],
        "tasks": [],
        "risks": [],
        "team_members": [],
        "needs_approval": False,
        "approval_status": "",
        "needs_reallocation": False,
        "project_plan": "",
        "messages": []
    }

    config = {"configurable": {"thread_id": request.thread_id}}

    result = graph.invoke(initial_state, config=config)

    return ProjectResponse(
        project_plan=result["project_plan"],
        status="completed",
        needs_approval=result["needs_approval"]
    )

@app.post("/projects/{thread_id}/approve")
async def approve_project(thread_id: str, approved: bool):
    """审批项目"""
    graph = create_production_graph()

    config = {"configurable": {"thread_id": thread_id}}

    # 获取当前状态
    current_state = graph.get_state(config)

    # 更新审批状态
    updated_state = {
        **current_state.values,
        "approval_status": "approved" if approved else "rejected"
    }

    # 继续执行
    result = graph.invoke(updated_state, config=config)

    return {"status": "success", "project_plan": result["project_plan"]}
```

**小结**

这个完整的实战案例展示了:
- ✅ 如何设计复杂的状态结构
- ✅ 如何组合多个节点实现业务逻辑
- ✅ 如何实现条件分支和循环
- ✅ 如何集成 LLM 和工具
- ✅ 如何处理人机交互
- ✅ 如何添加错误处理和重试
- ✅ 如何进行测试
- ✅ 如何部署到生产环境

通过这个案例,你应该能够独立构建复杂的 LangGraph 应用。

---
### 第 12 章:架构设计模式

#### 12.1 职责划分原则

**核心思想**

在构建复杂的 LangGraph 应用时,合理的职责划分是关键。

**单一职责原则(SRP)**

每个节点应该只做一件事情,且做好这件事情。

**反面案例**:

```python
def god_node(state: State) -> dict:
    """一个节点做所有事情"""
    # 调用 LLM
    llm_response = llm.invoke(state["input"])
    
    # 解析输出
    parsed = json.loads(llm_response.content)
    
    # 验证数据
    if not validate(parsed):
        raise ValueError("Invalid data")
    
    # 存储到数据库
    db.save(parsed)
    
    # 发送通知
    send_email(parsed)
    
    # 生成报告
    report = generate_report(parsed)
    
    return {"result": report}
```

问题:
- 职责不清晰
- 难以测试
- 难以复用
- 错误处理复杂

**正确案例**:

```python
def call_llm(state: State) -> dict:
    """职责:调用 LLM"""
    response = llm.invoke(state["input"])
    return {"llm_output": response.content}

def parse_output(state: State) -> dict:
    """职责:解析输出"""
    try:
        parsed = json.loads(state["llm_output"])
        return {"parsed_data": parsed, "parse_error": ""}
    except json.JSONDecodeError as e:
        return {"parsed_data": None, "parse_error": str(e)}

def validate_data(state: State) -> dict:
    """职责:验证数据"""
    data = state["parsed_data"]
    errors = []
    
    if not data:
        errors.append("数据为空")
    
    return {"validation_errors": errors}
```

优势:
- 每个节点职责单一
- 易于测试和复用
- 错误处理清晰
- 可以灵活组合

**关注点分离(Separation of Concerns)**

不同类型的逻辑应该分离到不同的节点或子图。

**层次划分**:

| 层次 | 职责 | 示例节点 |
|------|------|----------|
| **业务逻辑层** | 实现核心业务规则 | 需求分析、任务分解 |
| **数据处理层** | 数据转换、验证 | 解析、验证、格式化 |
| **集成层** | 与外部系统交互 | 调用 LLM、查询数据库、调用 API |
| **控制流层** | 决策和路由 | 条件判断、循环控制 |

#### 12.2 图结构模式

##### 12.2.1 模式1:线性流水线

**适用场景**: 简单的顺序处理流程

```
输入 → 步骤1 → 步骤2 → 步骤3 → 输出
```

**实现**:

```python
builder.set_entry_point("step1")
builder.add_edge("step1", "step2")
builder.add_edge("step2", "step3")
builder.set_finish_point("step3")
```

**优点**:
- 简单易懂
- 容易调试
- 执行路径确定

**缺点**:
- 缺乏灵活性
- 无法处理异常分支
- 无法迭代优化

##### 12.2.2 模式2:条件分支

**适用场景**: 根据条件选择不同的处理路径

```
输入 → 分类器 → [类型A] → 处理A → 输出
                ↓
              [类型B] → 处理B → 输出
                ↓
              [类型C] → 处理C → 输出
```

##### 12.2.3 模式3:循环迭代

**适用场景**: 需要反复优化的场景

```
输入 → 生成 → 评估 → [不满意] → 改进 → 生成
                 ↓
               [满意]
                 ↓
               输出
```

**最佳实践**:

```python
def should_continue_safe(state: State) -> str:
    """安全的循环判断"""
    # 1. 硬性限制迭代次数
    if state["iteration"] >= MAX_ITERATIONS:
        return "finish"
    
    # 2. 检查是否有改进
    if state["iteration"] > 0:
        improvement = state["current_score"] - state["previous_score"]
        if improvement < 0.01:  # 改进不明显
            return "finish"
    
    # 3. 达到质量要求
    if state["current_score"] >= TARGET_SCORE:
        return "finish"
    
    return "continue"
```

#### 12.3 状态设计模式

##### 12.3.1 模式1:最小化状态

**原则**: 只保留必要的状态,避免冗余

**正确案例**:

```python
class State(TypedDict):
    # 输入
    user_input: str
    
    # 关键中间状态
    current_stage: Literal["parsing", "processing", "finalizing"]
    intermediate_result: dict  # 只保留最新的中间结果
    
    # 输出
    final_result: str
    
    # 元数据
    metadata: dict  # 可选的调试信息
```

##### 12.3.2 模式2:结构化状态

**正确案例**:

```python
class User(TypedDict):
    name: str
    email: str
    phone: str

class Task(TypedDict):
    id: str
    name: str
    status: str

class State(TypedDict):
    user: User
    task: Task
    result: dict
```

#### 12.4 最佳实践总结

**图设计检查清单**:
- [ ] 是否画出了流程图?
- [ ] 每个节点是否只有一个职责?
- [ ] 是否有超过 10 个节点的图?(考虑拆分子图)
- [ ] 是否有死循环的风险?
- [ ] 是否为每个节点编写了文档?

**状态设计检查清单**:
- [ ] 状态是否使用了 TypedDict?
- [ ] 是否有不必要的冗余字段?
- [ ] 状态序列化后是否小于 1MB?
- [ ] 字段名是否清晰?

**小结**

架构设计模式的核心原则:
- ✅ 单一职责,关注点分离
- ✅ 根据场景选择合适的图结构模式
- ✅ 设计清晰、最小化的状态
- ✅ 使用检查清单避免常见错误

---

### 第 13 章:性能优化策略

#### 13.1 性能瓶颈分析

**常见瓶颈**

| 瓶颈类型 | 表现 | 占比 |
|---------|------|------|
| **LLM 调用延迟** | 单次调用 1-10 秒 | 60-80% |
| **网络 I/O** | API 调用、数据库查询 | 10-20% |
| **状态序列化** | 检查点保存/加载 | 5-10% |
| **计算密集** | 数据处理、解析 | 5-10% |

**性能测量**

```python
import time
from functools import wraps

def measure_time(func):
    """测量节点执行时间"""
    @wraps(func)
    def wrapper(state):
        start = time.time()
        result = func(state)
        elapsed = time.time() - start
        
        print(f"[PERF] {func.__name__}: {elapsed:.2f}s")
        
        return result
    
    return wrapper

@measure_time
def slow_node(state: State) -> dict:
    """慢节点"""
    result = expensive_operation(state["data"])
    return {"data": result}
```

#### 13.2 并行化优化

##### 13.2.1 使用并行节点

**并行版本(快)**:

```python
from langgraph.constants import Send

def fan_out(state: State):
    """启动并行任务"""
    return [
        Send("task_a", state),
        Send("task_b", state),
        Send("task_c", state)
    ]

# 构建图
builder.add_node("fan_out", fan_out)
builder.add_node("task_a", task_a)
builder.add_node("task_b", task_b)
builder.add_node("task_c", task_c)
builder.add_node("aggregate", aggregate_results)

builder.set_entry_point("fan_out")
builder.add_edge("task_a", "aggregate")
builder.add_edge("task_b", "aggregate")
builder.add_edge("task_c", "aggregate")
```

**性能提升**: 串行 6 秒 → 并行 2 秒 = 3x

##### 13.2.2 批量处理

**批量处理(快)**:

```python
def process_items_in_batch(state: State) -> dict:
    """批量处理"""
    items = state["items"]
    
    # 一次性发送所有项目
    batch_prompt = "批量处理以下项目:\n" + "\n".join(
        f"{i+1}. {item}" for i, item in enumerate(items)
    )
    
    result = llm.invoke(batch_prompt)
    results = parse_batch_result(result)
    
    return {"results": results}
```

**性能提升**: 逐个 10 秒 → 批量 2 秒 = 5x

#### 13.3 LLM 调用优化

##### 13.3.1 减少不必要的 LLM 调用

**策略1:使用缓存**

```python
from functools import lru_cache

@lru_cache(maxsize=100)
def call_llm_cached(prompt: str) -> str:
    """缓存 LLM 结果"""
    response = llm.invoke(prompt)
    return response.content
```

**策略2:条件调用**

```python
def conditional_llm_call(state: State) -> dict:
    """只在必要时调用 LLM"""
    # 先尝试规则引擎
    rule_result = apply_rules(state["input"])
    
    if rule_result.confidence > 0.9:
        return {"result": rule_result.value, "method": "rule"}
    
    # 规则不确定,使用 LLM
    llm_result = llm.invoke(state["input"])
    return {"result": llm_result.content, "method": "llm"}
```

##### 13.3.2 使用更快的模型

| 任务复杂度 | 推荐模型 | 速度 | 成本 |
|-----------|---------|------|------|
| 简单分类 | gpt-3.5-turbo | 快 | 低 |
| 一般任务 | gpt-4-turbo | 中 | 中 |
| 复杂推理 | gpt-4 | 慢 | 高 |

##### 13.3.3 优化 Prompt

```python
# ✅ 简洁的 Prompt
short_prompt = """
Analyze and summarize:

{text}
"""
```

**性能提升**:
- Token 数量减少 50%
- 响应时间减少 30%
- 成本降低 50%

#### 13.4 状态管理优化

##### 13.4.1 减少检查点频率

```python
class State(TypedDict):
    data: dict
    checkpoint_flag: bool  # 标记是否需要保存
```

##### 13.4.2 状态压缩

```python
def cleanup_state(state: State) -> dict:
    """清理状态中的临时数据"""
    cleaned_state = {
        k: v for k, v in state.items()
        if not k.startswith("_temp")
    }
    return cleaned_state
```

##### 13.4.3 选择合适的 Checkpointer

| Checkpointer | 写入速度 | 读取速度 | 适用场景 |
|-------------|---------|---------|---------|
| InMemorySaver | 极快 | 极快 | 开发/测试 |
| SqliteSaver | 快 | 快 | 单机应用 |
| PostgresSaver | 中 | 中 | 生产环境 |

#### 13.5 最佳实践

**性能优化检查清单**:

**并行化**:
- [ ] 是否识别了可以并行的任务?
- [ ] 是否使用了 Send API?

**LLM 优化**:
- [ ] 是否使用了合适的模型?
- [ ] 是否缓存了重复的调用?
- [ ] 是否优化了 Prompt 长度?
- [ ] 是否使用了批量处理?

**状态管理**:
- [ ] 状态是否最小化?
- [ ] 是否使用了合适的 Checkpointer?

**小结**

性能优化的核心策略:
- ✅ 测量优先:先测量,再优化
- ✅ 并行化:利用并行节点和异步执行
- ✅ LLM 优化:选择合适的模型,优化 Prompt
- ✅ 状态管理:最小化状态,控制检查点

记住:过早优化是万恶之源。先保证正确性,再优化性能。

---

### 第 14 章:错误处理与可靠性

#### 14.1 错误分类

**LangGraph 应用中的常见错误**

| 错误类型 | 原因 | 示例 | 严重程度 |
|---------|------|------|---------|
| **LLM 调用失败** | 网络问题、API 限流 | OpenAI API 超时 | 高 |
| **解析错误** | LLM 输出格式不符 | JSON 解析失败 | 中 |
| **数据验证失败** | 输入不符合要求 | 缺少必填字段 | 中 |
| **业务逻辑错误** | 违反业务规则 | 任务依赖循环 | 低 |
| **资源不足** | 内存、数据库连接 | 数据库连接池耗尽 | 高 |

#### 14.2 节点级错误处理

##### 14.2.1 基本错误捕获

```python
from typing import TypedDict, Optional

class State(TypedDict):
    data: dict
    error: Optional[str]
    error_node: Optional[str]

def safe_node(state: State) -> dict:
    """安全的节点实现"""
    try:
        result = risky_operation(state["data"])
        return {"data": result, "error": None}
    except ValueError as e:
        return {
            "error": f"数据验证失败: {str(e)}",
            "error_node": "safe_node"
        }
    except Exception as e:
        return {
            "error": f"未知错误: {str(e)}",
            "error_node": "safe_node"
        }
```

##### 14.2.2 重试机制

**策略1:使用 tenacity 库**

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type
)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type((TimeoutError, ConnectionError))
)
def call_external_api(data: dict) -> dict:
    """调用外部 API,自动重试"""
    response = requests.post("https://api.example.com", json=data)
    response.raise_for_status()
    return response.json()
```

##### 14.2.3 降级策略

```python
def node_with_fallback(state: State) -> dict:
    """带降级的节点"""
    try:
        # 首选:使用 LLM
        result = llm.invoke(state["prompt"])
        return {"result": result.content, "method": "llm"}
    except Exception as llm_error:
        try:
            # 降级:使用规则引擎
            result = rule_based_processor(state["prompt"])
            return {"result": result, "method": "rule"}
        except Exception:
            # 最终降级:返回默认值
            return {
                "result": "抱歉,暂时无法处理您的请求",
                "method": "default"
            }
```

#### 14.3 图级错误处理

##### 14.3.1 错误状态传播

```python
from typing import List

class Error(TypedDict):
    """错误详情"""
    node: str
    type: str
    message: str
    timestamp: str

class State(TypedDict):
    data: dict
    errors: Annotated[List[Error], "append"]
    has_fatal_error: bool

def error_router(state: State) -> str:
    """错误路由"""
    if state.get("has_fatal_error", False):
        return "handle_fatal_error"
    
    if len(state.get("errors", [])) > 0:
        return "handle_error"
    
    return "continue"
```

#### 14.4 LLM 调用的错误处理

##### 14.4.1 处理 API 限流

```python
from tenacity import retry, wait_random_exponential, stop_after_attempt

@retry(
    wait=wait_random_exponential(min=1, max=60),
    stop=stop_after_attempt(5)
)
def call_llm_with_rate_limit_handling(prompt: str) -> str:
    """处理限流的 LLM 调用"""
    try:
        response = llm.invoke(prompt)
        return response.content
    except RateLimitError as e:
        print(f"遇到限流,等待后重试: {e}")
        raise  # 触发重试
```

##### 14.4.2 处理解析错误

```python
from pydantic import BaseModel, ValidationError
import json

def parse_llm_output_safe(state: State) -> dict:
    """安全解析 LLM 输出"""
    llm_output = state["llm_output"]
    
    try:
        parsed = json.loads(llm_output)
        return {"tasks": parsed, "error": None}
    except json.JSONDecodeError as e:
        return {"tasks": [], "error": f"JSON 解析失败: {e}"}
    except ValidationError as e:
        return {"tasks": [], "error": f"数据验证失败: {e}"}
```

#### 14.5 可观测性

##### 14.5.1 结构化日志

```python
import logging
import json

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def instrumented_node(state: State) -> dict:
    """带日志的节点"""
    logger.info(f"开始执行 instrumented_node")
    
    try:
        result = process(state["data"])
        return {"data": result}
    except Exception as e:
        logger.error(f"节点执行失败: {e}", exc_info=True)
        raise
```

#### 14.6 最佳实践

**错误处理检查清单**:

**节点级**:
- [ ] 是否捕获了所有可能的异常?
- [ ] 是否实现了重试机制?
- [ ] 是否有降级方案?

**图级**:
- [ ] 是否在状态中传播错误信息?
- [ ] 是否有全局错误处理器?
- [ ] 是否记录了错误日志?

**可观测性**:
- [ ] 是否添加了结构化日志?
- [ ] 是否收集了性能指标?

**小结**

错误处理与可靠性的核心原则:
- ✅ 分层处理:节点级、图级、应用级
- ✅ 失败快速:不可恢复的错误快速失败
- ✅ 重试机制:处理瞬时错误
- ✅ 降级策略:保证基本功能可用
- ✅ 可观测性:日志、追踪、指标

记住:错误处理不是可选的,是必须的。

---
### 第 15 章:测试与调试

#### 15.1 测试策略

**测试金字塔**

```
       /\
      /  \  E2E Tests (少量)
     /────\
    /      \ Integration Tests (适量)
   /────────\
  /          \ Unit Tests (大量)
 /────────────\
```

**测试层次**

| 层次 | 目标 | 工具 | 比例 |
|------|------|------|------|
| **单元测试** | 测试单个节点 | pytest | 70% |
| **集成测试** | 测试子图或完整图 | pytest + mock | 20% |
| **端到端测试** | 测试完整工作流 | pytest + 真实依赖 | 10% |

#### 15.2 单元测试

##### 15.2.1 测试单个节点

**基本节点测试**

```python
import pytest

def test_parse_requirement_node():
    """测试需求解析节点"""
    # 准备输入
    state = {
        "user_requirement": "开发登录功能",
        "parsed_requirements": [],
        "messages": []
    }
    
    # 执行节点
    result = parse_requirements(state)
    
    # 验证输出
    assert "parsed_requirements" in result
    assert len(result["parsed_requirements"]) > 0
    assert result["parsed_requirements"][0]["id"].startswith("REQ-")

def test_parse_requirement_node_empty_input():
    """测试空输入"""
    state = {
        "user_requirement": "",
        "parsed_requirements": [],
        "messages": []
    }
    
    result = parse_requirements(state)
    
    # 验证错误处理
    assert len(result["parsed_requirements"]) == 0
    # 或者抛出异常
```

**带错误处理的节点测试**

```python
def test_risky_node_success():
    """测试成功场景"""
    state = {"data": {"value": 10}}
    
    result = risky_node(state)
    
    assert result["error"] is None
    assert "data" in result

def test_risky_node_failure():
    """测试失败场景"""
    state = {"data": {"value": -1}}  # 无效输入
    
    result = risky_node(state)
    
    assert result["error"] is not None
    assert "error_node" in result
```

**使用参数化测试**

```python
@pytest.mark.parametrize("input_value,expected_output", [
    ("简单需求", 1),
    ("复杂需求,包含多个功能", 2),
    ("", 0),
])
def test_parse_requirements_parametrized(input_value, expected_output):
    """参数化测试"""
    state = {
        "user_requirement": input_value,
        "parsed_requirements": [],
        "messages": []
    }
    
    result = parse_requirements(state)
    
    assert len(result["parsed_requirements"]) == expected_output
```

##### 15.2.2 测试条件函数

```python
def test_should_continue_finish():
    """测试终止条件"""
    state = {
        "iteration": 5,
        "quality_score": 0.95
    }
    
    result = should_continue(state)
    
    assert result == "finish"

def test_should_continue_continue():
    """测试继续条件"""
    state = {
        "iteration": 1,
        "quality_score": 0.5
    }
    
    result = should_continue(state)
    
    assert result == "improve"
```

##### 15.2.3 使用 Fixtures

```python
@pytest.fixture
def sample_state():
    """创建示例状态"""
    return {
        "user_input": "测试输入",
        "data": {},
        "messages": []
    }

@pytest.fixture
def mock_llm(monkeypatch):
    """Mock LLM"""
    def mock_invoke(prompt):
        class MockResponse:
            content = '{"tasks": ["task1", "task2"]}'
        return MockResponse()
    
    monkeypatch.setattr("module.llm.invoke", mock_invoke)

def test_node_with_mock_llm(sample_state, mock_llm):
    """使用 fixture 的测试"""
    result = llm_node(sample_state)
    
    assert "tasks" in result
```

#### 15.3 集成测试

##### 15.3.1 测试子图

```python
def test_validation_subgraph():
    """测试验证子图"""
    graph = create_validation_subgraph()
    
    # 测试有效输入
    result = graph.invoke({
        "content": "这是一段有效的内容",
        "is_valid": False,
        "errors": []
    })
    
    assert result["is_valid"] == True
    assert len(result["errors"]) == 0
    
    # 测试无效输入
    result = graph.invoke({
        "content": "",
        "is_valid": False,
        "errors": []
    })
    
    assert result["is_valid"] == False
    assert len(result["errors"]) > 0
```

##### 15.3.2 测试完整图

```python
def test_full_graph_happy_path():
    """测试完整图的正常流程"""
    graph = create_project_management_graph()
    
    initial_state = {
        "user_requirement": "开发一个简单功能",
        "parsed_requirements": [],
        "tasks": [],
        "risks": [],
        "team_members": [],
        "needs_approval": False,
        "approval_status": "",
        "needs_reallocation": False,
        "project_plan": "",
        "messages": []
    }
    
    config = {"configurable": {"thread_id": "test-001"}}
    
    result = graph.invoke(initial_state, config=config)
    
    # 验证最终状态
    assert result["project_plan"] != ""
    assert len(result["tasks"]) > 0
    assert all(t["assigned_to"] != "" for t in result["tasks"])

def test_full_graph_with_approval():
    """测试需要审批的流程"""
    graph = create_project_management_graph()
    
    # 构造高风险场景
    initial_state = {
        "user_requirement": "开发一个超大型系统",  # 会触发高风险
        # ... 其他字段
    }
    
    config = {"configurable": {"thread_id": "test-002"}}
    
    # 第一次执行:到审批点
    result = graph.invoke(initial_state, config=config)
    
    assert result["needs_approval"] == True
    assert result["approval_status"] == "pending"
    
    # 模拟人工审批
    updated_state = {**result, "approval_status": "approved"}
    
    # 第二次执行:审批后继续
    final_result = graph.invoke(updated_state, config=config)
    
    assert final_result["project_plan"] != ""
```

##### 15.3.3 使用 Mock 依赖

```python
from unittest.mock import Mock, patch

def test_graph_with_mock_llm():
    """使用 Mock LLM 测试图"""
    
    # Mock LLM 响应
    mock_llm = Mock()
    mock_llm.invoke.return_value = Mock(
        content='{"requirements": [{"id": "REQ-001", "description": "test"}]}'
    )
    
    with patch('module.llm', mock_llm):
        graph = create_project_management_graph()
        
        result = graph.invoke(initial_state, config=config)
        
        # 验证 LLM 被调用
        assert mock_llm.invoke.called
        
        # 验证结果
        assert len(result["parsed_requirements"]) > 0

def test_graph_with_mock_db():
    """使用 Mock 数据库测试图"""
    
    mock_db = Mock()
    mock_db.query.return_value = [
        {"name": "Alice", "skills": ["backend"]},
        {"name": "Bob", "skills": ["frontend"]}
    ]
    
    with patch('module.db', mock_db):
        graph = create_project_management_graph()
        
        result = graph.invoke(initial_state, config=config)
        
        # 验证数据库被查询
        assert mock_db.query.called
```

#### 15.4 端到端测试

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver

def test_e2e_with_real_dependencies():
    """端到端测试,使用真实依赖"""
    
    # 使用真实的 checkpointer
    conn = sqlite3.connect(":memory:")
    checkpointer = SqliteSaver(conn)
    
    graph = create_project_management_graph_with_checkpointer(checkpointer)
    
    # 执行完整流程
    initial_state = {
        "user_requirement": "开发用户认证系统",
        # ... 所有字段
    }
    
    config = {"configurable": {"thread_id": "e2e-test-001"}}
    
    # 执行
    result = graph.invoke(initial_state, config=config)
    
    # 验证结果
    assert result["project_plan"] != ""
    assert len(result["tasks"]) > 0
    
    # 验证持久化
    saved_state = graph.get_state(config)
    assert saved_state.values["project_plan"] == result["project_plan"]
```

#### 15.5 调试技巧

##### 15.5.1 使用流式输出调试

```python
def debug_with_streaming():
    """使用流式输出调试"""
    graph = create_graph()
    
    print("=== 执行流程 ===")
    
    for event in graph.stream(initial_state, config=config):
        # event 是 {node_name: state_update}
        for node_name, state_update in event.items():
            print(f"\n[执行节点: {node_name}]")
            print(f"状态更新: {state_update.keys()}")
            
            # 打印关键信息
            if "error" in state_update and state_update["error"]:
                print(f"❌ 错误: {state_update['error']}")
            
            if "messages" in state_update:
                for msg in state_update["messages"]:
                    print(f"  📝 {msg}")
```

##### 15.5.2 检查状态历史

```python
def debug_with_state_history():
    """检查状态历史"""
    graph = create_graph()
    
    config = {"configurable": {"thread_id": "debug-001"}}
    
    # 执行
    result = graph.invoke(initial_state, config=config)
    
    # 查看历史
    print("\n=== 状态历史 ===")
    history = graph.get_state_history(config)
    
    for i, checkpoint in enumerate(history):
        print(f"\n检查点 {i}:")
        print(f"  节点: {checkpoint.metadata.get('source', 'unknown')}")
        print(f"  时间: {checkpoint.metadata.get('ts', 'unknown')}")
        print(f"  状态: {checkpoint.values.keys()}")
```

##### 15.5.3 使用调试节点

```python
def create_debug_node(label: str):
    """创建调试节点"""
    def debug_node(state: State) -> dict:
        print(f"\n=== DEBUG: {label} ===")
        print(f"当前状态:")
        for key, value in state.items():
            if key.startswith("_"):
                continue  # 跳过私有字段
            
            if isinstance(value, (str, int, float, bool)):
                print(f"  {key}: {value}")
            elif isinstance(value, list):
                print(f"  {key}: [{len(value)} items]")
            else:
                print(f"  {key}: {type(value).__name__}")
        
        return {}  # 不修改状态
    
    return debug_node

# 在图中插入调试节点
builder.add_node("debug_after_parse", create_debug_node("解析后"))
builder.add_edge("parse", "debug_after_parse")
builder.add_edge("debug_after_parse", "decompose")
```

##### 15.5.4 断点调试

```python
import pdb

def node_with_breakpoint(state: State) -> dict:
    """带断点的节点"""
    
    # 在特定条件下设置断点
    if state.get("debug_mode", False):
        pdb.set_trace()
    
    result = process(state["data"])
    
    return {"data": result}
```

#### 15.6 常见问题排查

##### 15.6.1 问题:状态更新不生效

**症状**: 节点返回了更新,但状态没有变化

**原因**: 使用了错误的 reducer 或返回了空字典

**排查步骤**:

```python
def test_state_update():
    """测试状态更新"""
    
    # 检查节点返回值
    state = {"data": "old"}
    result = my_node(state)
    
    print(f"节点返回: {result}")
    
    # 验证返回值不为空
    assert result != {}
    assert "data" in result
    
    # 验证返回值与输入不同
    assert result["data"] != state["data"]
```

**解决方案**:

```python
# ❌ 错误:返回空字典
def bad_node(state: State) -> dict:
    state["data"] = "new"  # 直接修改状态
    return {}  # 返回空字典

# ✅ 正确:返回更新
def good_node(state: State) -> dict:
    return {"data": "new"}
```

##### 15.6.2 问题:图执行卡住

**症状**: 图执行到某个节点后不继续

**原因**: 
1. 条件边的条件函数返回了未定义的边
2. 节点抛出了未捕获的异常
3. 在等待人工输入

**排查步骤**:

```python
def debug_stuck_graph():
    """调试卡住的图"""
    graph = create_graph()
    
    # 使用流式输出查看执行到哪里
    for event in graph.stream(initial_state, config=config):
        node_name = list(event.keys())[0]
        print(f"执行: {node_name}")
    
    # 如果没有打印,说明在第一个节点就卡住了
```

**解决方案**:

```python
# 检查条件函数
def check_condition_function():
    """测试条件函数"""
    state = {"status": "unknown"}
    
    result = route_by_status(state)
    
    # 验证返回值是有效的边名称
    assert result in ["continue", "error", "end"]

# 添加默认边
builder.add_conditional_edges(
    "router",
    route_by_status,
    {
        "continue": "next_node",
        "error": "error_handler"
    }
    # ❌ 缺少 "end" 的映射
)

# ✅ 正确:添加所有可能的映射
builder.add_conditional_edges(
    "router",
    route_by_status,
    {
        "continue": "next_node",
        "error": "error_handler",
        "end": END
    }
)
```

##### 15.6.3 问题:循环次数超出预期

**症状**: 循环节点执行了太多次

**原因**: 终止条件不正确

**排查步骤**:

```python
def test_loop_termination():
    """测试循环终止"""
    state = {
        "iteration": 0,
        "quality_score": 0.5
    }
    
    max_iterations = 10
    
    for i in range(max_iterations):
        decision = should_continue(state)
        
        print(f"迭代 {i}: decision={decision}")
        
        if decision == "finish":
            print(f"在第 {i} 次迭代终止")
            break
        
        # 模拟状态更新
        state["iteration"] += 1
        state["quality_score"] += 0.1
    else:
        print(f"警告: 达到最大迭代次数 {max_iterations}")
```

**解决方案**:

```python
def should_continue_safe(state: State) -> str:
    """安全的循环条件"""
    # 1. 硬性限制
    if state["iteration"] >= 10:
        return "finish"
    
    # 2. 质量检查
    if state["quality_score"] >= 0.9:
        return "finish"
    
    # 3. 改进检查
    if state["iteration"] > 0:
        improvement = state["current_score"] - state["previous_score"]
        if improvement < 0.01:
            return "finish"
    
    return "continue"
```

##### 15.6.4 问题:LLM 输出解析失败

**症状**: JSON 解析总是失败

**原因**: LLM 输出格式不稳定

**排查步骤**:

```python
def debug_llm_output():
    """调试 LLM 输出"""
    state = {"prompt": "生成任务列表"}
    
    # 多次调用,查看输出稳定性
    for i in range(5):
        response = llm.invoke(state["prompt"])
        print(f"\n尝试 {i+1}:")
        print(response.content)
        
        try:
            parsed = json.loads(response.content)
            print("✅ 解析成功")
        except json.JSONDecodeError as e:
            print(f"❌ 解析失败: {e}")
```

**解决方案**:

```python
# 1. 改进 Prompt
prompt = """
生成任务列表,**必须**输出有效的 JSON 格式:

```json
{
  "tasks": [
    {"id": "1", "name": "任务1"},
    {"id": "2", "name": "任务2"}
  ]
}
```

不要输出任何其他内容。
"""

# 2. 使用结构化输出
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser()

prompt_with_instructions = f"""
{task_description}

{parser.get_format_instructions()}
"""

# 3. 清理 LLM 输出
def clean_llm_output(output: str) -> str:
    """清理 LLM 输出"""
    # 移除 markdown 代码块
    output = output.strip()
    if output.startswith("```json"):
        output = output[7:]
    if output.startswith("```"):
        output = output[3:]
    if output.endswith("```"):
        output = output[:-3]
    
    return output.strip()
```

#### 15.7 测试最佳实践

**测试检查清单**:

**单元测试**:
- [ ] 每个节点都有测试?
- [ ] 测试了成功和失败场景?
- [ ] 使用了参数化测试?
- [ ] 测试了边界条件?

**集成测试**:
- [ ] 测试了子图?
- [ ] 测试了完整图?
- [ ] 使用了 Mock 依赖?
- [ ] 测试了人机交互流程?

**调试**:
- [ ] 使用了流式输出?
- [ ] 检查了状态历史?
- [ ] 添加了调试日志?

**持续集成**:
- [ ] 测试在 CI 中自动运行?
- [ ] 测试覆盖率达标?(建议 >80%)
- [ ] 有性能基准测试?

**小结**

测试与调试的核心原则:
- ✅ 测试金字塔:大量单元测试,适量集成测试,少量 E2E 测试
- ✅ Mock 依赖:隔离测试,提高速度
- ✅ 流式调试:查看执行流程
- ✅ 状态历史:回溯问题
- ✅ 结构化日志:记录关键信息

记住:好的测试是最好的文档。

---

## 结语

### 学习路径建议

**初学者(第 1-2 周)**:
1. 阅读第 1-3 章,理解核心概念
2. 跟随第 4 章快速上手,运行示例代码
3. 完成第 5 章的贯穿案例

**进阶者(第 3-4 周)**:
4. 深入学习第 6-9 章的高级特性
5. 理解第 10 章的模块化设计
6. 参考第 11 章实现自己的项目

**专家级(持续学习)**:
7. 掌握第 12-15 章的最佳实践
8. 优化性能和可靠性
9. 贡献开源,分享经验

### 进一步学习资源

**官方资源**:
- [LangGraph 官方文档](https://langchain-ai.github.io/langgraph/)
- [LangGraph GitHub](https://github.com/langchain-ai/langgraph)
- [LangChain 文档](https://python.langchain.com/)

**社区资源**:
- [LangChain Discord](https://discord.gg/langchain)
- [LangChain Twitter](https://twitter.com/langchainai)
- [LangGraph 示例库](https://github.com/langchain-ai/langgraph/tree/main/examples)

**相关技术**:
- Prompt Engineering
- RAG (Retrieval-Augmented Generation)
- Fine-tuning LLMs
- Vector Databases

### 常见问题 FAQ

**Q: LangGraph 和 LangChain 的关系?**

A: LangGraph 是 LangChain 生态系统的一部分,专注于构建有状态的、循环的 Agent 工作流。LangChain 提供了更高层的抽象(如 Chain、Agent),而 LangGraph 提供了更底层的控制。

**Q: 什么时候应该使用 LangGraph?**

A: 当你需要:
- 复杂的条件分支
- 循环和迭代
- 人机交互
- 长时间运行的工作流
- 跨会话的状态持久化

**Q: LangGraph 的性能如何?**

A: 性能主要取决于:
- LLM 调用次数(最大瓶颈)
- 状态大小(影响序列化)
- Checkpointer 选择(影响持久化速度)

通过并行化、缓存、批量处理等优化可以显著提升性能。

**Q: 如何处理成本问题?**

A: 
- 减少不必要的 LLM 调用
- 使用更便宜的模型(如 gpt-3.5-turbo)
- 优化 Prompt 长度
- 使用缓存
- 添加规则引擎降级

**Q: LangGraph 适合生产环境吗?**

A: 是的。但需要:
- 使用 PostgresSaver 持久化
- 添加完善的错误处理
- 实现监控和告警
- 进行充分的测试
- 考虑成本和性能

### 致谢

感谢 LangChain 团队创建了如此优秀的框架。

感谢所有为本指南提供反馈和建议的读者。

希望这份指南能帮助你构建出色的 AI Agent 应用!

---

**版本**: v1.0
**最后更新**: 2026-02
**作者**: Claude Code
**许可**: MIT

