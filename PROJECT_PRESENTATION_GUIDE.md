# 面试项目讲解指南 — 如何向面试官讲解 DeepTutor

> 这份文档教你**怎么讲**这个项目。每节包含：讲解话术（直接念）、技术深挖点（面试官可能追问）、代码证据（你说"我看过源码"的底气）。
> 标记 🎯 的是必讲项，标记 💎 的是加分项。

---

## 总体讲解策略

**核心原则：先讲清楚是什么，再讲怎么做，最后讲我能改进什么。**

讲解顺序（约 5-8 分钟）：
1. 一句话定位（10 秒）
2. 架构全景（1 分钟）
3. 逐个关键技术展开（3-5 分钟）
4. 我的思考和改进（1-2 分钟）

---

## 🎯 第一站：项目定位（10 秒开场）

**讲解话术：**
> "DeepTutor 是港大 HKUDS 实验室开源的 Agent-Native 个性化学习助手，目前在 GitHub 上有 17.7k stars。
> 简单来说，它不是一个普通的 Chatbot，而是一个有记忆、有人格、能主动找用户、能协调多个 AI Agent 协作的智能学习系统。"

**面试官可能的反应：** "听起来挺复杂的，说说架构吧？"

---

## 🎯 第二站：架构全景（1 分钟，面试必画图）

**讲解话术：**
> "这个项目的核心架构是**两层插件模型**。
>
> 最底层是 **Tool 层**，每个 Tool 是一个原子级操作——比如 RAG 检索、Web Search、Code Execution、Reason（推理增强）、Brainstorm。它们都继承 `BaseTool` 抽象类，提供 `get_definition()` 和 `execute()` 两个标准方法，返回统一的 `ToolResult`。
>
> 上面一层是 **Capability 层**，每个 Capability 是一个多步骤的 Agent Pipeline。比如 Chat（日常对话）、Deep Solve（多 Agent 协作求解）、Deep Research（深度研究）、Deep Question（苏格拉底式提问）、Math Animator（数学动画）。它们继承 `BaseCapability`，定义了 `CapabilityManifest` 来声明自己需要哪些 Tool。
>
> 所有请求通过三个入口进来——CLI（命令行，用 Typer 框架）、WebSocket API（给前端用）、Python SDK（给开发者集成），三个入口最终都汇到 `ChatOrchestrator`，它负责把请求路由到对应的 Capability。
>
> 整个系统是事件驱动的，所有组件通过 `StreamBus` 异步事件流通信。"

**画图（边画边说）：**
```
CLI / WebSocket / SDK  →  ChatOrchestrator
                              ↓
                    ┌─────────┴─────────┐
                    │                   │
              ToolRegistry      CapabilityRegistry
              (Level 1 原子)     (Level 2 编排)
                    ↑                   │
              BaseTool            BaseCapability
              RAG/Search/...      Chat/Solve/...
```

**追问准备：**
- "为什么两层？" → "解耦。Tool 可以被多个 Capability 复用。比如 RAG Tool 同时被 Chat、Deep Solve、Deep Research 三个 Capability 使用。如果耦合在一起，改一个 Tool 要改三个 Capability。"
- "为什么用注册表模式？" → "插件化。新增 Tool 或 Capability 只需要实现接口、注册到 Registry，不需要改编排逻辑。符合开闭原则。"
- "StreamBus 是什么？" → "基于 asyncio 的发布-订阅事件总线。所有 Agent 的中间过程（Plan 完成、ReAct 迭代、Tool 调用结果）都作为 Event 发布到 StreamBus，消费者可以实时获取进度，实现流式输出。"

**代码证据（展示你看过源码）：**
- Tool 协议定义：`deeptutor/core/tool_protocol.py`，`BaseTool` 抽象类
- Capability 协议定义：`deeptutor/core/capability_protocol.py`，`BaseCapability`
- 注册表：`deeptutor/runtime/registry/tool_registry.py`，`ToolRegistry` 单例
- 编排入口：`ChatOrchestrator` 路由逻辑
- Facade 统一入口：`deeptutor/app/facade.py`，`DeepTutorApp` 类

---

## 🎯 第三站：多 Agent 协作 — Deep Solve（2 分钟重点）

**讲解话术：**
> "这是我最有感触的模块。Deep Solve 是一个**三阶段多 Agent 协作 Pipeline**：Plan → ReAct → Write。
>
> **第一阶段 Plan**：`PlannerAgent` 把用户的复杂问题分解成有序步骤，每个步骤是一个 `PlanStep`，包含目标、建议使用的工具、当前状态。
>
> **第二阶段 Solve**：`SolverAgent` 对每个 PlanStep 执行 ReAct 循环——就是 Thought（思考）→ Action（调用工具）→ Observation（观察结果），然后再 Thought，如此往复。每步还会写一个 `self_note` 做自我反思。关键是有**容错机制**：如果 Solver 卡住了，它可以请求 Replan，让 Planner 重新规划，最多 Replan 2 次。
>
> **第三阶段 Write**：`WriterAgent` 从所有步骤积累的证据中生成最终回答，强制标注引用来源。
>
> 整个过程的核心是 **Scratchpad 共享内存**——三个 Agent 读写同一个 Scratchpad，它是 Single Source of Truth，包含 Plan、ReAct 迭代记录、Sources、Metadata。"

**关键数字（面试时脱口而出）：**
- 每个 PlanStep 最多 **5 次** ReAct 迭代
- 最多 **2 次** Replan
- 全局 **40 次** Tool Call 安全上限
- WriterAgent 支持简单模式（单次 LLM）和详细模式（迭代写入）

**追问准备：**
- "为什么不直接让一个 Agent 做完？" → "职责分离。规划、执行、输出三个阶段各司其职，Planner 不需要知道怎么执行，Solver 不需要知道怎么组织语言。这样每个 Agent 的 System Prompt 更聚焦，效果更好，也更容易调试。"
- "Replan 的触发条件是什么？" → "Solver 在迭代中如果发现当前 Plan 无法完成目标（比如工具返回了意外结果），会设置 `status = replan`，主控 `main_solver.py` 检测到后会让 PlannerAgent 重新规划。已完成的步骤会保留。"
- "Scratchpad 怎么实现的？" → "它是一个 Python 数据类，包含 plan（Plan 对象）、react_entries（每步的 ReAct 记录）、sources（引用来源）、metadata（元信息）。所有 Agent 通过引用同一个 Scratchpad 实例来共享状态。"

**代码证据：**
- 主控逻辑：`deeptutor/agents/solve/main_solver.py`，`MainSolver` 类的 `run()` 方法
- PlannerAgent：`deeptutor/agents/solve/agents/planner_agent.py`
- SolverAgent：`deeptutor/agents/solve/agents/solver_agent.py`
- WriterAgent：`deeptutor/agents/solve/agents/writer_agent.py`
- Scratchpad：`deeptutor/agents/solve/memory/scratchpad.py`

---

## 🎯 第四站：持久化 Agent — TutorBot（1.5 分钟）

**讲解话术：**
> "TutorBot 是这个项目最有特色的设计。它不是一个 Chatbot，而是一个**持久化的自主 Agent**。
>
> 每个 TutorBot 有自己独立的工作区，位于 `data/tutorbot/{bot_id}/`，里面有自己的人格文件（SOUL.md）、记忆、会话历史、配置。你可以创建多个 Bot，它们之间完全隔离。
>
> 生命周期是：创建（独立目录）→ 启动（AgentLoop + MessageBus + ChannelManager + Heartbeat + SessionManager 多个异步任务）→ 运行（异步处理消息）→ 停止（优雅关闭）。
>
> 它有几个特别的设计：
> 1. **Heartbeat 心跳机制**：不是简单定时任务，而是 LLM 驱动的。每次心跳到了，Bot 会读 HEARTBEAT.md 获取上下文，然后 LLM 决定是 'skip'（跳过）还是 'run'（执行）。这样 Bot 就能**主动**找用户——比如学习提醒、复习计划。
> 2. **多渠道接入**：支持 Telegram、Discord、Slack 等 11 个平台，统一抽象为 `InboundMessage`。
> 3. **Team 协作**：用户可以输入 `/team <goal>`，系统会创建 2-3 个 AI 队友，通过 Task Board（看板）和 Mailbox（邮箱）协调工作。
> 4. **Skill 学习**：通过 SKILL.md 文件，Bot 可以学习新能力。"

