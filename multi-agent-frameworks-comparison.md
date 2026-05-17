# 多 Agent 调度框架对比

> 横向对比主流多 Agent 编排框架，帮助理解调度模型差异与技术选型。

## 背景：A2A 与多 Agent 调度的关系

**A2A (Agent2Agent Protocol)** 是 Google 在 2025 年开源的开放协议，让不同厂商、不同框架构建的 Agent 能跨进程互通。它与 Anthropic 的 MCP 互补：

- **MCP**：Agent ↔ 工具/数据（垂直）
- **A2A**：Agent ↔ Agent（水平）

而本文对比的"多 Agent 调度框架"解决的是**单个系统内部**多个 Agent 如何协作的问题。两者的关系是：

- 框架（LangGraph / CrewAI / ADK ...）负责**进程内**的 Agent 编排
- A2A 协议负责**跨系统**的 Agent 互联

理想架构：内部用合适的框架编排，对外通过 A2A 暴露能力。

---

## 一张总览表

| 框架 | 调度模型 | 状态管理 | 通信机制 | A2A 原生支持 | 最佳场景 |
|---|---|---|---|---|---|
| **LangGraph** | 显式有向图（StateGraph）+ 条件边 | 共享 State 对象，checkpoint 持久化 | 节点间通过 State 读写 | 否（需自己接） | 复杂流程、长任务、人在回路 |
| **CrewAI** | Sequential / Hierarchical Process | Task 输出链式传递 | Task context + delegation | 否 | 角色分工明确的业务流 |
| **AutoGen** | GroupChat + Manager 选人 | 消息历史 | 多 Agent 对话 | 否 | 代码生成、对话式协作 |
| **Google ADK** | 层级树 + transfer_to_agent + Workflow Agent | Session State | A2A / 函数调用 | ✅ 原生 | 生产级、Google Cloud 集成 |
| **OpenAI Agents SDK** | Handoffs（Agent 之间转交） | 极简，靠 function call | Handoff function | 否 | 学习入门、轻量场景 |

---

## 逐个拆解

### 1. LangGraph — "把流程画成图"

```python
from langgraph.graph import StateGraph, END

graph = StateGraph(MyState)
graph.add_node("planner", planner_fn)
graph.add_node("coder",   coder_fn)
graph.add_node("reviewer", reviewer_fn)

graph.add_conditional_edges(
    "planner",
    route_fn,
    {"code": "coder", "end": END}
)
graph.add_edge("coder", "reviewer")
```

**核心思想**
显式定义状态机，每个节点读写一个共享 `State` 对象。控制流由"边"决定，可以是普通边或条件边。

**杀手锏**
- **Checkpointer**：流程可中断、可恢复、可时间旅行（重放到任意节点）
- **HITL**：原生支持人工审批、修改 state 后继续
- **Streaming**：节点级、token 级流式输出

**代价**
- 心智成本高，要自己画图
- API 抽象比较多（StateGraph / MessageGraph / Send API ...）

**适合**
流程复杂、需要严格控制、要做人在回路、需要持久化恢复的生产系统。

---

### 2. CrewAI — "组团干活"

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(role="研究员", goal="...", backstory="...")
writer     = Agent(role="作家",   goal="...", backstory="...")

task1 = Task(description="调研 X", agent=researcher)
task2 = Task(description="写报告", agent=writer, context=[task1])

crew = Crew(
    agents=[researcher, writer],
    tasks=[task1, task2],
    process=Process.hierarchical  # 或 Process.sequential
)
result = crew.kickoff()
```

**核心思想**
用"角色 + 任务"的隐喻组织协作。`Sequential` 按顺序跑；`Hierarchical` 自动生成一个 Manager Agent 负责分派。

**杀手锏**
- 上手最快，写起来像剧本
- Task 之间的 `context` 自动传递

**代价**
- 灵活性差，复杂分支不好表达
- 黑盒程度高，调试不直观
- Manager 由 LLM 担任，稳定性受模型影响

**适合**
角色边界清晰、流程线性或浅层级的业务（内容生产、调研报告、邮件处理）。

---

### 3. AutoGen — "把会议室搬到代码里"

```python
from autogen import AssistantAgent, UserProxyAgent, GroupChat, GroupChatManager

coder    = AssistantAgent("coder",    llm_config=cfg)
reviewer = AssistantAgent("reviewer", llm_config=cfg)
user     = UserProxyAgent("user", code_execution_config={"work_dir": "code"})

chat = GroupChat(agents=[user, coder, reviewer], messages=[], max_round=10)
manager = GroupChatManager(groupchat=chat, llm_config=cfg)

