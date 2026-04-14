# Agent 开发面试 — 完整速记卡（唯一文件）

> 面试前过这一份文件就够了。按顺序读，每张卡 2-3 分钟，总共约 40 分钟。
> 重点标记 ★ 的必背，没有 ★ 的选背。

---

# 第一部分：开场与项目介绍

---

## 卡片 1：自我介绍（1 分钟）

> "我是 [你的名字]，最近在研究 AI Agent 开发方向。
> 我深入分析了 DeepTutor 这个开源项目（17.7k stars，港大 HKUDS），
> 重点研究了它的**两层插件架构**（Tool + Capability）、**多 Agent 协作系统**（Deep Solve）、
> 和**持久化 Agent**（TutorBot）的设计。
> 我对 Agent 编排、RAG 系统、工具调用这些方向特别感兴趣，
> 觉得 Agent 开发是 AI 应用最有潜力的方向。"

**注意：** 不要超过 1 分钟。留时间让面试官提问。

---

## ★ 卡片 2：项目一句话描述

> "DeepTutor 是港大 HKUDS 实验室开源的 Agent-Native 个性化学习助手（17.7k stars），
> 核心架构是两层插件模型（Tools + Capabilities），有三个入口：CLI、WebSocket API、Python SDK。
> 项目包含多 Agent 协作系统（Deep Solve）、持久化 TutorBot、RAG 知识库、引导式学习等模块。"

---

## ★ 卡片 3：架构全景（面试必画）

```
Entry Points:  CLI (Typer)  |  WebSocket /api/v1/ws  |  Python SDK
                    ↓                   ↓                   ↓
              ┌─────────────────────────────────────────────────┐
              │              ChatOrchestrator                    │
              │   routes to ChatCapability (default)             │
              │   or a selected deep Capability                  │
              └──────────┬──────────────┬───────────────────────┘
                         │              │
              ┌──────────▼──┐  ┌────────▼──────────┐
              │ ToolRegistry │  │ CapabilityRegistry │
              │  (Level 1)   │  │   (Level 2)        │
              └──────────────┘  └────────────────────┘
```

**Level 1 — Tools**：原子级单功能工具（RAG、Web Search、Code Execution、Reason、Brainstorm、Paper Search）

**Level 2 — Capabilities**：多步骤 Agent Pipeline（Chat、Deep Solve、Deep Question、Deep Research、Math Animator）

**追问准备：**
- 为什么两层？→ Tool 可被多个 Capability 复用，解耦
- 为什么三个入口？→ CLI 给开发者，WebSocket 给前端，SDK 给集成

---

## 卡片 4：讲项目的万能模板

**开场（10 秒）：**
> "DeepTutor 是一个 Agent-Native 个性化学习助手，17.7k stars。"

**架构（30 秒）：**
> "核心是**两层插件架构**：
> - **Tool 层**：原子工具（RAG、Web Search、Code Execution、Reason）
> - **Capability 层**：多步骤 Agent Pipeline（Chat、Deep Solve、Deep Research）
> - **三个入口**：CLI、WebSocket API、Python SDK，通过 Facade 统一"

**亮点（30 秒）：**
> "我最有感触的是**多 Agent 协作**（Deep Solve）：
> Plan → ReAct → Write 三阶段，有共享 Scratchpad 和 Replan 容错。
> 还有 **TutorBot** 是持久化 Agent，有心跳机制能主动找用户。"

**收尾（10 秒）：**
> "我也发现了一些可以改进的地方，比如并发安全和背压机制。"

---

# 第二部分：核心模块详解

---

## ★ 卡片 5：Deep Solve 三阶段

```
Plan（规划）→ ReAct（求解）→ Write（输出）
```

**PlannerAgent：**
- 把复杂问题分解为有序步骤（PlanStep）
- 每个 PlanStep 有 goal、tools_hint、status

**SolverAgent：**
- 对每个 PlanStep 做 ReAct 循环（Thought → Action → Observation）
- 最多 **5 次**迭代，支持 Replan（最多 **2 次**）
- 每步有 self_note 做自我反思

**WriterAgent：**
- 从积累的证据生成最终回答，强制标注引用来源

**共享内存：Scratchpad**
- 包含 Plan、ReAct Entries、Sources、Metadata
- 所有 Agent 读写同一个 Scratchpad（Single Source of Truth）