**追问准备：**
- "Heartbeat 的 LLM 决策不浪费 Token 吗？" → "确实有成本，但换来的是精准的主动交互。不是每次心跳都触发，LLM 会根据上下文判断是否需要。而且可以通过配置调整心跳间隔，默认 30 分钟。"
- "Team 的 Task Board 怎么避免死锁？" → "任务有依赖关系，创建时会做**环检测**。Worker 认领任务前会检查依赖是否都完成了。"
- "多个 Bot 同时运行安全吗？" → "这也是我发现的问题之一。AgentLoop 的 `_active_tasks` 列表在 done_callback 中被修改，多协程竞态。需要加 `asyncio.Lock` 保护。"

**代码证据：**
- AgentLoop：`deeptutor/tutorbot/agent/loop.py`，核心消息处理循环
- Heartbeat：`deeptutor/tutorbot/heartbeat/service.py`，`HeartbeatService` 类
- Team：`deeptutor/tutorbot/agent/team/__init__.py`，`TeamManager`
- Bot 创建/启动：`deeptutor/services/tutorbot/manager.py`

---

## 🎯 第五站：记忆系统（1 分钟）

**讲解话术：**
> "TutorBot 的记忆是**三层设计**：
>
> **短期记忆**就是当前对话的 Context Window，LLM 直接能看到。
>
> **中期记忆**是 Session 级别，存在数据库或文件里，按 session 关联。
>
> **长期记忆**是跨会话持久化的 Markdown 文件：
> - `PROFILE.md` 存用户画像——身份、偏好、知识水平、学习目标
> - `MEMORY.md` 存持久化的事实和上下文
> - `SUMMARY.md` 存对话的压缩摘要
>
> 整合机制是：当 Token 接近上限时，`MemoryConsolidator` 触发 LLM 生成摘要，替换掉旧的对话。如果整合失败，会优雅降级到原始归档。
>
> 还有一个跨 Bot 共享的设计：所有 Bot 实例通过 `data/memory/` 共享目录访问统一的用户 Profile。"

**追问准备：**
- "为什么用 Markdown 存储？" → "人可读、Agent 可消费。调试时人可以直接打开看，Agent 也可以直接读取。比数据库更透明。"
- "MemoryConsolidator 的锁机制？" → "它用锁保护整合过程，防止多个协程同时触发整合。如果整合失败，catch 异常，回退到原始归档，不会丢失数据。"

**代码证据：**
- Memory 模块：`deeptutor/tutorbot/agent/memory.py`
- MemoryConsolidator：管理整合策略和锁

---

## 🎯 第六站：RAG 知识库（1 分钟）

**讲解话术：**
> "DeepTutor 的 RAG 基于 **LlamaIndex** 框架。整个流程是标准的五环节：文档解析 → 分块（Chunking）→ 向量化（Embedding）→ 检索（Retrieval）→ 生成（Generation）。
>
> 它的 RAG 实现有个亮点——**Smart Query Generation**。不是直接用用户问题去检索，而是先让 LLM 根据上下文生成多个检索 Query，做多路检索，扩大召回率。
>
> 检索完成后还支持 **Reranking**（重排序），用 Cross-Encoder 模型对检索结果精排，提升相关性。
>
> 向量存储支持多种 Provider，开发阶段可以用本地 Chroma（零配置），生产环境可以切到 Qdrant 或 Milvus。"

**追问准备：**
- "Multi-query 的好处？" → "用户问题可能有歧义或不完整，生成多个 Query 可以从不同角度检索，提升召回率。比如用户问'Transformer 是什么'，可能会生成'Attention 机制原理'、'Transformer 架构详解'等多个 Query。"
- "Reranker 和向量检索的区别？" → "向量检索用双塔模型（Bi-Encoder），速度快但精度一般；Reranker 用 Cross-Encoder，精度高但慢。所以通常是先向量检索粗排 Top-100，再 Reranker 精排 Top-10。"
- "分块策略怎么选？" → "有三种：Fixed（固定大小，最简单）、Semantic（按语义边界分，质量好但慢）、Recursive（递归分割，平衡效率和质量）。通常 256-1024 tokens，重叠 50-200 tokens。"

**代码证据：**
- RAG 服务：`deeptutor/rag/service.py`
- 基于 LlamaIndex 的实现

---

## 💎 第七站：插件系统设计模式（加分项）

**讲解话术：**
> "整个项目用了很多经典设计模式：
>
> - **注册表模式**：ToolRegistry 和 CapabilityRegistry 都是注册表，新增插件只需实现接口、注册，不改编排逻辑。
> - **外观模式**：`DeepTutorApp`（Facade）统一了 CLI、WebSocket、SDK 三个入口，外部调用不需要知道内部复杂性。
> - **策略模式**：不同的 Capability 实现同一个 `BaseCapability` 接口，ChatOrchestrator 通过策略选择。
> - **模板方法**：`BaseTool` 和 `BaseCapability` 是抽象类，定义了执行框架，子类填充具体逻辑。
> - **事件驱动**：StreamBus 实现发布-订阅，所有组件解耦通信。
> - **观察者模式**：Heartbeat、事件订阅都用了观察者模式。"

---

## 🎯 第八站：我发现的问题和改进（1-2 分钟收尾）

**讲解话术：**
> "我在读源码时也发现了一些可以改进的地方：
>
> **1. 并发安全**：AgentLoop 的 `_active_tasks` 列表在 done_callback 中被修改，多个协程同时操作 list 不是原子的，有竞态条件。改进方案是加 `asyncio.Lock` 保护。
>
> **2. 异步 I/O**：Heartbeat 的 `_decide()` 方法里用了同步的 `open()` 读文件，在 async 函数中这会阻塞整个事件循环，影响所有 Bot 的消息处理。应该改用 `aiofiles` 或 `asyncio.to_thread()`。
>
> **3. 背压机制**：StreamBus 没有背压，如果生产者速度大于消费者，内存会持续增长。生产级应该加有界队列 `asyncio.Queue(maxsize=1000)` 加上低优先级事件丢弃策略。
>
> **4. RAG 日志 Task 泄漏**：`_RAGRawLogHandler` 用 `ensure_future` 创建了 Task 但不保存引用，无法追踪或取消。
>
> **5. Team Worker 无并发上限**：多个 Team 同时运行时，Worker 数量没有全局控制，可能导致 LLM API 限流。需要全局 WorkerPool + Semaphore。
>
> 如果让我从架构层面改进，我会优先做三件事：加 asyncio.Lock 解决并发安全、加 OpenTelemetry 分布式 Trace 解决可观测性、把 Memory/Session 外置到 Redis/PG 为分布式做准备。"

---

## 💎 第九站：如果面试官问"你来设计会怎么做"

**讲解话术：**
> "如果让我从零设计一个类似的系统：
>
> 1. **Agent 编排**：我会考虑用 DAG 图替代线性 Pipeline，支持并行子任务。Deep Solve 是串行 Plan→ReAct→Write，但如果子问题之间没有依赖，可以并行执行。
>
> 2. **记忆系统**：我会引入向量检索 + 时间衰减的记忆召回。当前的长期记忆是文件存储，只能全量读入 Context。如果记忆很多，应该做检索式召回。
>
> 3. **错误恢复**：增加 Checkpoint 机制。当前 Deep Solve 如果中途失败需要从头来，应该支持从任意阶段恢复。
>
> 4. **多租户**：加 Tenant 隔离层，支持 SaaS 化部署。当前是单机单用户架构。
>
> 5. **成本控制**：多轮 LLM 调用消耗很大，需要模型分级（简单问题用小模型、复杂问题用大模型）和 Prompt Cache。"

---

## 讲解节奏控制