user.initiate_chat(manager, message="实现快速排序并测试")
```

**核心思想**
所有 Agent 在一个"群聊"里发言，由 `GroupChatManager` 用 LLM 决定**下一个该谁说话**。

**杀手锏**
- 代码执行 Agent (`UserProxyAgent`) 内建沙箱
- 对话式协作天然适合代码生成、debug 场景
- 微软在背后，与 VS Code、Magentic-One 生态联动

**代价**
- 由 LLM 选人 → 不可预测、容易死循环
- 大规模 Agent 协作时 token 爆炸（每个 Agent 看到全部历史）

**适合**
代码生成、自动化研究、对话式协作；不适合需要确定性流程的业务。

---

### 4. Google ADK (Agent Development Kit) — "面向生产的 A2A 原生方案"

```python
from google.adk.agents import Agent, SequentialAgent, ParallelAgent

researcher = Agent(name="researcher", model="gemini-2.0-flash", tools=[...])
writer     = Agent(name="writer",     model="gemini-2.0-flash")

pipeline = SequentialAgent(
    name="content_pipeline",
    sub_agents=[researcher, writer]
)
```

**核心思想**
层级树形结构。父 Agent 通过 `transfer_to_agent` 把控制权交给子 Agent；同时提供 `SequentialAgent` / `ParallelAgent` / `LoopAgent` 三种 Workflow Agent 做确定性编排。

**杀手锏**
- **原生 A2A**：每个 Agent 自动生成 Agent Card，可以直接被其他 A2A 客户端调用
- **Session State** 自动管理上下文
- 与 Vertex AI、Cloud Run、Gemini 深度集成
- 显式区分"LLM 路由"和"代码路由"（Workflow Agent）

**代价**
- 绑定 Google 生态较重（虽然支持其他模型）
- 中文资料少，社区还在成长

**适合**
要上生产、要跨系统互通（A2A）、已经在 Google Cloud 上的团队。

---

### 5. OpenAI Agents SDK — "用 handoff 转交对话"

```python
from agents import Agent, Runner, handoff

billing  = Agent(name="Billing",  instructions="处理账单问题")
support  = Agent(name="Support",  instructions="处理技术问题")
triage   = Agent(
    name="Triage",
    instructions="判断用户问题类别后转交",
    handoffs=[billing, support]
)

result = await Runner.run(triage, "我的发票有问题")
```

**核心思想**
极简模型：每个 Agent 是一个"工具集 + 指令"，通过 `handoff` 把对话整个转交给另一个 Agent，原 Agent 退出。

**杀手锏**
- API 极简，10 行代码就能跑
- 内置 tracing、guardrails、structured output
- 与 OpenAI 平台深度集成

**代价**
- Handoff 是单向转交，不适合需要多 Agent 同时协作的场景
- 状态管理薄弱，复杂流程要自己实现

**适合**
学习入门、客服路由、单线程任务派发。

---

## 调度模型的本质差异

抛开 API，五个框架其实代表了三种调度哲学：

| 调度哲学 | 代表 | 特点 |
|---|---|---|
| **代码编排（确定性）** | LangGraph、ADK Workflow Agent、CrewAI Sequential | 流程由开发者写死，可预测、可调试 |
| **LLM 编排（动态）** | AutoGen GroupChat、CrewAI Hierarchical、ADK LLM Agent | 由模型决定下一步，灵活但不稳定 |
| **混合** | LangGraph + Tool、ADK 全家桶 | 关键路径写死，分支由 LLM 决 |

**经验法则**：业务流程越确定，越往代码编排靠；探索性任务越多，越往 LLM 编排靠。生产系统几乎都是混合。

---

## 选型建议

按"我现在在哪个阶段"来选：

| 你的情况 | 推荐 |
|---|---|
| 刚学习，想理解概念 | **OpenAI Agents SDK** 或 **CrewAI**（上手最快） |
| 做原型 / Demo | **CrewAI**（角色化思维容易讲清楚） |
| 复杂生产流程、要 HITL、要可恢复 | **LangGraph** |
| 代码生成、debug、研究类任务 | **AutoGen** |
| 要上生产、要跨系统 A2A 互通 | **Google ADK** |
| 已深度使用 OpenAI 平台 | **OpenAI Agents SDK** |

---

## 共同的核心问题

不管用哪个框架，多 Agent 系统都绕不开这 4 件事：

1. **任务分解**：把用户请求拆成子任务（Planner Agent 或显式 workflow）
2. **能力路由**：选谁来做（基于 Agent Card / role / 描述匹配）
3. **状态管理**：上下文怎么传、历史怎么存（共享 state vs 消息历史）
4. **错误恢复**：失败重试、降级、人工介入

框架的区别本质是这 4 件事的**抽象层次和默认实现**不同。

---

## 参考资料

- A2A Protocol Spec: <https://a2aproject.github.io/A2A/>
- LangGraph: <https://langchain-ai.github.io/langgraph/>
- CrewAI: <https://docs.crewai.com/>
- AutoGen: <https://microsoft.github.io/autogen/>
- Google ADK: <https://google.github.io/adk-docs/>
- OpenAI Agents SDK: <https://openai.github.io/openai-agents-python/>