**关键数字：** 5 次 ReAct、2 次 Replan、40 次全局 tool call 上限

**一句话总结：**
> "Plan 把复杂问题拆解，ReAct 逐步求解，Write 综合输出。
> 关键设计是 Scratchpad 共享内存和 Replan 容错机制。"

---

## ★ 卡片 6：TutorBot 持久化 Agent

> "TutorBot 不是一个 Chatbot，而是一个持久化的自主 Agent：
> 独立工作区、独立记忆、独立人格（SOUL.md）、心跳机制（主动发起交互）、
> 支持多渠道（Telegram/Discord/Slack 等 11 个）。"

**生命周期：**
```
创建（独立目录 data/tutorbot/{id}/）
→ 启动（AgentLoop + MessageBus + Channels + Heartbeat）
→ 运行（异步消息处理，最多 40 次 tool call）
→ 停止（优雅关闭所有异步任务）
```

**五大核心特性：**
1. **独立工作区**：完全隔离，有自己的 SOUL.md（人格）、记忆、会话
2. **Heartbeat 心跳**：LLM 驱动决策（skip or run），能主动发起交互
3. **多渠道接入**：Telegram、Discord、Slack 等 11 个，统一 InboundMessage
4. **Team 协作**：`/team <goal>` 创建 2-3 个 AI 队友，Task Board + Mailbox + approve/reject
5. **Skill 学习**：通过 SKILL.md 文件学习新能力

---

## ★ 卡片 7：记忆系统

```
短期：Context Window（当前对话）
中期：Session（数据库/文件，按 session 关联）
长期：PROFILE.md + MEMORY.md + SUMMARY.md（跨会话持久化）
```

**整合机制：**
- Token 接近上限 → 触发 LLM 生成摘要 → 替换旧对话
- MemoryConsolidator 管理，有锁保护
- 整合失败 → 优雅降级到原始归档

**跨 Bot 共享：** 通过 `data/memory/` 共享目录

**面试亮点：** Markdown 格式存储，人可读、Agent 可消费

---

## ★ 卡片 8：插件系统（Tool Use）

**Tool 协议层：**
```python
class BaseTool(ABC):
    def get_definition(self) -> dict: ...      # OpenAI function-calling 兼容
    async def execute(self, **kwargs) -> ToolResult: ...
```

**Capability 协议层：**
```python
class BaseCapability(ABC):
    manifest: CapabilityManifest   # 名称、描述、阶段、工具依赖
    async def execute(self, request) -> AsyncIterator[Event]: ...
```

**注册发现：**
- ToolRegistry：单例模式，load_builtins() 加载内置工具
- CapabilityRegistry：管理所有 Capability
- 支持别名映射和动态加载

**执行管线：**
```
ToolRegistry.invoke() → 解析别名、合并参数
→ SolveToolRuntime → 注入运行时上下文（API keys、输出目录）
→ BaseTool.execute() → 返回标准化 ToolResult
→ ToolEventSink → 进度追踪
```

---

## 卡片 9：Heartbeat 心跳机制

**两阶段设计：**
1. **决策阶段** `_decide()`：读 HEARTBEAT.md → LLM 决定 "skip" or "run"
2. **执行阶段** `_tick()`：仅在 LLM 决定 "run" 时触发 → 执行 → 评估 → 通知

**亮点：** 不是简单定时任务，是 LLM 驱动的智能决策

---

## 卡片 10：Team 团队协作

```
/team <goal>  → 创建 2-3 个 AI 队友

Task Board（看板）：
  - pending → in_progress → completed
  - 支持任务依赖（有环检测）
  - 支持人工审批（approve/reject）

Mailbox（邮箱）：
  - 队友间异步通信

Worker Runtime：
  - 自主认领任务，最多 25 次迭代
```

**亮点：** 去中心化协作 + Human-in-the-loop 审批

---

# 第三部分：通用 Agent 知识

---

## ★ 卡片 11：必背 8 个概念

**每个用一句话说清楚：**