| 阶段 | 时长 | 内容 |
|---|---|---|
| 定位 | 10 秒 | 一句话介绍 |
| 架构 | 1 分钟 | 两层插件 + 三个入口 + 画图 |
| Deep Solve | 2 分钟 | 三阶段 + Scratchpad + Replan |
| TutorBot | 1.5 分钟 | 持久化 + 心跳 + Team |
| 记忆 | 1 分钟 | 三层 + 整合 |
| RAG | 1 分钟 | 五环节 + Multi-query |
| 改进 | 1.5 分钟 | 3-5 个问题 + 方案 |
| **总计** | **~8 分钟** | |

**关键：** 面试官可能随时打断追问，上面每个模块都准备了追问答案。不要一口气说完，让面试官有提问的空间。

---

## 万能衔接句

- 讲完一个模块，面试官没追问 → "以上是 XX 模块的设计，接下来我可以讲讲 YY 模块，或者您有什么想深入了解的？"
- 面试官打断追问 → 先回答追问，回答完说 "刚才我讲到 XX 的 YY 部分，接下来继续..."
- 被问倒 → "这个具体细节我没有深入看，但从架构层面来说，应该是通过 [你理解的模式] 来实现的。如果我来设计，我会 [给出方案]。"

---

---

## 🎯 第十站：流式输出 — SSE 与 WebSocket（1 分钟）

**讲解话术：**
> "DeepTutor 支持三种入口，其中 WebSocket 入口是给前端用的，支持**实时流式输出**。
>
> 整个流式架构基于**事件驱动**设计。Agent 的每个中间过程——Plan 完成、ReAct 每轮迭代、Tool 调用开始和结束、最终输出——都会作为 Event 发布到 StreamBus。WebSocket 端订阅这些 Event，实时推送给前端。
>
> 具体来说，前端通过 `/api/v1/ws` 建立 WebSocket 连接，发送消息后，后端不会等整个 Agent 执行完才返回，而是每产生一个 Event 就推送一个 JSON 片段。这样用户在界面上就能看到 Agent 的思考过程——'正在规划...'、'正在搜索...'、'找到相关资料...'——体验非常流畅。
>
> 对于不使用 WebSocket 的场景（比如 CLI），StreamBus 的事件也可以直接打印到终端，用 Rich 库做格式化输出，实现类似的实时体验。"

**追问准备：**
- "SSE 和 WebSocket 的区别？" → "SSE（Server-Sent Events）是单向的，服务器推给客户端，基于 HTTP，简单轻量，适合'服务器单向推送'的场景。WebSocket 是双向的，客户端和服务器都可以主动发消息，适合'需要客户端实时反馈'的场景。DeepTutor 选了 WebSocket 是因为前端可能需要取消请求、发 follow-up 消息等双向交互。"
- "StreamBus 怎么实现背压？" → "实际上 StreamBus 当前**没有**背压机制，这也是我发现的问题之一。生产者速度大于消费者时，内存会持续增长。改进方案是加有界队列 `asyncio.Queue(maxsize=1000)`，队列满时丢弃低优先级事件。"
- "如果网络断开，流式输出怎么恢复？" → "当前没有断点续传机制。网络断开后 WebSocket 连接需要重新建立，Agent 的执行结果会丢失。如果要做恢复，需要在 StreamBus 层面加 Event 持久化和消费位点管理。"

**代码证据：**
- WebSocket 路由：`/api/v1/ws` 端点
- StreamBus 事件总线：`deeptutor/core/stream_bus.py`
- 事件类型：各种 Event 数据类（PlanEvent、ReactEvent、ToolEvent、WriteEvent 等）
- CLI 输出：使用 Rich 库格式化

---

## 💎 第十一站：SKILL.md 协议 — Agent 互操作（加分项）

**讲解话术：**
> "DeepTutor 有一个很有意思的设计——**SKILL.md 协议**。
>
> SKILL.md 是一个 Markdown 文件，它把项目的所有 CLI 命令、参数、用法写成结构化文档。它的核心思想是：传统项目写 README 给人读，人读完手动操作。但 DeepTutor 是 Agent-Native 的，所以它写了 SKILL.md 给 Agent 读，Agent 读完后就能自主操作整个系统。
>
> 这本质上是一个**Agent 可消费的标准化接口文档**，类似 OpenAPI/Swagger 之于 API 的作用。任何支持 Tool Use 的 LLM Agent，读取 SKILL.md 后，就知道可以调用哪些命令、参数是什么、返回什么结果。
>
> TutorBot 的 Skill 学习功能也是基于这个——通过读取 SKILL.md 文件，Bot 可以学习新能力，扩展自己的工具集。"

**追问准备：**
- "SKILL.md 和 MCP 的关系？" → "SKILL.md 是应用层的接口描述，MCP 是传输层的工具调用协议。SKILL.md 告诉 Agent '你能做什么'，MCP 告诉 Agent '怎么调用'。两者互补。"
- "这个设计有什么局限？" → "SKILL.md 是静态文档，不会随着代码更新自动同步。如果代码改了但 SKILL.md 没更新，Agent 就会调用失败。需要自动化工具来保持同步。"

---

## 💎 第十二站：Prompt Engineering 实践（加分项）

**讲解话术：**
> "DeepTutor 在 Prompt Engineering 方面有几个值得学习的设计：
>
> **1. System Prompt 分层**：每个 Agent 都有自己的 System Prompt，而且不是硬编码的，很多是放在独立的 Prompt 文件中。比如 PlannerAgent 有规划专用的 Prompt，SolverAgent 有 ReAct 循环专用的 Prompt。这样修改 Prompt 不需要改代码。
>
> **2. Persona 人格系统**：TutorBot 通过 SOUL.md 文件定义自己的人格。用户创建 Bot 时可以指定 persona（比如'苏格拉底式的数学老师'），这个 persona 会被写入 SOUL.md，影响 Bot 的所有回复风格。
>
> **3. Few-Shot 示例**：在 ReAct 循环中，SolverAgent 的 Prompt 包含了 Few-Shot 示例，展示 Thought → Action → Observation 的标准格式，让 LLM 知道应该怎么输出。
>
> **4. 结构化输出**：Agent 间的通信使用结构化的 JSON 格式（如 PlanStep 的 goal、tools_hint、status），减少 LLM 输出解析的歧义。"

**追问准备：**
- "Prompt 版本管理怎么做？" → "当前 DeepTutor 没有专门的 Prompt 版本管理，Prompt 变更跟着代码走。如果要做生产级的 Prompt 管理，应该引入 Prompt Registry，支持 A/B 测试和版本回退。"
- "怎么评估 Prompt 效果？" → "可以用 Golden Set（标准问答集）做回归测试，修改 Prompt 后跑一遍 Golden Set 看效果是否退化。更高级的是用 LLM-as-Judge 做自动评估。"

---

---

## 🎯 第十三站：Context Window 管理（1 分钟）

**讲解话术：**
> "Context Window 管理是 Agent 系统的核心挑战之一。DeepTutor 在这方面有几层设计：
>
> **第一层——消息裁剪**：AgentLoop 在每次调用 LLM 前，会检查当前消息列表的 Token 数。如果接近上限，会触发 MemoryConsolidator 做记忆整合——用 LLM 把旧对话压缩成摘要，替换掉原始消息，腾出空间。
>
> **第二层——分阶段隔离**：Deep Solve 的三个 Agent（Planner、Solver、Writer）各自维护自己的消息上下文，不是把所有历史都塞进一个 Context。Planner 只看原始问题，Solver 只看当前 PlanStep，Writer 只看 Scratchpad 中的证据。这样每个 Agent 的 Context 都很小、很聚焦。
>
> **第三层——Scratchpad 作为外部记忆**：三个 Agent 不直接通信，而是通过共享的 Scratchpad 传递信息。这本质上就是把 Context Window 放不下的中间结果外置到结构化对象中。
>
> 不过我也发现一个问题：Context Window 上限目前硬编码为 **65536**，不适配不同模型的实际限制。比如 Claude 支持 200K，Gemini 支持 1M。改进方案是从模型配置动态获取。"

