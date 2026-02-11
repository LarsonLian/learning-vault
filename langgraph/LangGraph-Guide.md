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
