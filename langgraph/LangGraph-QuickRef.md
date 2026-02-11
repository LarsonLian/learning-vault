# LangGraph 快速参考

> 关键概念速查、常用代码模式、最佳实践清单

---

## 核心概念速查

### 1. 基础概念

| 概念 | 定义 | 关键点 |
|------|------|--------|
| **StateGraph** | 有向状态图,LangGraph 的核心抽象 | 节点 + 边 + 状态 |
| **Node** | 执行具体任务的函数 | 接收状态,返回更新 |
| **Edge** | 连接节点的路径 | 普通边、条件边、循环边 |
| **State** | 在节点间流转的数据结构 | TypedDict 定义,支持 Reducer |
| **Checkpointer** | 状态持久化机制 | 断点续传、时间旅行 |

### 2. 状态管理

**State 定义模式**

```python
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

# 基础状态
class BasicState(TypedDict):
    messages: list[str]
    current_step: str

# 带消息历史的状态(推荐)
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # 自动追加消息
    context: dict
    iteration_count: int
```

**Reducer 函数**

```python
# 自定义 Reducer
def merge_lists(existing: list, new: list) -> list:
    return existing + new

class State(TypedDict):
    items: Annotated[list, merge_lists]
```

### 3. 节点定义

**标准节点函数签名**

```python
def node_function(state: State) -> dict:
    # 1. 读取状态
    current_value = state["key"]

    # 2. 执行逻辑
    result = process(current_value)

    # 3. 返回更新(只返回要更新的字段)
    return {"key": result, "step": "completed"}
```

**LLM 调用节点**

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4")

def llm_node(state: State):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}
```

**工具调用节点**

```python
from langchain_core.tools import tool

@tool
def search_tool(query: str) -> str:
    """搜索工具"""
    return f"Results for: {query}"

def tool_node(state: State):
    # 从 LLM 响应中提取工具调用
    last_message = state["messages"][-1]
    tool_calls = last_message.tool_calls

    # 执行工具
    results = [search_tool.invoke(call) for call in tool_calls]
    return {"messages": results}
```

### 4. 边的类型

**普通边(固定路由)**

```python
graph.add_edge("node_a", "node_b")  # A 总是跳转到 B
graph.add_edge("node_b", END)       # B 结束流程
```

**条件边(动态路由)**

```python
def router(state: State) -> str:
    if state["score"] > 0.8:
        return "success"
    elif state["retry_count"] < 3:
        return "retry"
    else:
        return "fail"

graph.add_conditional_edge(
    "evaluate",                    # 源节点
    router,                        # 路由函数
    {                              # 路由映射
        "success": "output",
        "retry": "generate",
        "fail": END
    }
)
```

**循环边**

```python
# 方式1: 条件边返回源节点
def should_continue(state: State) -> str:
    if state["iteration"] < 5:
        return "generate"  # 循环回 generate 节点
    return END

graph.add_conditional_edge("evaluate", should_continue)

# 方式2: 普通边直接连回
graph.add_edge("improve", "generate")
```

### 5. 图的构建

**标准构建流程**

```python
from langgraph.graph import StateGraph, START, END

# 1. 定义状态
class MyState(TypedDict):
    messages: list
    result: str

# 2. 创建图
graph = StateGraph(MyState)

# 3. 添加节点
graph.add_node("step1", step1_function)
graph.add_node("step2", step2_function)

# 4. 添加边
graph.add_edge(START, "step1")
graph.add_edge("step1", "step2")
graph.add_edge("step2", END)

# 5. 编译
app = graph.compile()

# 6. 运行
result = app.invoke({"messages": []})
```

### 6. Checkpointer 配置

**内存存储(开发/测试)**

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)

# 带 thread_id 运行
config = {"configurable": {"thread_id": "user-123"}}
app.invoke(initial_state, config)
```

**持久化存储(生产)**

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/db"
)
app = graph.compile(checkpointer=checkpointer)
```

### 7. 人机协作

**Interrupt 模式**

```python
# 1. 编译时启用 interrupt_before
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["human_approval"]
)

# 2. 运行到 interrupt 点
config = {"configurable": {"thread_id": "123"}}
result = app.invoke(input_state, config)