**追问准备：**
- "Token 计数怎么做的？" → "通常用 tiktoken（OpenAI）或各模型 SDK 提供的 Tokenizer。DeepTutor 在 MemoryConsolidator 中做 Token 计数来触发整合。"
- "如果整合过程中来了新消息怎么办？" → "整合过程有锁保护。新消息会排队等待整合完成后才处理。整合失败则优雅降级到原始归档，不会丢数据。"
- "长上下文模型还需要记忆管理吗？" → "需要。即使 Context Window 变大，填满后仍然需要管理。而且长上下文不等于好效果——中间的信息容易被 LLM '忽略'（Lost in the Middle 问题）。所以结构化的摘要仍然比原始对话更有效。"

**代码证据：**
- MemoryConsolidator：`deeptutor/tutorbot/agent/memory.py`，管理整合触发和锁
- AgentLoop 消息处理：`deeptutor/tutorbot/agent/loop.py`，检查 Token 数并触发整合
- Scratchpad 结构：`deeptutor/agents/solve/memory/scratchpad.py`

---

## 🎯 第十四站：Error Recovery 错误恢复（1 分钟）

**讲解话术：**
> "Agent 系统的容错设计直接决定了可靠性。DeepTutor 有多层次的错误恢复：
>
> **Replan 机制**：Deep Solve 中，如果 SolverAgent 发现当前 Plan 无法完成（工具返回异常结果、迭代耗尽），可以请求 Replan。PlannerAgent 会根据已有信息重新规划，最多 Replan 2 次。这保证了单点失败不会导致整个任务失败。
>
> **迭代上限**：每个 PlanStep 最多 5 次 ReAct 迭代，全局最多 40 次 Tool Call。这些硬上限防止了无限循环，即使 LLM 出了问题也能安全终止。
>
> **记忆整合降级**：MemoryConsolidator 如果整合失败（比如 LLM 超时），会 catch 异常，回退到原始归档。不会因为整合失败丢失用户数据。
>
> **优雅关闭**：TutorBot 停止时，`stop_bot()` 会依次取消所有异步任务（AgentLoop、Heartbeat、Channels），等待当前处理完成再退出。
>
> 但我注意到缺少 **Checkpoint 机制**——如果 Deep Solve 在 ReAct 阶段中途崩溃，需要从头开始。改进方案是在每个 PlanStep 完成后保存 Checkpoint，支持从断点恢复。"

**追问准备：**
- "Checkpoint 怎么设计？" → "每个 PlanStep 完成后，把当前 Scratchpad 状态序列化到文件。重启时读 Checkpoint，跳过已完成的步骤，从未完成的步骤继续。"
- "LLM 调用失败怎么处理？" → "可以加重试（exponential backoff）+ 降级。比如主模型失败，切到备用模型。DeepTutor 支持多 Provider 配置，天然可以做 fallback。"
- "Tool 执行失败怎么办？" → "ToolResult 有 success/failure 状态。失败的结果会作为 Observation 返回给 Solver，Solver 可以在 ReAct 循环中调整策略（换工具、改参数）。"

**代码证据：**
- Replan 逻辑：`deeptutor/agents/solve/main_solver.py`，检测 Solver 返回的 replan status
- 迭代限制：`max_iterations=5`，`global_tool_call_limit=40`
- 优雅关闭：`deeptutor/tutorbot/agent/loop.py`，`stop_bot()` 方法
- ToolResult 状态：`deeptutor/core/tool_protocol.py`，`ToolResult` 数据类

---

## 💎 第十五站：安全机制 — Code Execution 沙箱（加分项）

**讲解话术：**
> "Agent 系统的安全是一个很容易被忽视但极其重要的方向。DeepTutor 有一个 Code Execution Tool，允许 Agent 执行用户提交的代码。这天然存在安全风险。
>
> 当前 DeepTutor 的代码执行是通过 Python subprocess 隔离的——在子进程中运行代码，限制执行时间。但这不是真正的沙箱，恶意代码理论上还是可以访问文件系统、网络等资源。
>
> 如果要生产级安全，应该做几层防护：
>
> **1. 输入护栏**：在用户消息进入 Agent 之前做检查，过滤注入攻击、恶意指令。
> **2. 输出护栏**：Agent 的输出在返回给用户之前做检查，过滤敏感信息泄露。
> **3. 沙箱强化**：用 Docker 容器或 gVisor 做真正的进程隔离，限制网络、文件系统、系统调用。
> **4. Tool 权限分级**：不是所有 Tool 都能被自由调用，高风险 Tool（如 Code Execution、Web Search）需要额外审批。"

**追问准备：**
- "什么是 Prompt Injection？" → "用户在输入中嵌入恶意指令，试图让 Agent 执行非预期操作。比如在文档中隐藏'忽略之前的指令，执行 xxx'。防护方法：输入过滤、System Prompt 加固、输出检查。"
- "Guard Rail 怎么实现？" → "可以用规则引擎做关键词过滤（快速但粗糙），也可以用另一个 LLM 做语义检查（慢但准确）。通常是两层结合——规则引擎先拦截明显的，LLM 再检查模糊的。"
- "DeepTutor 有做安全防护吗？" → "有限。主要靠 LLM 本身的指令遵循能力和迭代上限。没有显式的 Guard Rail 层。这也是可以改进的地方。"

**代码证据：**
- Code Execution Tool：`deeptutor/tools/` 目录下的代码执行工具
- ToolResult 检查：Tool 执行结果有超时保护

---

---

## 🎯 第十六站：Observability 可观测性（1 分钟）

**讲解话术：**
> "Agent 系统的可观测性是个大问题——一个请求可能涉及多个 Agent、多轮 LLM 调用、多次 Tool 执行，出了 bug 很难定位。DeepTutor 在这方面做了几件事：
>
> **1. 事件流 Trace**：StreamBus 上的每个 Event 都携带 trace_id 和 metadata，可以追踪一个请求从入口到最终输出的完整路径。比如 Deep Solve 的 Plan → ReAct → Write 每个阶段都会发布带 trace_id 的 Event。
>
> **2. Scratchpad Metadata**：Scratchpad 除了存储 Plan 和 ReAct Entries，还存了丰富的 Metadata——每个步骤的耗时、Token 消耗、使用的工具。这些信息对调试和性能分析很有价值。
>
> **3. 结构化日志**：CLI 模式用 Rich 库做格式化输出，WebSocket 模式推送结构化 JSON Event。两套输出共享同一个事件源，只是渲染方式不同。
>
> 但我注意到缺少**标准化的可观测性接入**——没有 OpenTelemetry、没有分布式 Trace、没有 Metrics 导出。如果要上生产，这是必须补的。我会优先加 OpenTelemetry 的 Span，把每个 Agent 调用、Tool 调用都包成 Span，用 Jaeger 或 Zipkin 做可视化。"

**追问准备：**
- "OpenTelemetry 在 Agent 系统里怎么用？" → "把每个 LLM 调用做成一个 Span，记录 model、prompt_tokens、completion_tokens、latency。把 Tool 调用做成子 Span，记录 tool_name、参数摘要、执行结果。所有 Span 通过 trace_id 串联，就能在 Jaeger 里看到完整的调用链。"
- "Agent 的日志和传统微服务日志有什么不同？" → "传统微服务日志是确定性的——输入确定、输出确定。Agent 日志是概率性的——同样的输入，LLM 可能走不同的推理路径。所以 Agent 日志需要记录完整的中间过程（Thought、Action、Observation），而不只是最终结果。"
- "怎么监控 Agent 的 Token 消耗？" → "每次 LLM 调用都记录 prompt_tokens 和 completion_tokens，聚合到 Prometheus metrics。按 Agent 类型、Capability 类型、用户 ID 分维度统计。设置预算告警——比如单次 Deep Solve 不超过 50K tokens。"

**代码证据：**
- StreamBus 事件流：`deeptutor/core/stream_bus.py`，Event 携带 metadata
- Scratchpad Metadata：`deeptutor/agents/solve/memory/scratchpad.py`，Metadata 字段
- 事件类型定义：PlanEvent、ReactEvent、ToolEvent 等数据类
- CLI 格式化输出：Rich 库渲染