1. **ReAct** = Reasoning + Acting 交替，Agent 的核心推理模式
2. **Function Calling** = LLM 输出结构化 JSON 调用工具，Agent 的手脚
3. **RAG** = 检索增强生成（解析→分块→向量化→检索→生成），让 Agent 获取外部知识
4. **Memory** = 三层：短期(Context Window) + 中期(Session) + 长期(Profile/文件)
5. **MCP** = Anthropic 的工具连接标准协议（Agent → Tool）
6. **A2A** = Google 的 Agent 间通信协议（Agent ↔ Agent）
7. **Prompt Caching** = 相同 Prompt 前缀复用计算，成本降 90%
8. **Guard Rail** = Agent 输出的后置安全检查（输入护栏 + 输出护栏）

---

## 卡片 12：RAG 五环节

```
文档 → 解析(Parsing) → 分块(Chunking) → 向量化(Embedding) → 检索(Retrieval) → 生成(Generation)
```

**关键优化：**
- 分块：Fixed / Semantic / Recursive，通常 256-1024 tokens，重叠 50-200
- 检索：向量检索（粗排 Top-100）→ Reranker 精排 Top-10
- DeepTutor 基于 LlamaIndex，支持多种 Provider

---

## 卡片 13：向量数据库选型

| 场景 | 选择 | 原因 |
|---|---|---|
| 原型/MVP | Chroma | 零配置 |
| 生产中等规模 | Qdrant | 高性能+过滤 |
| 大规模分布式 | Milvus | 云原生、亿级 |
| 已有 PostgreSQL | pgvector | 最省事 |

---

## 卡片 14：Agent 框架对比

| 框架 | 定位 | 选它的理由 |
|---|---|---|
| **LangChain** | 通用编排 | 最全面，快速搭建 |
| **LlamaIndex** | RAG 专用 | 知识库场景最强 |
| **AutoGen** | 多 Agent 对话 | 微软出品 |
| **CrewAI** | 角色扮演团队 | 模拟团队协作 |
| **nanobot** | 超轻量引擎 | 嵌入式场景 |

**DeepTutor 的选择：** LlamaIndex（RAG）+ nanobot（Agent 引擎），没选 LangChain

---

## 卡片 15：设计模式速查

| 模式 | 应用 |
|---|---|
| 注册表模式 | ToolRegistry、CapabilityRegistry |
| 外观模式 | DeepTutorApp（Facade）统一三个入口 |
| 事件驱动 | StreamBus 异步事件流 |
| 策略模式 | 不同 Capability 实现同一接口 |
| 模板方法 | BaseTool、BaseCapability 抽象类 |

---

# 第四部分：批判性思维与改进

---

## ★ 卡片 16：项目不足（必说 3 个）

**按优先级：**

> **1. 并发安全（展示基本功）**
> "AgentLoop 的 `_active_tasks` 的 done_callback 有竞态条件，
> 多协程同时操作 list 不是原子的。改进：加 `asyncio.Lock`。"

> **2. 异步 I/O（展示功底）**
> "Heartbeat 在 `async def _decide()` 中用了同步 `open()` 读取文件，
> 会阻塞整个事件循环，影响所有 Bot 的消息处理。
> 改进：改用 `aiofiles` 或 `asyncio.to_thread()`。"

> **3. 背压机制（展示架构思维）**
> "StreamBus 没有背压机制，生产者速度 > 消费者速度时内存溢出。
> 改进：加有界队列 `asyncio.Queue(maxsize=1000)` + 低优先级事件丢弃。"

**追问"如果要你改进"：**
> "三件事：1) asyncio.Lock 保护共享状态 2) OpenTelemetry 分布式 Trace 3) Memory/Session 外置到 Redis/PG"

---

## 卡片 17：更多改进点（备用）

4. **RAG 日志 Task 泄漏**：`_RAGRawLogHandler` 用 `ensure_future` 创建 Task 但不保存引用，无法追踪或取消
5. **Team Worker 无并发上限**：多个 Team 同时运行 → LLM API 限流爆炸。改进：全局 WorkerPool + Semaphore
6. **Context Window 硬编码 65536**：不适配 Claude(200K)、Gemini(1M)。改进：从模型配置动态获取
7. **Replan 不保留已完成步骤**：重新规划后可能重复执行。改进：传入已完成步骤摘要

---

# 第五部分：代码题与现场应对

---

## ★ 卡片 18：现场写代码（3 种必背模板）

