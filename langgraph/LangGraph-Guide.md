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