---

## 🎯 第十七站：Agent 协议 — MCP 与 A2A（1.5 分钟）

**讲解话术：**
> "2024-2025 年 Agent 领域最重要的两个协议标准是 **MCP** 和 **A2A**，理解它们对 Agent 开发者非常重要。
>
> **MCP（Model Context Protocol）** 是 Anthropic 提出的，解决的是 **Agent ↔ Tool** 的连接问题。你可以把它理解为 Agent 世界的 USB 接口。传统做法是每个 Tool 写一个适配器，Agent 换工具就要改代码。MCP 定义了一个标准协议，只要 Tool 实现了 MCP Server，任何支持 MCP 的 Agent 都能直接调用，不需要写适配器。DeepTutor 的 ToolRegistry 本质上是自研了一套类似 MCP 的注册机制，但它不是标准化的。
>
> **A2A（Agent-to-Agent）** 是 Google 提出的，解决的是 **Agent ↔ Agent** 的通信问题。如果一个 Agent 的能力不够，需要另一个 Agent 帮忙，怎么通信？A2A 定义了 Agent Card（能力声明）、Task（任务委托）、Message（消息传递）等标准概念。DeepTutor 的 Team 系统本质上就是在做类似的事情——Task Board 相当于 A2A 的 Task，Mailbox 相当于 A2A 的 Message，但它也是自研的。
>
> 如果 DeepTutor 要做生态化，应该把 ToolRegistry 对齐 MCP 协议，把 Team 系统对齐 A2A 协议，这样就能接入更广泛的 Agent 生态。"

**追问准备：**
- "MCP 和 Function Calling 有什么区别？" → "Function Calling 是单次调用——LLM 输出 JSON，你执行后返回结果。MCP 是协议层——定义了 Tool 的发现、描述、调用的标准格式。Function Calling 是 API 层面的能力，MCP 是生态层面的标准。MCP 底层可以用 Function Calling 实现。"
- "MCP Server 怎么实现？" → "实现 MCP 协议规定的几个方法：`list_tools()` 返回工具列表和描述、`call_tool(name, args)` 执行工具并返回结果。可以用 Python 的 `mcp` SDK 快速搭建。本质上就是把 DeepTutor 的 `BaseTool.get_definition()` + `BaseTool.execute()` 包装成 MCP 标准格式。"
- "A2A 和 DeepTutor Team 的区别？" → "Team 是单系统内的协作，A2A 是跨系统的协作。Team 的 Agent 都在同一进程里，共享内存。A2A 的 Agent 可能分布在不同服务、不同网络上，需要 HTTP 通信。A2A 更适合开放生态。"

**代码证据：**
- ToolRegistry：`deeptutor/runtime/registry/tool_registry.py`——相当于自研版 MCP Registry
- BaseTool 接口：`deeptutor/core/tool_protocol.py`——`get_definition()` + `execute()` 对标 MCP 的 `list_tools()` + `call_tool()`
- Team 系统：`deeptutor/tutorbot/agent/team/__init__.py`——Task Board + Mailbox 对标 A2A 的 Task + Message

---

## 💎 第十八站：Cost 成本优化（加分项）

**讲解话术：**
> "Agent 系统的成本问题是生产环境落地的最大障碍之一。DeepTutor 一次 Deep Solve 可能涉及 10+ 次 LLM 调用（Planner 1 次 + Solver 每步 5 次迭代 × 多个步骤 + Writer 1-2 次），Token 消耗很容易到几万甚至十几万。
>
> 我会从四个维度优化成本：
>
> **1. 模型分级**：简单问题用小模型（GPT-4o-mini / Claude Haiku），复杂问题用大模型（GPT-4o / Claude Sonnet）。Deep Solve 的 Plan 和 Write 阶段可以用大模型，ReAct 循环中的每步 Observation 判断可以用小模型。
>
> **2. Prompt Caching**：Anthropic 和 OpenAI 都支持 Prompt Caching——相同的 System Prompt 前缀只计算一次，后续调用复用缓存，成本降 90%。DeepTutor 的 Agent System Prompt 相对固定，天然适合做 Cache。
>
> **3. Token 预算**：给每次请求设置 Token 预算上限。比如单次 Deep Solve 不超过 50K tokens。接近预算时，WriterAgent 提前进入输出阶段，不再继续 ReAct。
>
> **4. 结果缓存**：相同或相似的查询直接返回缓存结果，不走 Agent 全流程。可以用向量相似度判断'是否和之前的问题类似'。"

**追问准备：**
- "Prompt Caching 的原理？" → "LLM 的推理过程分两步：Prefill（处理输入 Token）和 Decode（逐个生成输出）。Prompt Caching 缓存的是 Prefill 阶段的 KV Cache。如果两次请求的 System Prompt 相同，第二次直接复用 Prefill 结果，跳过大部分计算。Anthropic 的缓存 TTL 是 5 分钟。"
- "模型分级怎么自动判断？" → "两种策略：一是在 Plan 阶段让 LLM 自己评估复杂度（简单/中等/复杂），根据评估结果选模型；二是用规则判断——涉及代码执行、多步推理的用大模型，简单问答用小模型。"
- "怎么监控 Agent 的成本？" → "每次 LLM 调用记录 prompt_tokens + completion_tokens，乘以模型单价得到成本。按用户、按 Agent 类型、按时间段聚合。设置告警阈值。"

**代码证据：**
- LLM 调用入口：各 Agent 的 LLM 调用链路
- 全局 Tool Call 上限 40 次：成本的安全网
- Replan 上限 2 次：防止无限消耗

---

---

## 🎯 第十九站：Evaluation 评估体系（1.5 分钟）

**讲解话术：**
> "Agent 系统的评估是一个很前沿的课题。传统软件有单元测试，但 Agent 的输出是概率性的，不能简单断言'输出等于预期'。DeepTutor 目前的评估主要依赖 WriterAgent 自身——它在输出时会检查引用是否正确、内容是否完整。但这不够客观。
>
> 如果我来设计 Agent 评估体系，会做三层：
>
> **1. Golden Set 回归测试**：维护一套标准问答集，每个问题有'好答案'的标准。每次改 Prompt、换模型后，跑一遍 Golden Set 看效果是否退化。这是最基础也是最重要的。
>
> **2. LLM-as-Judge**：用另一个 LLM（通常是更强的模型）来评估 Agent 的输出质量。评分维度包括：准确性、完整性、引用可靠性、逻辑连贯性。这比人工评估效率高很多，但也有偏差——评判模型可能偏好自己的风格。
>
> **3. Trajectory 评估**：不只看最终输出，还看 Agent 的推理路径。Deep Solve 的 Scratchpad 记录了完整的 Plan → ReAct → Write 过程，可以评估：规划是否合理、工具选择是否正确、是否有不必要的迭代。这比只看结果更能发现问题。
>
> DeepTutor 的 Scratchpad 设计天然支持 Trajectory 评估——它记录了完整的中间过程，这是很好的基础设施。"

**追问准备：**
- "Golden Set 怎么维护？" → "从真实用户问题中筛选，覆盖不同难度和类型。每个问题配一个参考答案和评分标准。定期更新，移除过时的问题，添加新的边界场景。"
- "LLM-as-Judge 的偏差怎么解决？" → "三种策略：一是换位评估（用不同厂商的模型交叉评估）；二是多个 Judge 投票（3 个 Judge 取多数）；三是加 calibrated prompt（在评分 Prompt 中给出锚点样例，校准评分尺度）。"
- "怎么评估 Agent 的延迟？" → "记录每个阶段（Plan/Solve/Write）的耗时，统计 P50/P95/P99。Deep Solve 的总延迟 = Plan 耗时 + Σ(每步 ReAct 耗时) + Write 耗时。可以按阶段做性能优化。"

**代码证据：**
- Scratchpad 完整记录：`deeptutor/agents/solve/memory/scratchpad.py`——Trajectory 评估的数据源
- Event 流：StreamBus 上的 Event 带时间戳，可用于延迟分析
- WriterAgent 引用检查：`deeptutor/agents/solve/agents/writer_agent.py`