# 3. 人工处理后继续
updated_state = {"approval": True}
final_result = app.invoke(updated_state, config)
```

**手动控制流**

```python
# 使用 stream 逐步执行
for step in app.stream(input_state, config):
    print(step)
    if should_pause(step):
        user_input = get_user_input()
        # 继续执行
```

---

## 常用代码模式

### 模式 1: 生成-评估-改进循环

```python
class State(TypedDict):
    prompt: str
    draft: str
    score: float
    iteration: int

def generate(state: State) -> dict:
    draft = llm.invoke(state["prompt"])
    return {
        "draft": draft.content,
        "iteration": state.get("iteration", 0) + 1
    }

def evaluate(state: State) -> dict:
    score = evaluate_quality(state["draft"])
    return {"score": score}

def should_continue(state: State) -> str:
    if state["score"] > 0.8 or state["iteration"] >= 3:
        return END
    return "generate"

# 构建图
graph = StateGraph(State)
graph.add_node("generate", generate)
graph.add_node("evaluate", evaluate)
graph.add_edge(START, "generate")
graph.add_edge("generate", "evaluate")
graph.add_conditional_edge("evaluate", should_continue)
```

### 模式 2: 并行执行 + 聚合

```python
from langgraph.graph import END, START

def parallel_task_1(state: State) -> dict:
    result = process_1(state["input"])
    return {"result_1": result}

def parallel_task_2(state: State) -> dict:
    result = process_2(state["input"])
    return {"result_2": result}

def aggregate(state: State) -> dict:
    combined = combine(state["result_1"], state["result_2"])
    return {"final_result": combined}

graph = StateGraph(State)
graph.add_node("task1", parallel_task_1)
graph.add_node("task2", parallel_task_2)
graph.add_node("aggregate", aggregate)

# START 到多个节点表示并行
graph.add_edge(START, "task1")
graph.add_edge(START, "task2")

# 汇聚到 aggregate
graph.add_edge("task1", "aggregate")
graph.add_edge("task2", "aggregate")
graph.add_edge("aggregate", END)
```

### 模式 3: 意图路由

```python
def classify_intent(state: State) -> dict:
    intent = llm.invoke(f"Classify intent: {state['query']}")
    return {"intent": intent.content}

def route_by_intent(state: State) -> str:
    intent_map = {
        "search": "search_handler",
        "create": "create_handler",
        "update": "update_handler"
    }
    return intent_map.get(state["intent"], "fallback")

graph.add_node("classify", classify_intent)
graph.add_node("search_handler", search_handler)
graph.add_node("create_handler", create_handler)
graph.add_node("update_handler", update_handler)
graph.add_node("fallback", fallback_handler)

graph.add_edge(START, "classify")
graph.add_conditional_edge(
    "classify",
    route_by_intent,
    {
        "search_handler": "search_handler",
        "create_handler": "create_handler",
        "update_handler": "update_handler",
        "fallback": "fallback"
    }
)
```

### 模式 4: 工具调用循环

```python
from langchain_core.messages import ToolMessage

def agent_node(state: State):
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

def tool_node(state: State):
    tool_calls = state["messages"][-1].tool_calls
    results = []
    for call in tool_calls:
        result = tools[call["name"]].invoke(call["args"])
        results.append(ToolMessage(content=result, tool_call_id=call["id"]))
    return {"messages": results}

def should_continue(state: State) -> str:
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return END

graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.add_edge(START, "agent")
graph.add_conditional_edge("agent", should_continue)
graph.add_edge("tools", "agent")  # 循环回 agent
```

### 模式 5: 子图封装

```python
# 子图定义
def create_subgraph():
    subgraph = StateGraph(SubState)
    subgraph.add_node("step1", step1)
    subgraph.add_node("step2", step2)
    subgraph.add_edge(START, "step1")
    subgraph.add_edge("step1", "step2")
    subgraph.add_edge("step2", END)
    return subgraph.compile()

# 在主图中使用
main_graph = StateGraph(MainState)
main_graph.add_node("preprocess", preprocess)
main_graph.add_node("subflow", create_subgraph())
main_graph.add_node("postprocess", postprocess)