**题 1：ReAct Agent 循环**
```python
async def run(self, question: str) -> str:
    messages = [{"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": question}]
    for i in range(self.max_iterations):
        response = await self.llm.call(messages)
        messages.append({"role": "assistant", "content": response})
        parsed = json.loads(response)
        if parsed.get("action") == "done":
            return parsed["answer"]
        tool_name = parsed["action"]
        tool_args = parsed.get("action_input", {})
        result = await self.tools[tool_name](**tool_args)
        messages.append({"role": "user", "content": f"Observation: {result}"})
    return "Reached iteration limit."
```

**题 2：流式输出（SSE）**
```python
@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    async def generate():
        async for chunk in llm.stream(request.messages):
            yield f"data: {json.dumps({'text': chunk})}\n\n"
        yield f"data: {json.dumps({'type': 'done'})}\n\n"
    return StreamingResponse(generate(), media_type="text/event-stream")
```

**题 3：带重试的工具调用**
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=30))
async def call_tool(name, **kwargs):
    return await tools[name](**kwargs)
```

---

## ★ 卡片 19：万能应对公式

**遇到不会的问题：**
> "这个具体细节我没有深入研究，但从 Agent 架构的角度来看，
> 它应该涉及到 [相关概念]。如果让我来设计，
> 我会 [给出你的方案]，因为 [原因]。"

**遇到需要时间思考的问题：**
> "这个问题很好，让我想一下..."（沉默 3-5 秒是完全正常的）

**遇到被质疑：**
> "你说得对，我之前的理解确实不够准确。更准确地说应该是..."
（不要硬杠，灵活调整）

**遇到完全没听过的技术：**
> "这个技术我还没有接触过，但听起来和 [你熟悉的技术] 有相似之处，
> 是不是解决 [某个问题] 的？"

---

# 第六部分：临场策略

---

## 卡片 20：反问面试官（选 2-3 个）

1. "团队目前在做什么类型的 Agent？"
2. "Agent 的工具生态是自研还是用开源框架？"
3. "新人加入后大概多长时间能独立负责一个模块？"
4. "团队怎么评估 Agent 的效果？有专门的 Eval 体系吗？"

---

## 卡片 21：学习方法

> "三个渠道：
> 1. 研读开源项目源码（DeepTutor 的 Agent 架构）
> 2. 关注行业动态（OpenAI/Anthropic/Google 的 Agent 发布）
> 3. 动手实践（用 Claude Code 等工具体验 Agent 能力边界）"

---

## 卡片 22：技术栈速查

| 层次 | 技术 |
|---|---|
| 后端 | FastAPI + Python 3.11+ |
| 前端 | Next.js 16 + React 19 |
| Agent 引擎 | nanobot（HKUDS 自研） |
| RAG | LlamaIndex |
| CLI | Typer + Rich |
| 异步 | asyncio + StreamBus |
| 部署 | Docker + docker-compose |

---

## 卡片 23：关键代码位置（面试官可能追问）

| 模块 | 路径 |
|---|---|
| 架构总览 | `AGENTS.md` |
| Tool 协议 | `deeptutor/core/tool_protocol.py` |
| Capability 协议 | `deeptutor/core/capability_protocol.py` |
| Deep Solve 主控 | `deeptutor/agents/solve/main_solver.py` |
| Planner Agent | `deeptutor/agents/solve/agents/planner_agent.py` |
| Solver Agent | `deeptutor/agents/solve/agents/solver_agent.py` |
| Scratchpad | `deeptutor/agents/solve/memory/scratchpad.py` |
| TutorBot Loop | `deeptutor/tutorbot/agent/loop.py` |
| Heartbeat | `deeptutor/tutorbot/heartbeat/service.py` |
| Team | `deeptutor/tutorbot/agent/team/__init__.py` |
| Memory | `deeptutor/tutorbot/agent/memory.py` |
| Tool Registry | `deeptutor/runtime/registry/tool_registry.py` |
| StreamBus | `deeptutor/core/stream_bus.py` |

---

# 最终提醒

- ★ 标记的卡片（2、3、5、6、7、8、11、16、18、19）**必背**
- 其余卡片选背，时间不够可以跳过
- **说清楚比说得多更重要**：一个概念讲清楚胜过十个概念泛泛而谈
- **展示工程思维**：不是背概念，而是"如果我来做，我会怎么做"
- **保持自信**：你准备了 96 道面试题 + 23 张速记卡，比绝大多数候选人都充分

**祝你明天面试顺利！**