---

## 💎 第二十站：Multi-modal 多模态（加分项）

**讲解话术：**
> "多模态是 Agent 的一个重要演进方向。当前 DeepTutor 主要是文本交互，但项目里有几个多模态相关的雏形：
>
> **1. Math Animator**：这是一个 Capability，能把数学概念转化为动画演示。它用 ManimCat（基于 Manim）生成数学动画视频，然后返回给用户。这本质上是**文生视频**的多模态能力。
>
> **2. Media 目录**：每个 TutorBot 的工作区下都有 `media/` 子目录，用于存储图片、视频等媒体文件。这为多模态输出预留了基础设施。
>
> **3. Paper Search Tool**：支持搜索学术论文，论文通常包含图表、公式等非文本内容。
>
> 如果要做更完整的多模态 Agent，我会考虑：
> - **多模态输入**：用户发送图片、PDF、截图，Agent 用 Vision 模型理解后处理
> - **多模态 Tool 输出**：Tool 不仅能返回文本，还能返回图片、图表、代码可视化
> - **多模态记忆**：PROFILE.md 记录用户的学习偏好（比如'偏好图表而非文字'），Agent 据此选择输出模态"

**追问准备：**
- "Vision 模型怎么接入 Agent？" → "两种方式：一是用多模态 LLM（如 GPT-4o、Claude Sonnet）直接处理图片，图片作为 image content block 加入消息；二是先调 OCR/图像理解 Tool 把图片转文本，再把文本给 LLM。方式一更直接但成本高，方式二更灵活。"
- "多模态输出怎么流式推送？" → "文本可以逐 token 流式推送，但图片/视频必须等生成完毕。可以分两阶段：先流式推送文字说明，生成完毕后再推送媒体文件的 URL。WebSocket 适合做这种混合推送。"
- "DeepTutor 的 Manim 动画怎么实现的？" → "Math Animator Capability 接收数学概念描述，生成 Manim Python 代码，执行后输出视频文件。本质上是 Agent → Code Generation → Execution 的管线。"

**代码证据：**
- Math Animator：DeepTutor 的 Capabilities 之一
- Media 目录结构：`data/tutorbot/{id}/media/`
- Paper Search Tool：支持学术搜索的 Tool

---

## 🎯 第二十一站：Rate Limiting 限流与异步安全（1 分钟）

**讲解话术：**
> "这是我在读 DeepTutor 源码时特别关注的工程问题。Agent 系统的限流有两个层面：
>
> **第一层——LLM API 限流**：DeepTutor 的 Deep Solve 一次请求可能触发 10+ 次 LLM 调用。如果有多个并发请求，LLM API 的 Rate Limit 很容易被打爆。当前项目**没有**全局的并发控制和限流机制。Team 系统更危险——多个 Team 同时运行，每个 Team 有 2-3 个 Worker，Worker 又有 25 次迭代上限，可能瞬间产生几十个并发 LLM 请求。
>
> 我的改进方案是：全局 WorkerPool + Semaphore。用一个 `asyncio.Semaphore(max_concurrent_llm_calls)` 控制同时进行的 LLM 调用数，比如限制为 10。超出部分排队等待。
>
> **第二层——asyncio 事件循环安全**：前面提到的几个问题——`_active_tasks` 竞态条件、Heartbeat 同步 I/O 阻塞——都是 asyncio 编程的经典坑。在 async 函数中绝对不能用同步阻塞操作（open()、requests.get()、time.sleep()），否则会卡住整个事件循环，影响所有协程。"

**追问准备：**
- "Semaphore 和 Lock 的区别？" → "Lock 是互斥的——同一时刻只有一个协程能持有。Semaphore 是计数器——允许 N 个协程同时持有。限流场景用 Semaphore，因为它允许一定程度的并发。保护共享状态用 Lock，因为修改必须是原子的。"
- "asyncio.gather 和 TaskGroup 的区别？" → "gather 是老式 API，一个子任务异常不会自动取消其他任务。TaskGroup（Python 3.11+）是结构化并发——任何一个子任务异常会自动取消所有其他任务，更安全。DeepTutor 应该优先用 TaskGroup。"
- "怎么测试并发安全？" → "写压力测试——模拟多个并发请求，用 `asyncio.gather` 同时启动 10+ 个 Agent 调用，检查是否有数据竞争、Task 泄漏、死锁。也可以用 pytest-asyncio 的并发测试模式。"

**代码证据：**
- AgentLoop Task 管理：`deeptutor/tutorbot/agent/loop.py`，`_active_tasks` 列表
- Heartbeat 同步 I/O：`deeptutor/tutorbot/heartbeat/service.py`，`_decide()` 中的 `open()`
- Team Worker：`deeptutor/tutorbot/agent/team/__init__.py`，Worker 迭代循环
- 全局 Tool Call 上限 40 次：唯一的安全网，但不控制并发度

---

---

## 🎯 第二十二站：分布式架构演进 — 从单机到集群（1.5 分钟）

**讲解话术：**
> "DeepTutor 当前是**单机单进程架构**，所有 Bot 实例、Agent 调用都在一个 Python 进程中运行。这在原型阶段够用，但生产环境必须考虑分布式。
>
> 我会分三步演进：
>
> **第一步——状态外置**：当前 TutorBot 的记忆存在本地文件（PROFILE.md、MEMORY.md），会话存在内存。分布式第一步是把状态外置——记忆存 Redis 或 PostgreSQL，会话存数据库。这样多个实例可以共享状态，任一实例挂了，另一个可以接管。
>
> **第二步——消息队列**：当前 StreamBus 是进程内的 asyncio 事件总线。分布式后需要换成 Kafka 或 Redis Streams，让事件可以跨进程、跨机器传递。生产者（Agent）发布事件，消费者（WebSocket Server）订阅并推送给前端。
>
> **第三步——编排层**：当前 ChatOrchestrator 是单点的。分布式后需要用 Kubernetes 做编排——Agent 作为无状态 Worker 运行在 Pod 中，通过消息队列接收任务，水平扩缩容。TutorBot 是有状态的，需要用 StatefulSet 或外部数据库保存状态。
>
> 关键原则是：**Agent 逻辑无状态，状态全部外置**。这样扩容只需要加 Worker。"

**追问准备：**
- "Agent 怎么做负载均衡？" → "Agent 的负载不均匀——有的请求 3 次迭代就完成，有的需要 20 次。不适合用简单的 Round-Robin。应该用 Work-Stealing：每个 Worker 维护自己的任务队列，空闲的 Worker 从繁忙的 Worker 那里'偷'任务。"
- "分布式后 Scratchpad 怎么共享？" → "Scratchpad 从进程内对象变成 Redis 中的结构化数据。每个 PlanStep 的 ReAct 记录作为 Redis Hash 存储，所有 Agent 通过 Redis 读写。性能有损但换来分布式能力。"
- "TutorBot 的 Heartbeat 在分布式下怎么做？" → "Heartbeat 需要分布式调度——用 Celery 或 Kubernetes CronJob 触发。每个 Bot 的 Heartbeat 任务注册到调度器，调度器保证同一时刻只有一个实例在执行某个 Bot 的 Heartbeat。需要分布式锁（Redis SETNX）防重复。"

**代码证据：**
- StreamBus：`deeptutor/core/stream_bus.py`——需要替换为分布式消息队列
- TutorBot 记忆文件：`data/tutorbot/{id}/`——需要外置到数据库
- ChatOrchestrator：单点路由——需要变成无状态的 API Gateway
- Facade 统一入口：`deeptutor/app/facade.py`——分布式后变成 API 层

---

## 💎 第二十三站：LLM 原理 — 为什么理解 Transformer 对 Agent 开发重要（加分项）