main_graph.add_edge(START, "preprocess")
main_graph.add_edge("preprocess", "subflow")
main_graph.add_edge("subflow", "postprocess")
main_graph.add_edge("postprocess", END)
```

---

## 最佳实践清单

### 状态设计

- [ ] **最小化状态**: 只在状态中存储必要信息
- [ ] **使用 TypedDict**: 获得类型提示和 IDE 支持
- [ ] **选择合适的 Reducer**: 消息用 `add_messages`,列表用自定义 Reducer
- [ ] **避免嵌套过深**: 扁平化状态结构,最多 2-3 层
- [ ] **分离临时和持久数据**: 临时计算结果不需要在状态中持久化

### 节点设计

- [ ] **单一职责**: 每个节点只做一件事
- [ ] **纯函数**: 避免在节点中修改全局状态
- [ ] **明确返回**: 只返回需要更新的字段
- [ ] **错误处理**: 在节点内捕获异常,转换为状态更新
- [ ] **可测试性**: 节点函数应易于单元测试

### 图结构

- [ ] **控制复杂度**: 节点数量控制在 10-15 个以内
- [ ] **避免深度嵌套**: 图的层次不超过 3 层
- [ ] **明确起止点**: 使用 START 和 END 常量
- [ ] **循环保护**: 设置最大迭代次数,避免无限循环
- [ ] **并行优化**: 无依赖的操作使用并行执行

### 错误处理

- [ ] **分类错误**: 区分可恢复和不可恢复错误
- [ ] **优雅降级**: 提供 fallback 路径
- [ ] **重试机制**: 对临时性错误自动重试
- [ ] **记录错误**: 在状态中记录错误信息和上下文
- [ ] **避免静默失败**: 确保错误被妥善处理

### 性能优化

- [ ] **惰性加载**: 延迟加载大型模型或数据
- [ ] **批处理**: 合并多个小请求
- [ ] **缓存结果**: 对幂等操作缓存结果
- [ ] **流式输出**: 使用 `stream()` 提供即时反馈
- [ ] **监控指标**: 记录执行时间、Token 使用量

### 可观测性

- [ ] **结构化日志**: 使用 JSON 格式记录关键事件
- [ ] **追踪 ID**: 每个执行使用唯一 thread_id
- [ ] **状态快照**: 利用 Checkpointer 保存状态历史
- [ ] **可视化**: 使用 `graph.get_graph().draw_png()` 查看图结构
- [ ] **测试覆盖**: 单元测试 + 集成测试

---

## 常见问题 Q&A

### Q1: 什么时候使用 LangGraph 而不是简单 Chain?

**A:** 当你需要以下任一特性时:
- **循环/迭代**: 生成-评估-改进流程
- **条件分支**: 根据 LLM 输出动态路由
- **人机协作**: 需要人工审批或中断
- **长时间运行**: 跨多个会话的工作流
- **复杂状态**: 需要管理对话历史、上下文等

简单的单向流程使用 Chain 即可。

### Q2: State 应该包含什么?

**A:** 遵循以下原则:
- **必须**: 节点间共享的数据(消息历史、上下文、中间结果)
- **必须**: 控制流所需的信息(计数器、标志位)
- **不必**: 可以从其他字段计算出的数据
- **不必**: 仅在单个节点内使用的临时变量

### Q3: 如何避免无限循环?

**A:** 三种策略:
1. **计数器**: 在状态中记录迭代次数,设置上限
2. **超时**: 使用 `recursion_limit` 参数
3. **终止条件**: 在路由函数中明确终止条件

```python
# 示例
def should_continue(state: State) -> str:
    if state["iteration"] >= 5:  # 策略1
        return END
    if state["quality"] > 0.9:   # 策略3
        return END
    return "generate"

