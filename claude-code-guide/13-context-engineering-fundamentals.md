# 13. 上下文工程基础概念

> 上下文工程是构建可靠、高效 AI Agent 的核心工程学科

## 目录
- [13.1 什么是上下文工程](#131-什么是上下文工程)
- [13.2 为什么需要上下文工程](#132-为什么需要上下文工程)
- [13.3 上下文窗口的本质](#133-上下文窗口的本质)
- [13.4 上下文问题分类](#134-上下文问题分类)
- [13.5 四大核心策略](#135-四大核心策略)
- [13.6 上下文工程的核心原则](#136-上下文工程的核心原则)
- [13.7 实际案例与实现](#137-实际案例与实现)
- [13.8 工具与框架](#138-工具与框架)

---

## 13.1 什么是上下文工程

### 定义

**上下文工程（Context Engineering）**是一门关于如何有效管理和优化大语言模型上下文窗口的工程学科，旨在让 AI Agent 在有限的上下文空间中获得"刚刚好"的信息。

```yaml
核心目标:
  - 最大化上下文利用效率
  - 最小化无关信息干扰
  - 保证关键信息可用
  - 平衡成本与性能
```

### 上下文工程 vs 提示工程

| 维度 | 提示工程 (Prompt Engineering) | 上下文工程 (Context Engineering) |
|------|------------------------------|--------------------------------|
| **关注点** | 如何写好单个提示词 | 如何管理整个对话历史 |
| **时间跨度** | 单次交互 | 长期会话 |
| **核心问题** | "说什么" | "记住什么、遗忘什么" |
| **优化目标** | 单次响应质量 | 整体系统效率 |
| **类比** | 写一封好邮件 | 管理一个邮箱 |

### 演变历程

```
阶段 1: 单轮对话 (2020-2022)
  - 简单的提示-响应
  - 无需上下文管理

阶段 2: 多轮对话 (2022-2023)
  - 简单的历史拼接
  - 遇到上下文窗口限制

阶段 3: 上下文工程 (2023-现在)
  - 系统化的上下文管理
  - 智能压缩与选择
  - 记忆系统架构

阶段 4: 认知架构 (未来)
  - 类人的记忆系统
  - 自适应上下文策略
  - 元认知能力
```

---

## 13.2 为什么需要上下文工程

### 核心挑战：有限的工作记忆

大语言模型的上下文窗口类似于计算机的 **RAM（随机存储器）**：

```yaml
相似性:
  - 容量有限
  - 访问快速
  - 易失性（会话结束即丢失）
  - 成本昂贵（每个 token 都要付费）

差异:
  - RAM 可以精确定位
  - 上下文窗口是语义理解
  - RAM 容量可扩展
  - 上下文窗口有固定上限
```

### 上下文窗口的容量演进

```
模型进化:
  GPT-3 (2020):        4k tokens   (~3000 字)
  GPT-3.5 (2022):      4k tokens
  GPT-4 (2023):        8k/32k tokens
  GPT-4 Turbo (2023):  128k tokens (~10万字)
  Claude 3 (2024):     200k tokens (~15万字)
  Gemini 1.5 (2024):   1M tokens   (~75万字)
  Claude Opus 4 (2025): 200k tokens

现实:
  - 窗口虽大，但质量会下降
  - "Lost in the Middle" 问题
  - 成本随窗口大小线性增长
  - 延迟随窗口大小增加
```

### 长期任务的上下文积累

当 AI Agent 执行长期任务时，上下文会不断积累：

```yaml
上下文类型:

  1. 指令 (Instructions)
     - 系统提示词
     - 用户偏好设置
     - 少样本示例 (Few-shot examples)
     - 工具描述和使用说明
     - 记忆和规则

  2. 知识 (Knowledge)
     - 项目文档
     - 代码库信息
     - 用户历史偏好
     - 领域知识
     - 事实性信息

  3. 交互历史 (History)
     - 用户的问题和请求
     - AI 的回复
     - 中间推理过程
     - 错误和修正

  4. 工具反馈 (Tool Outputs)
     - API 调用结果
     - 代码执行输出
     - 文件读取内容
     - 搜索结果
     - 数据库查询返回
```

### 实际案例：上下文爆炸

```yaml
场景: 代码审查 Agent

轮次 1: (2k tokens)
  User: "请审查这个 PR"
  Agent: 读取 PR 内容 (1k)
  Agent: 分析代码 (500)
  Agent: 给出建议 (500)

轮次 2: (4k tokens)
  User: "请详细说明第一个问题"
  + 之前所有内容 (2k)
  Agent: 详细解释 (2k)

轮次 3: (8k tokens)
  User: "请检查相关文件"
  + 之前所有内容 (4k)
  Agent: 读取 5 个文件 (3k)
  Agent: 分析 (1k)

轮次 4: (16k tokens)
  ...继续积累

轮次 N: 超过上下文窗口！
  - 早期信息被截断
  - 关键上下文丢失
  - Agent 开始"遗忘"
```

---

## 13.3 上下文窗口的本质

### 工作原理

```
┌────────────────────────────────────────────────┐
│         上下文窗口 (Context Window)             │
├────────────────────────────────────────────────┤
│                                                 │
│  [Token 1] [Token 2] ... [Token N]             │
│     ↓         ↓            ↓                    │
│  [Embedding 1] [Embedding 2] ... [Embedding N] │
│     ↓         ↓            ↓                    │
│  ┌──────────────────────────────────┐          │
│  │    Attention Mechanism            │          │
│  │  - Self-Attention                │          │
│  │  - 每个 token 关注其他 token     │          │
│  │  - 计算复杂度: O(N²)              │          │
│  └──────────────────────────────────┘          │
│     ↓                                           │
│  [下一个 token 的预测]                          │
│                                                 │
└────────────────────────────────────────────────┘

关键特性:
  1. 位置敏感
     - 相同内容在不同位置效果不同
     - 开头和结尾位置最重要

  2. 注意力衰减
     - 距离越远，注意力越弱
     - "Lost in the Middle" 现象

  3. Token 限制
     - 硬性上限（如 200k tokens）
     - 超出即截断或报错

  4. 成本线性
     - 每个 token 都计费
     - 输入和输出分别计费
```

### Token 计算

```yaml
常见误解: 1 token = 1 个字

实际情况:
  英文: ~1 token = 4 个字符 = 0.75 个单词
  中文: ~1 token = 1.5-2 个汉字
  代码: ~1 token = 3-4 个字符

示例:
  "Hello, world!"
  → ["Hello", ",", " world", "!"]
  → 4 tokens

  "你好世界"
  → ["你好", "世界"]
  → 2 tokens

  "function hello() {}"
  → ["function", " hello", "()", " {", "}"]
  → 5 tokens

工具:
  - OpenAI Tokenizer: https://platform.openai.com/tokenizer
  - tiktoken (Python): pip install tiktoken
  - gpt-tokenizer (Node.js): npm install gpt-tokenizer
```

### 上下文窗口 ≠ 无限记忆

```yaml
常见误解:
  ❌ "200k tokens 足够大，不需要管理"
  ❌ "把所有信息都塞进去就好"
  ❌ "上下文越多，效果越好"

现实问题:

  1. 质量下降
     Position: |---开头---|---中间---|---结尾---|
     Quality:  |  ★★★★★  |  ★★★☆☆  |  ★★★★★  |
     - 中间部分容易被"忽略"
     - 大海捞针任务成功率下降

  2. 成本爆炸
     100k tokens × $0.015/1k = $1.50 每次调用
     每天 100 次 = $150/天 = $4,500/月

  3. 延迟增加
     200k tokens → 处理时间 10-30 秒
     用户体验显著下降

  4. 干扰增加
     无关信息越多，模型越容易分心
     核心任务被次要细节淹没
```

---

## 13.4 上下文问题分类

### 四大上下文病症

#### 1. 上下文中毒 (Context Poisoning)

**定义：** 幻觉内容被写入上下文，污染后续推理

```yaml
示例场景:
  Agent: "我检查了 database.js 文件..." (但实际没有这个文件)
  → 幻觉被写入上下文
  User: "请基于刚才的分析继续"
  Agent: "根据 database.js 的内容..." (继续基于错误信息)

危害:
  - 错误信息被当作事实
  - 后续推理完全错误
  - 难以自我纠正

预防:
  - 验证工具输出
  - 定期清理上下文
  - 使用结构化输出
  - 事实核查机制
```

#### 2. 上下文干扰 (Context Distraction)

**定义：** 过多无关信息压倒模型对核心任务的关注

```yaml
示例场景:
  Task: "优化这个函数的性能"
  Context:
    - 整个项目历史 (50k tokens)
    - 所有文件内容 (30k tokens)
    - 详细的会议记录 (10k tokens)
    - 目标函数 (200 tokens) ← 真正需要的

  结果:
    Agent 花大量时间讨论无关的历史问题
    而忽略了当前函数的性能优化

危害:
  - 注意力分散
  - 响应偏离主题
  - 效率低下

预防:
  - 只提供相关信息
  - 明确标记优先级
  - 使用分层上下文
  - 动态过滤
```

#### 3. 上下文混淆 (Context Confusion)

**定义：** 冗余或矛盾信息导致输出不一致

```yaml
示例场景:
  Context 包含:
    Message 1: "用户偏好 TypeScript"
    Message 50: "用户说 'JavaScript 就够了'"
    Message 100: "用户要求用 TypeScript"

  Agent 行为不稳定:
    有时生成 TS 代码
    有时生成 JS 代码
    取决于哪条消息被"注意到"

危害:
  - 输出不一致
  - 用户体验差
  - 难以预测行为

预防:
  - 去重和合并
  - 保持信息一致性
  - 显式覆盖旧信息
  - 版本化偏好设置
```

#### 4. 上下文冲突 (Context Clash)

**定义：** 上下文中存在相互矛盾的事实或指令

```yaml
示例场景:
  Instruction A: "始终遵循 PEP 8 规范"
  Instruction B: "使用团队自定义的 4 空格缩进"
  (PEP 8 要求 4 空格，但如果团队规范不同...)

  Tool Output 1: "文件不存在"
  Tool Output 2: "文件内容如下..." (缓存的旧数据)

危害:
  - Agent 无法决策
  - 行为随机
  - 可能陷入循环

预防:
  - 明确优先级规则
  - 冲突检测
  - 自动解决策略
  - 时间戳和版本控制
```

### 问题严重性矩阵

```
           │ 低频发生 │ 中频发生 │ 高频发生 │
───────────┼─────────┼─────────┼─────────┤
高影响     │ 中毒    │ 冲突    │ 干扰    │
中影响     │         │ 混淆    │         │
低影响     │         │         │         │

优先处理: 干扰 > 冲突 > 中毒 > 混淆
```

---

## 13.5 四大核心策略

上下文工程的四大策略（WSCI 框架）：

```
┌────────────────────────────────────────────────┐
│         上下文工程 WSCI 框架                    │
├────────────────────────────────────────────────┤
│                                                 │
│  W - Write   (写入) → 保存到上下文外            │
│  S - Select  (选择) → 检索最相关信息            │
│  C - Compress(压缩) → 减少 token 使用           │
│  I - Isolate (隔离) → 拆分避免干扰             │
│                                                 │
└────────────────────────────────────────────────┘
```

### 策略 1: Write（写入上下文外）

**核心思想：** 将重要信息保存到上下文窗口之外，供后续使用

```yaml
实现方式:

  1. Scratchpad (草稿板)
     - 用途: 单次会话临时存储
     - 内容: 计划、中间结论、TODO
     - 实现: 文件系统或运行时 State
     - 生命周期: 当前会话

  2. Long-term Memory (长期记忆)
     - 用途: 跨会话持久化
     - 内容: 用户偏好、项目知识、历史决策
     - 实现: 数据库、向量存储
     - 生命周期: 永久

  3. Structured State (结构化状态)
     - 用途: 任务状态管理
     - 内容: 当前进度、依赖关系
     - 实现: JSON、数据库
     - 生命周期: 任务完成

示例:
  # Anthropic 多智能体研究系统
  主研究员: "将研究计划写入 Memory"
  → 防止因上下文截断而丢失计划
  → 其他研究员可读取共享计划
```

### 策略 2: Select（选择上下文）

**核心思想：** 动态检索最相关的信息注入当前上下文

```yaml
选择维度:

  1. 从 Scratchpad 选择
     - State 存储: 开发者控制字段暴露
     - 工具实现: 通过工具调用读取

  2. 记忆检索
     a) 程序性记忆 (Procedural)
        - CLAUDE.md 规则文件
        - 行为模式和流程

     b) 情景性记忆 (Episodic)
        - 少样本示例
        - 历史交互案例

     c) 语义性记忆 (Semantic)
        - 事实知识
        - 向量嵌入检索
        - 知识图谱

  3. 工具选择
     - RAG 技术检索相关工具
     - 仅加载当前需要的工具描述
     - 提升工具选择准确率

  4. 知识检索 (RAG)
     - 向量搜索
     - 关键词匹配
     - AST 语法树解析 (代码)
     - 知识图谱查询
     - 重排序 (Reranking)

技术栈:
  向量数据库: Pinecone, Weaviate, Qdrant
  嵌入模型: OpenAI ada-002, Cohere
  重排序: Cohere Rerank, Cross-Encoder
```

### 策略 3: Compress（压缩上下文）

**核心思想：** 保留完成任务所需的最少 token

```yaml
压缩技术:

  1. 上下文摘要 (Summarization)
     时机:
       - 接近上下文上限时
       - 特定节点 (如搜索工具后)
       - 定期摘要 (每 N 轮)

     方法:
       - 递归摘要 (分段后再汇总)
       - 分层摘要 (按重要性层级)
       - 专用摘要模型 (微调)

     示例:
       原始 (1000 tokens):
         User: "分析这个文件..."
         Agent: "我读取了文件，发现..."
         [详细分析过程]

       摘要 (100 tokens):
         "已分析 auth.js: 使用 JWT，存在 XSS 风险"

  2. 上下文修剪 (Trimming/Pruning)
     策略:
       - 基于时间: 删除最早消息
       - 基于重要性: 保留关键信息
       - 基于相关性: 删除无关内容
       - 智能剪枝器: 训练模型判断

     实现:
       - 滑动窗口
       - 优先级队列
       - 注意力分数
       - 人工规则

  3. 符号化表示
     将详细内容替换为简洁符号:
       原: "执行了 npm install，安装了 50 个包..."
       后: "[✓ npm install]"

  4. 增量更新
     只保存变化部分:
       原: 完整文件内容
       后: Diff + 引用
```

### 策略 4: Isolate（隔离上下文）

**核心思想：** 将上下文拆分，避免相互干扰

```yaml
隔离方式:

  1. 子代理隔离 (Subagents)
     - 每个子任务独立上下文
     - 主代理只接收结果摘要
     - 避免细节污染主上下文

     示例:
       Main Agent Context (精简):
         "搜索结果: 找到 5 个相关文件"

       Sub-Agent Context (详细):
         [完整的搜索过程和所有结果]

  2. 会话隔离 (Sessions)
     - 不同任务使用不同会话
     - 跨会话共享必要的记忆
     - 定期清理过期会话

  3. 命名空间隔离
     - 按功能模块分离上下文
     - 明确的边界和接口
     - 减少跨模块干扰

  4. 分层上下文
     - 核心上下文: 始终可见
     - 任务上下文: 仅当前任务
     - 工具上下文: 使用时加载

架构示例:
  ┌─────────────────────┐
  │   Global Context    │ ← 系统提示词、用户偏好
  └─────────────────────┘
           │
  ┌────────┴────────┐
  │  Task Context   │ ← 当前任务相关
  └─────────────────┘
           │
  ┌────────┴────────┐
  │  Tool Context   │ ← 工具调用详情
  └─────────────────┘
```

---

## 13.6 上下文工程的核心原则

### 原则 1: 刚刚好原则（Goldilocks Principle）

```yaml
❌ 太少信息:
  - Agent 缺乏足够上下文
  - 无法完成任务
  - 频繁询问用户

✅ 刚刚好:
  - 提供完成任务所需的信息
  - 不多不少
  - 清晰聚焦

❌ 太多信息:
  - 信息过载
  - 注意力分散
  - 成本浪费

目标: 在每个任务步骤中，让 Agent 获得"刚刚好"的信息
```

### 原则 2: 分层原则（Layered Principle）

```yaml
信息分层:
  Layer 0: 必需 (Always On)
    - 系统提示词
    - 核心规则
    - 当前任务

  Layer 1: 重要 (Frequently Used)
    - 用户偏好
    - 项目上下文
    - 最近历史

  Layer 2: 可选 (On Demand)
    - 详细文档
    - 历史记录
    - 工具输出

加载策略:
  - Layer 0: 始终加载
  - Layer 1: 默认加载，可压缩
  - Layer 2: 按需加载

好处:
  - 控制上下文大小
  - 优先级明确
  - 灵活调整
```

### 原则 3: 时效原则（Temporal Principle）

```yaml
信息时效性:
  新鲜 (Fresh):
    - 最近几轮对话
    - 当前任务状态
    - 最新工具输出
    → 高优先级，保留

  中期 (Medium):
    - 几轮前的上下文
    - 相关历史决策
    → 可压缩，选择性保留

  陈旧 (Stale):
    - 早期对话
    - 不相关的历史
    → 可以丢弃或深度压缩

衰减策略:
  Priority = Relevance × Recency

  Recency:
    - 最近 5 轮: 1.0
    - 5-10 轮: 0.7
    - 10-20 轮: 0.4
    - >20 轮: 0.1
```

### 原则 4: 相关性原则（Relevance Principle）

```yaml
相关性判断:

  1. 任务相关性
     "当前任务需要这个信息吗？"
     Yes → 保留
     No → 可以移除

  2. 因果相关性
     "这个信息影响后续决策吗？"
     Yes → 高优先级
     No → 低优先级

  3. 语义相关性
     "这个信息与主题相关吗？"
     使用嵌入向量计算相似度

实现:
  def is_relevant(info, current_task):
      # 语义相关性
      semantic_score = cosine_similarity(
          embed(info),
          embed(current_task)
      )

      # 时间相关性
      time_score = recency_decay(info.timestamp)

      # 综合评分
      return semantic_score * 0.7 + time_score * 0.3
```

### 原则 5: 一致性原则（Consistency Principle）

```yaml
维护一致性:

  1. 信息版本控制
     - 使用时间戳
     - 明确最新版本
     - 覆盖旧信息

  2. 去重
     - 检测重复内容
     - 合并相似信息
     - 保留最完整版本

  3. 冲突解决
     优先级规则:
       用户明确指令 > 系统规则 > 历史偏好

  4. 单一真相来源 (SSOT)
     关键信息只存一份
     通过引用访问

示例:
  ❌ 多处存储用户偏好:
    Message 1: "用户喜欢 TS"
    Message 50: "用户偏好 TS"
    Message 100: "用户要求 TS"

  ✅ 单一来源:
    Memory["user_preference"]["language"] = "TypeScript"
    所有地方引用: ${user_preference.language}
```

### 原则 6: 可追溯原则（Traceability Principle）

```yaml
信息溯源:

  1. 来源标记
     每条信息标注来源:
       - 用户输入
       - 工具输出
       - 系统生成
       - 记忆检索

  2. 置信度
     标记信息可靠性:
       - 确定性事实: 100%
       - 工具输出: 90%
       - Agent 推理: 70%
       - 外部检索: 60%

  3. 时间戳
     记录信息产生时间:
       - 检测过期信息
       - 解决冲突
       - 审计追踪

示例结构:
  {
    "content": "用户偏好 TypeScript",
    "source": "user_input",
    "confidence": 1.0,
    "timestamp": "2026-02-10T10:30:00Z",
    "context_id": "session_123"
  }
```

---

## 总结

上下文工程已从"技巧"演变为 **AI Agent 开发的核心工程学科**。

### 核心要点

```yaml
1. 本质理解:
   - 上下文窗口 = 工作记忆
   - 有限、昂贵、易失
   - 质量 > 数量

2. 四大问题:
   - 中毒 (Poisoning)
   - 干扰 (Distraction)
   - 混淆 (Confusion)
   - 冲突 (Clash)

3. WSCI 框架:
   - Write: 保存到外部
   - Select: 检索相关
   - Compress: 智能压缩
   - Isolate: 上下文隔离

4. 核心原则:
   - 刚刚好
   - 分层
   - 时效性
   - 相关性
   - 一致性
   - 可追溯
```

### 类比总结

```
上下文工程 ≈ 操作系统的内存管理

操作系统管理 RAM:          上下文工程管理 Context:
- 页面置换算法             - 信息选择策略
- 缓存管理                 - 记忆系统
- 内存压缩                 - 上下文压缩
- 进程隔离                 - 会话隔离
- 垃圾回收                 - 上下文清理

目标都是: 有限资源的最优利用
```

---

## 13.7 实际案例与实现

### 案例 1: Anthropic 多智能体研究系统

Anthropic 的多智能体研究系统是上下文工程的经典案例。

```yaml
场景: 长期研究任务

挑战:
  - 研究计划可能很长（数千 tokens）
  - 上下文窗口 200k tokens 可能被截断
  - 多个子研究员需要共享计划

解决方案:
  Write 策略:
    - LeadResearcher 将研究计划写入 Memory
    - 持久化存储，避免截断丢失
    - 跨会话可访问

  Isolate 策略:
    - 多个子研究员并行工作
    - 每个有独立的上下文窗口
    - 专注于不同的子任务

效果:
  - 多代理 15× 的 token 使用量
  - 但并行处理，总时间大幅缩短
  - 每个子代理专注度更高

实现代码示例:
  # 主研究员保存计划
  def lead_researcher_plan(task):
      plan = generate_research_plan(task)

      # Write: 保存到 Memory
      save_to_memory("research_plan", plan)

      # Isolate: 启动子研究员
      subagents = [
          spawn_agent("researcher-1", "探索方法 A"),
          spawn_agent("researcher-2", "探索方法 B"),
          spawn_agent("researcher-3", "文献综述")
      ]

      return coordinate_subagents(subagents)
```

### 案例 2: Claude Code 的自动压缩

Claude Code 在接近上下文上限时自动触发压缩。

```yaml
触发机制:
  当上下文使用率 > 95% 时:
    → 触发 auto-compact

压缩策略:
  1. 保留最近的对话（Working Memory）
  2. 递归摘要历史对话
  3. 提取关键决策和重要上下文
  4. 过滤冗余和重复信息

实现逻辑:
  def auto_compact(context, threshold=0.95):
      usage = context.token_count / MAX_TOKENS

      if usage > threshold:
          # Compress: 摘要策略
          recent_msgs = context.messages[-10:]  # 保留最近 10 条
          historical = context.messages[:-10]

          summary = summarize_recursive(historical)

          # 重建上下文
          context.messages = [
              system_prompt,
              summary,
              *recent_msgs
          ]

      return context

效果:
  - 原始: 190k tokens (95%)
  - 压缩后: 50k tokens (25%)
  - 保留率: 关键信息 100%
```

### 案例 3: ChatGPT 的长期记忆

ChatGPT、Cursor、Windsurf 都实现了跨会话的长期记忆。

```yaml
Write 策略:
  自动记忆生成:
    - 在对话中提取用户偏好
    - 识别重要事实
    - 生成语义记忆

  示例:
    User: "我住在旧金山，喜欢用 TypeScript"
    → Memory: {"location": "San Francisco", "preference": "TypeScript"}

Select 策略:
  记忆检索:
    - 使用向量嵌入索引
    - 根据当前上下文检索相关记忆
    - 注入到提示词中

潜在问题 (Simon Willison 案例):
  User: "生成一张城市图片"
  Agent: [检索到 location: San Francisco]
  Agent: "这是旧金山的图片" ← 未经用户明确要求

  用户感觉:
    - 上下文窗口"不再属于我"
    - 记忆检索过于主动
    - 缺乏透明度

改进建议:
  - 明确告知使用了哪些记忆
  - 允许用户控制记忆检索
  - 提供记忆管理界面
```

### 案例 4: 代码 Agent 的 RAG 策略

Windsurf 的 Varun 分享了代码 Agent 面临的挑战。

```yaml
问题: 索引代码 ≠ 上下文检索

  简单向量搜索的局限:
    - 代码库规模增长时不可靠
    - 语义相似 ≠ 代码相关
    - 缺少结构化信息

Windsurf 的多层次检索策略:

  Layer 1: 文件搜索
    - grep / file search
    - 快速定位文件

  Layer 2: AST 解析
    - 语法树解析
    - 按语义边界分块
    - 理解代码结构

  Layer 3: 向量搜索
    - 语义相似性
    - 辅助而非主要手段

  Layer 4: 知识图谱
    - 代码依赖关系
    - 函数调用图
    - 模块关系

  Layer 5: 重排序 (Reranking)
    - 根据相关性排序
    - 选择最相关的 top-k
    - 注入上下文

实现示例:
  def retrieve_code_context(query):
      # Layer 1: 文件搜索
      files = file_search(query)

      # Layer 2: AST 解析
      chunks = []
      for file in files:
          ast = parse_ast(file)
          chunks.extend(chunk_by_semantic(ast))

      # Layer 3: 向量搜索
      embeddings = embed(chunks)
      similar = vector_search(query, embeddings, top_k=50)

      # Layer 4: 知识图谱
      related = knowledge_graph.get_related(similar)

      # Layer 5: 重排序
      ranked = rerank(query, similar + related, top_k=10)

      return ranked
```

### 案例 5: HuggingFace CodeAgent 的沙箱隔离

HuggingFace 的 CodeAgent 使用代码输出而非工具调用。

```yaml
传统 Tool Calling:
  LLM → JSON {"tool": "search", "query": "..."} → Tool → Result

CodeAgent 方式:
  LLM → Python Code → Sandbox Execution → Selected Results

优势:
  Isolate 策略:
    - Token 密集对象（图片、音频）存储在环境中
    - 不占用 LLM 上下文窗口
    - 按需传递引用

示例:
  # LLM 生成代码
  image = generate_image("sunset")  # 生成图片
  processed = apply_filter(image, "vintage")  # 处理
  save(processed, "output.jpg")  # 保存

  # 沙箱执行
  # image 对象不返回给 LLM
  # 只返回: "Image saved to output.jpg"

  # 后续使用
  # LLM: show(image)  # 变量仍在沙箱中

好处:
  - 状态管理更好
  - Token 使用降低
  - 支持复杂对象
```

### 案例 6: OpenAI Swarm 的关注点分离

OpenAI Swarm 库强调多代理的关注点分离。

```yaml
设计理念:
  - 每个 Agent 负责特定子任务
  - 独立的工具集
  - 独立的指令
  - 独立的上下文窗口

示例: 客服系统
  Agent 1: 售前咨询
    Tools: [产品目录, 价格查询]
    Context: 产品知识库

  Agent 2: 技术支持
    Tools: [故障诊断, 工单系统]
    Context: 技术文档

  Agent 3: 退换货
    Tools: [订单查询, 退款流程]
    Context: 政策文档

  协调:
    Triage Agent 判断用户问题类型
    → 路由到相应的专家 Agent
    → 专家处理并返回结果
    → 需要时切换到其他专家

优势:
  - 每个 Agent 专注度高
  - 上下文窗口利用率高
  - 易于维护和扩展
  - 并行处理提升效率
```

---

## 13.8 工具与框架

### LangGraph - 低级编排框架

LangGraph 原生支持所有上下文工程策略。

```yaml
Write Context:
  1. Thread-scoped Memory (短期)
     - 使用 checkpointing 持久化状态
     - 跨 Agent 步骤访问
     - 类似 Scratchpad

  2. Long-term Memory (长期)
     - 跨会话持久化
     - 支持文件和向量检索
     - LangMem 提供高级抽象

Select Context:
  1. State Access
     - 每个节点可以 fetch state
     - 细粒度控制暴露内容

  2. Memory Retrieval
     - 文件检索
     - 向量嵌入检索
     - 支持多种检索方式

  3. Tool Selection
     - LangGraph Bigtool 库
     - 语义搜索工具描述
     - 处理大量工具

  4. RAG Integration
     - 丰富的 RAG 教程
     - 支持多种检索策略

Compress Context:
  1. Message Summarization
     - 内置 summarization utilities
     - 定期摘要消息列表

  2. Custom Compression Nodes
     - 在特定点添加摘要节点
     - 后处理工具调用输出

Isolate Context:
  1. State Schema
     - 定义 state schema
     - 字段级隔离
     - 选择性暴露

  2. Sandbox Support
     - E2B sandbox 集成
     - Pyodide sandbox
     - 状态持久化

  3. Multi-agent
     - Supervisor 库
     - Swarm 库
     - 丰富的示例

代码示例:
  from langgraph.graph import StateGraph
  from langgraph.checkpoint import MemorySaver

  # 定义状态
  class AgentState(TypedDict):
      messages: list  # 暴露给 LLM
      scratchpad: dict  # 内部存储，不暴露
      long_term_memory: list  # 长期记忆

  # 创建 graph
  graph = StateGraph(AgentState)

  # 添加节点
  graph.add_node("agent", agent_node)
  graph.add_node("summarize", summarize_node)  # 压缩节点

  # 条件边：决定是否压缩
  def should_summarize(state):
      return len(state["messages"]) > 20

  graph.add_conditional_edges(
      "agent",
      should_summarize,
      {True: "summarize", False: END}
  )

  # 使用 checkpointing
  checkpointer = MemorySaver()
  app = graph.compile(checkpointer=checkpointer)
```

### LangSmith - 可观测性与评估

```yaml
功能:
  1. Agent Tracing
     - 完整的执行轨迹
     - Token 使用追踪
     - 工具调用可视化

  2. Token Usage Monitoring
     - 每步 token 消耗
     - 成本分析
     - 识别优化机会

  3. Agent Evaluation
     - 测试上下文工程效果
     - A/B 测试不同策略
     - 性能基准测试

工作流:
  1. 观察 → 发现 token 浪费
  2. 假设 → 设计压缩策略
  3. 实现 → 在 LangGraph 中实现
  4. 评估 → LangSmith 测试效果
  5. 迭代 → 持续优化
```

### Claude Code - 内置上下文工程

```yaml
Auto-compact:
  - 95% 阈值触发
  - 递归摘要
  - 保留关键上下文

CLAUDE.md:
  - 程序性记忆
  - 项目特定规则
  - Select 策略的实现

Skills System:
  - 可复用的能力
  - 上下文隔离
  - 动态加载

Subagents:
  - 多个专业化代理
  - 独立上下文窗口
  - 并行执行
```

### Cursor & Windsurf - 代码 Agent

```yaml
共同特性:
  1. Rules Files
     - .cursorrules / .windsurfrules
     - 项目规则和偏好
     - 程序性记忆

  2. Long-term Memory
     - 自动提取用户偏好
     - 跨会话持久化
     - 向量检索

  3. 高级 RAG
     - AST 解析
     - 知识图谱
     - 多层次检索
     - 重排序

  4. Context Management
     - 自动摘要
     - 智能选择
     - Token 优化
```

### 最佳实践总结

```yaml
1. 始终监控 Token 使用
   工具: LangSmith, 自定义追踪

2. 从简单开始
   - 先实现基础功能
   - 再优化上下文

3. 测量效果
   - 建立 baseline
   - A/B 测试
   - 量化改进

4. 迭代优化
   - 小步快跑
   - 持续改进
   - 数据驱动

5. 选择合适的工具
   - LangGraph: 灵活性高
   - 托管服务: 快速上手
   - 自建: 完全控制
```

---

### 下一步

继续阅读 [02. 上下文工程](./02-context-engineering.md) 了解 Claude Code 的具体实现细节。

---

**参考资源：**
- [LangChain Blog: Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/)
- [掘金文章: 上下文工程](https://juejin.cn/post/7602850166731292681)
- [Claude Code: 02-context-engineering.md](./02-context-engineering.md)
- [Anthropic: Multi-Agent Researcher](https://www.anthropic.com/research/building-effective-agents)
- [OpenAI Swarm](https://github.com/openai/swarm)
- [HuggingFace CodeAgent](https://huggingface.co/docs/transformers/en/agents)

Sources:
- [Context Engineering for Agents - LangChain Blog](https://blog.langchain.com/context-engineering-for-agents/)
- [上下文工程（Context Engineering） - 掘金](https://juejin.cn/post/7602850166731292681)