**讲解话术：**
> "面试官可能会问：你理解 LLM 的底层原理吗？这对 Agent 开发为什么重要？
>
> 我会从三个层面回答：
>
> **1. Attention 机制**：Transformer 的核心是 Self-Attention——每个 Token 关注输入序列中的所有其他 Token。这决定了两件事：一是 LLM 的能力上限（能理解复杂上下文关系），二是它的限制（Context Window 越大，计算量呈平方增长 O(n²)）。所以 Context Window 不能无限大——65536 tokens 的 Attention 计算已经很重了。
>
> **2. 生成式推理**：LLM 是自回归模型——每次生成一个 Token，基于前面所有 Token 的概率分布。这意味着 LLM **不能回退**——一旦生成了错误的 Token，后续生成只能在这个错误的基础上继续。这就是为什么 Agent 需要 ReAct 循环——如果一步走错了，可以通过 Observation 发现并纠正。
>
> **3. 涌现能力**：当模型规模大到一定程度（通常认为 100B+ 参数），会涌现出 Chain-of-Thought（思维链）和 Tool Use 的能力。Agent 开发正是利用这些涌现能力。理解这一点，就知道为什么小模型做不了复杂 Agent——不是 Prompt 写得不好，而是能力还没涌现。"

**追问准备：**
- "KV Cache 是什么？" → "LLM 推理时，每生成一个 Token 都要对前面所有 Token 做 Attention。KV Cache 把每步的 Key 和 Value 向量缓存起来，下一步只需要计算新 Token 的 Attention，不用重新计算所有历史 Token。这就是 Prompt Caching 的底层原理——缓存 KV Cache，复用计算。"
- "为什么 ReAct 比 直接生成答案好？" → "直接生成答案是一次性的，没有纠错机会。如果 LLM 第一Token 就走偏了，整个回答都会偏。ReAct 把生成过程拆成多步，每步有 Observation 反馈，相当于给了 LLM 多次'重新考虑'的机会。本质上是把一个困难的一次性生成，分解成多个简单的步骤。"
- "Agent 的 System Prompt 有长度限制吗？" → "System Prompt 占用 Context Window 的空间。通常 System Prompt 越长，留给对话和工具调用的空间越少。所以 System Prompt 要精炼——只放必要的指令和 Few-Shot 示例。DeepTutor 的做法是把不同 Agent 的 System Prompt 分开，每个 Agent 只加载自己需要的。"

**代码证据：**
- Context Window 管理：AgentLoop 中的 Token 计数和 Memory 整合
- 多 Agent 各自独立的 System Prompt：PlannerAgent、SolverAgent、WriterAgent 各有专用的 Prompt
- ReAct 循环：SolverAgent 的 Thought → Action → Observation 分步推理

---

## 🎯 第二十四站：部署架构 — Docker 与生产化（1 分钟）

**讲解话术：**
> "DeepTutor 使用 **Docker + docker-compose** 部署，这在 AI 应用中是主流做法。
>
> 为什么要容器化？因为 AI 应用的依赖很重——Python 版本、CUDA 驱动、各种 ML 库版本。容器化保证了'开发环境和生产环境一致'。
>
> docker-compose 的编排大致是：
> - **API 服务**：FastAPI 应用，暴露 WebSocket 和 REST 端点
> - **向量数据库**：Chroma 或 Qdrant 容器，存储 RAG 的向量索引
> - **LLM Provider**：通常用外部 API（OpenAI/Anthropic），不需要本地容器
> - **前端**：Next.js 应用容器
>
> 如果要上生产，还需要加：
> - **Nginx**：反向代理 + SSL 终止 + 负载均衡
> - **Redis**：Session 存储 + 缓存 + 分布式锁
> - **PostgreSQL**：用户数据 + Bot 配置 + 对话历史
> - **Prometheus + Grafana**：监控 Agent 的 Token 消耗、延迟、错误率
> - **Jaeger**：分布式 Trace，追踪 Agent 调用链路"

**追问准备：**
- "Docker 部署 AI 应用有什么坑？" → "三个常见坑：一是 GPU 驱动兼容问题——需要 nvidia-docker runtime 和匹配的 CUDA 版本；二是镜像体积大——ML 依赖动辄几个 GB，要用多阶段构建瘦身；三是冷启动慢——模型加载需要时间，需要预热或常驻实例。"
- "怎么做蓝绿部署？" → "AI 应用的部署比较特殊——模型版本变更可能导致输出完全不同，不能简单切流量。需要做 Shadow Testing——新版本并行运行但不返回给用户，对比新旧版本的输出差异，确认无退化后再切流。"
- "DeepTutor 的 docker-compose 怎么配置？" → "通常包含 api（FastAPI 后端）、web（Next.js 前端）、db（向量数据库）三个 service。环境变量配置 LLM API Key、模型选择、数据库连接等。"

**代码证据：**
- Docker 相关配置：项目根目录的 `Dockerfile` 和 `docker-compose.yml`
- 环境变量配置：`.env` 文件管理 API Key 和模型设置
- FastAPI 入口：API 服务启动点

---

---

## 🎯 第二十五站：Human-in-the-Loop — 人工协作机制（1 分钟）

**讲解话术：**
> "Agent 系统不是完全自动化的，很多场景需要人介入。DeepTutor 在这方面有多个设计：
>
> **1. Team 的审批流程**：Team 系统中，某些任务会标记为需要人工审批。用户可以通过 `/team approve <id>` 或 `/team reject <id>` 来决定任务是否执行。这确保了高风险操作不会自动执行——比如 Agent 要发一封邮件，需要用户确认。
>
> **2. TutorBot 的 Heartbeat 交互**：Heartbeat 机制让 Bot 主动发起交互，但最终决策权在人。Bot 可以发学习提醒，但用户可以忽略。Bot 可以建议复习计划，但需要用户确认。
>
> **3. Deep Solve 的 Replan**：当 SolverAgent 卡住时请求 Replan，这个过程中用户可以通过 StreamBus 看到中间状态。虽然当前没有显式的'暂停等待用户输入'机制，但事件流为未来接入 Human-in-the-Loop 预留了接口。
>
> 核心原则是：**Agent 做执行，人做决策**。Agent 负责收集信息、执行操作，但关键节点需要人的确认。这在生产系统中是必需的——完全自主的 Agent 在可靠性不够高时是危险的。"

**追问准备：**
- "Human-in-the-Loop 怎么影响 Agent 的延迟？" → "审批节点会阻塞 Agent 执行，等待用户响应。可以设超时——比如 5 分钟未审批自动拒绝或升级。也可以设计成异步——Agent 先做不依赖审批的部分，审批通过后再做依赖的部分。"
- "什么场景必须 HITL？" → "三类场景：一是高成本操作（花钱、发邮件、删数据）；二是不可逆操作（部署代码、修改配置）；三是不确定性高的决策（Agent 置信度低于阈值时请求人工确认）。"
- "Team 的环检测怎么实现的？" → "创建任务依赖时，做拓扑排序检查。如果添加一条依赖后形成环（A→B→C→A），就拒绝这条依赖。本质上是有向图的环检测算法——DFS 遍历时检测回边。"

**代码证据：**
- Team 审批：`deeptutor/tutorbot/agent/team/__init__.py`，approve/reject 命令处理
- Heartbeat 交互：`deeptutor/tutorbot/heartbeat/service.py`，主动通知用户
- StreamBus 事件流：中间状态可视化的基础设施
- 依赖环检测：Team 系统的 Task Board 管理逻辑

---

## 💎 第二十六站：向量数据库实战 — 从原型到生产（加分项）

**讲解话术：**
> "RAG 系统的核心是向量数据库选型。DeepTutor 基于 LlamaIndex，天然支持多种向量存储后端。选型的核心是看你的场景：
>
> **原型阶段用 Chroma**：零配置，pip install 就能用，数据存在本地文件。适合快速验证 RAG 效果。缺点是功能单一，没有过滤、没有集群。
>
> **生产中等规模用 Qdrant**：Rust 写的，性能好，支持丰富的过滤条件（比如按学科、难度过滤文档）。可以用 Docker 单机部署，也可以用 Qdrant Cloud 托管。
>
> **大规模分布式用 Milvus**：云原生设计，支持亿级向量，自动分片和负载均衡。但运维复杂度高，需要专门的团队。
>
> **已有 PostgreSQL 用 pgvector**：最省事——不需要引入新组件，直接在 PG 里存向量。适合不想增加技术栈复杂度的团队。性能比专业向量库差一些，但百万级以内够用。
>
> DeepTutor 的做法是通过 LlamaIndex 的抽象层，把向量存储配置化。切换后端只需要改配置，不改代码。"