app = graph.compile(checkpointer=checkpointer)
app.invoke(input, config={"recursion_limit": 10})  # 策略2
```

### Q4: Checkpointer 会影响性能吗?

**A:** 会有影响,但可优化:
- **开发**: 使用 `MemorySaver`,性能影响小
- **生产**: 使用 PostgreSQL/Redis,考虑:
  - 异步写入
  - 批量保存
  - 只保存关键节点的状态
  - 设置过期时间清理旧数据

### Q5: 如何调试 LangGraph 应用?

**A:** 多种方法:
1. **可视化图结构**: `graph.get_graph().draw_png()`
2. **打印状态**: 在节点中添加 `print(state)`
3. **使用 stream**: 逐步观察执行过程
4. **查看历史**: `app.get_state_history(config)` 查看状态演进
5. **单元测试**: 单独测试每个节点函数

### Q6: 并行执行的节点如何共享数据?

**A:** 通过状态 + Reducer:
```python
def merge_results(existing: dict, new: dict) -> dict:
    return {**existing, **new}

class State(TypedDict):
    results: Annotated[dict, merge_results]

# 并行节点各自更新 results
def task1(state): return {"results": {"a": 1}}
def task2(state): return {"results": {"b": 2}}

# 聚合节点读取合并后的结果
def aggregate(state):
    print(state["results"])  # {"a": 1, "b": 2}
```

### Q7: 如何实现超时控制?

**A:** 在节点中使用超时装饰器:
```python
from functools import wraps
import signal

def timeout(seconds):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            def handler(signum, frame):
                raise TimeoutError()
            signal.signal(signal.SIGALRM, handler)
            signal.alarm(seconds)
            try:
                return func(*args, **kwargs)
            finally:
                signal.alarm(0)
        return wrapper
    return decorator

@timeout(30)
def slow_node(state):
    # 最多执行 30 秒
    result = expensive_operation()
    return {"result": result}
```

### Q8: 子图和普通节点有什么区别?

**A:**
| 维度 | 普通节点 | 子图 |
|------|---------|------|
| **复杂度** | 单个函数 | 完整的图结构 |
| **复用性** | 需手动封装 | 可直接复用 |
| **状态** | 共享主图状态 | 可有独立状态 |
| **测试** | 单元测试 | 可独立测试 |
| **适用场景** | 简单逻辑 | 复杂子流程 |

### Q9: 如何处理 LLM 调用失败?

**A:** 分层处理:
```python
def llm_node(state: State):
    for attempt in range(3):
        try:
            response = llm.invoke(state["messages"])
            return {"messages": [response], "error": None}
        except RateLimitError:
            time.sleep(2 ** attempt)  # 指数退避
        except Exception as e:
            if attempt == 2:
                return {
                    "messages": [fallback_response],
                    "error": str(e)
                }
```

### Q10: 生产环境部署建议?

**A:** 关键点:
1. **持久化**: 使用 PostgreSQL/Redis Checkpointer
2. **监控**: 集成 LangSmith 或自建监控
3. **限流**: 控制 LLM 调用频率
4. **容错**: 实现重试、降级、熔断
5. **水平扩展**: 无状态部署,状态存储分离
6. **安全**: API Key 管理、输入验证、输出过滤

---

## 速查表

### 关键 API

```python
# 创建图
from langgraph.graph import StateGraph, START, END
graph = StateGraph(StateType)

# 添加节点
graph.add_node("name", function)

# 添加边
graph.add_edge(START, "first_node")
graph.add_edge("node_a", "node_b")
graph.add_conditional_edge("node", router_func, path_map)

# 编译
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["node_name"],
    interrupt_after=["node_name"]
)

# 运行
result = app.invoke(input_state, config)
for step in app.stream(input_state, config):
    print(step)

# 状态管理
state = app.get_state(config)
app.update_state(config, values)
history = app.get_state_history(config)
```

### 常用导入

```python
# 核心
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated

# Checkpointer
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.postgres import PostgresSaver

# 消息
from langgraph.graph import add_messages
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage

# LLM
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# 工具
from langchain_core.tools import tool
```

---

## 学习资源

- **完整指南**: [LangGraph-Guide.md](LangGraph-Guide.md)
- **学习清单**: [LEARNING-CHECKLIST.md](LEARNING-CHECKLIST.md)
- **示例代码**: [examples/](examples/)
- **官方文档**: https://langchain-ai.github.io/langgraph/
- **API 参考**: https://langchain-ai.github.io/langgraph/reference/graphs/