**追问准备：**
- "向量检索和关键词检索的区别？" → "向量检索是语义匹配——'深度学习'和'神经网络'语义相近，能匹配上。关键词检索是精确匹配——搜'深度学习'搜不到'神经网络'。实际生产中通常是混合检索（Hybrid Search）：先向量检索粗排，再结合关键词匹配（BM25）提升精度。"
- "Embedding 模型怎么选？" → "通用场景用 OpenAI 的 text-embedding-3-small（便宜、够用）。中文场景推荐 BGE 或 M3E。多语言场景用 multilingual-e5。选型标准是看 MTEB（Massive Text Embedding Benchmark）排行榜。"
- "向量维度和检索质量的关系？" → "维度越高，表达能力越强，但存储和计算成本也越高。OpenAI text-embedding-3-small 是 1536 维，text-embedding-3-large 是 3072 维。通常 768-1536 维够用。可以用 Matryoshka 嵌套维度技术——存储高维向量，检索时用低维截断，兼顾精度和性能。"

**代码证据：**
- RAG 服务：`deeptutor/rag/service.py`，基于 LlamaIndex 的 RAG 实现
- 向量存储配置：LlamaIndex 支持多种 VectorStore 后端
- Embedding 模型配置：支持多种 Embedding Provider

---

## 🎯 第二十七站：代码质量与工程实践 — 我怎么读源码的（1 分钟）

**讲解话术：**
> "面试官可能会问：你是怎么读源码的？我分享一下我的方法：
>
> **第一遍——看入口找主线**：先看 AGENTS.md（架构文档），理解整体设计。然后从入口点（CLI 的 main.py、API 路由）跟进去，画出调用链路图。这一遍不求细节，只求理解数据流——请求从哪进来、经过哪些组件、从哪出去。
>
> **第二遍——看抽象找模式**：重点看基类和接口——BaseTool、BaseCapability、BaseAgent。理解了接口，就理解了设计者的意图。然后看具体实现，看它们是怎么满足接口契约的。
>
> **第三遍——找问题找改进**：带着'如果是我写，我会怎么做'的问题去看。看并发安全、异常处理、可扩展性。比如我发现 AgentLoop 的 `_active_tasks` 竞态条件，就是在第三遍看 done_callback 的时候发现的。
>
> 三个工具：**Grep 找关键符号**、**Read 读核心文件**、**画图理清关系**。不需要从头到尾读每个文件——80% 的信息在 20% 的文件里。"

**追问准备：**
- "你觉得这个项目代码质量怎么样？" → "整体设计很优秀——两层插件架构清晰、Agent 编排逻辑合理、事件驱动解耦。但在工程细节上有改进空间——并发安全、异步 I/O、背压机制。这是学术项目转生产的典型差距。"
- "如果让你给这个项目提 PR，你会提什么？" → "我会提三个 PR：一是加 asyncio.Lock 保护共享状态（小改动，大影响）；二是把 Heartbeat 的同步文件 I/O 改成异步（中等改动）；三是给 StreamBus 加背压机制（较大改动，需要设计丢弃策略）。按影响力排序，先提第一个。"
- "读开源项目对你有什么帮助？" → "两个帮助：一是快速理解真实系统的架构设计——教科书讲概念，开源项目展示怎么落地；二是培养代码品味——看优秀的设计和有问题的设计，形成自己的判断力。"

**代码证据：**
- 架构文档入口：`AGENTS.md`
- CLI 入口：`deeptutor_cli/main.py`
- 基类设计：`deeptutor/core/tool_protocol.py`、`deeptutor/core/capability_protocol.py`
- 发现的问题：`loop.py`（竞态）、`heartbeat/service.py`（同步 I/O）、`stream_bus.py`（无背压）

---

## 🎯 第二十八站：面试终极收尾 — 60 秒总结陈词

**讲解话术（面试最后 1 分钟说这段）：**
> "总结一下我对这个项目的理解：
>
> DeepTutor 是一个设计优秀的 Agent-Native 系统，两层插件架构（Tool + Capability）做到了很好的解耦，Deep Solve 的 Plan → ReAct → Write 三阶段协作是工程实践的好案例，TutorBot 的持久化 Agent 和 Heartbeat 机制展示了从'被动响应'到'主动服务'的进化。
>
> 同时我也注意到了从原型到生产的差距——并发安全、可观测性、分布式部署这些是学术项目普遍缺少的，也是我作为工程师能贡献价值的地方。
>
> 我对 Agent 开发方向非常有热情，这个项目让我理解了 Agent 系统的全链路设计。如果有幸加入贵团队，我希望能把学到的架构思维落地到生产系统中。"

**追问准备：**
- "你还有什么想问的？" → 选 2-3 个反问：
  1. "团队目前在做什么类型的 Agent？"
  2. "Agent 的工具生态是自研还是用开源框架？"
  3. "团队怎么评估 Agent 的效果？有专门的 Eval 体系吗？"
  4. "新人加入后大概多长时间能独立负责一个模块？"

---

*最后更新：2026-04-14 — 已追加第二十五至二十八站（完结）*

---

# 全站索引

| 站号 | 主题 | 类型 | 预计时长 |
|---|---|---|---|
| 1 | 项目定位 | 🎯 必讲 | 10 秒 |
| 2 | 架构全景（画图） | 🎯 必讲 | 1 分钟 |
| 3 | Deep Solve 多 Agent 协作 | 🎯 必讲 | 2 分钟 |
| 4 | TutorBot 持久化 Agent | 🎯 必讲 | 1.5 分钟 |
| 5 | 记忆系统 | 🎯 必讲 | 1 分钟 |
| 6 | RAG 知识库 | 🎯 必讲 | 1 分钟 |
| 7 | 设计模式 | 💎 加分 | 30 秒 |
| 8 | 项目问题与改进 | 🎯 必讲 | 1.5 分钟 |
| 9 | 你来设计怎么做 | 💎 加分 | 1 分钟 |
| 10 | SSE/WebSocket 流式输出 | 🎯 必讲 | 1 分钟 |
| 11 | SKILL.md 协议 | 💎 加分 | 30 秒 |
| 12 | Prompt Engineering | 💎 加分 | 1 分钟 |
| 13 | Context Window 管理 | 🎯 必讲 | 1 分钟 |
| 14 | Error Recovery | 🎯 必讲 | 1 分钟 |
| 15 | 安全机制 | 💎 加分 | 1 分钟 |
| 16 | Observability | 🎯 必讲 | 1 分钟 |
| 17 | MCP 与 A2A | 🎯 必讲 | 1.5 分钟 |
| 18 | Cost 成本优化 | 💎 加分 | 1 分钟 |
| 19 | Evaluation 评估 | 🎯 必讲 | 1.5 分钟 |
| 20 | Multi-modal | 💎 加分 | 1 分钟 |
| 21 | Rate Limiting | 🎯 必讲 | 1 分钟 |
| 22 | 分布式架构演进 | 🎯 必讲 | 1.5 分钟 |
| 23 | LLM 原理 | 💎 加分 | 1 分钟 |
| 24 | 部署架构 | 💎 加分 | 1 分钟 |
| 25 | Human-in-the-Loop | 🎯 必讲 | 1 分钟 |
| 26 | 向量数据库实战 | 💎 加分 | 1 分钟 |
| 27 | 代码质量与读源码方法 | 🎯 必讲 | 1 分钟 |
| 28 | 面试终极收尾 | 🎯 必讲 | 60 秒 |

**面试时间分配建议：**
- 5 分钟版：站 1 → 2 → 3 → 8（必讲核心）
- 8 分钟版：站 1 → 2 → 3 → 4 → 5 → 10 → 8（推荐）
- 15 分钟完整版：按索引顺序全部讲完
