# 多 Agent 调度框架对比

这篇文档比较 LangGraph、CrewAI、AutoGen、Google ADK、OpenAI Agents SDK 等框架。它们和 A2A 的关系是：框架负责系统内部编排，A2A 负责系统之间互联。

## A2A 与调度框架的边界

| 层次 | 解决的问题 | 代表 |
|---|---|---|
| Agent 内部工具连接 | Agent 如何使用工具和数据 | MCP、函数调用、API |
| 多 Agent 编排 | 一个系统内部多个 Agent 如何协作 | LangGraph、CrewAI、AutoGen、ADK、OpenAI Agents SDK |
| 跨系统 Agent 互联 | 不同系统、框架、组织的 Agent 如何通信 | A2A |

理想架构不是只选一个，而是组合使用：

```text
内部用 LangGraph / CrewAI / ADK 编排
对外用 A2A 暴露 Agent 能力
Agent 内部用 MCP / API / DB 接工具和数据
```

## 总览表

| 框架 | 调度模型 | 状态管理 | 通信机制 | A2A 支持 | 最适合场景 |
|---|---|---|---|---|---|
| LangGraph | 显式有向图 + 条件边 | 共享 State + checkpoint | 节点读写 State | 需要自行适配 | 复杂流程、长任务、人在回路 |
| CrewAI | Sequential / Hierarchical | Task 输出链式传递 | Task context + delegation | 需要自行适配 | 角色分工清晰的业务流 |
| AutoGen | GroupChat + Manager | 消息历史 | 多 Agent 对话 | 需要自行适配 | 代码生成、研究、对话式协作 |
| Google ADK | 层级 Agent + Workflow Agent | Session State | transfer / workflow / A2A | 原生支持更强 | 生产级、Google Cloud、A2A 互通 |
| OpenAI Agents SDK | Handoff | 轻量上下文 | Agent 间转交 | 需要自行适配 | 入门、客服路由、轻量任务派发 |

## LangGraph

LangGraph 的核心思想是把流程显式建模成图。每个节点执行一个步骤，节点之间通过边和条件边控制流转。

简化示例：

```python
from langgraph.graph import StateGraph, END

graph = StateGraph(MyState)
graph.add_node("planner", planner_fn)
graph.add_node("coder", coder_fn)
graph.add_node("reviewer", reviewer_fn)

graph.add_conditional_edges(
    "planner",
    route_fn,
    {"code": "coder", "end": END}
)
graph.add_edge("coder", "reviewer")
```

优势：

- 流程显式，可预测；
- checkpoint 能支持中断、恢复和重放；
- 适合人在回路；
- 适合复杂生产流程。

代价：

- 心智成本较高；
- 需要开发者自己设计状态结构；
- 对简单任务可能显得重。

适合把 LangGraph 外层包装成 A2A Agent：内部保持确定性图编排，对外暴露一个或多个业务 Skill。

## CrewAI

CrewAI 用“角色 + 任务”的方式组织 Agent 协作。

简化示例：

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(role="研究员", goal="调研主题", backstory="擅长资料整理")
writer = Agent(role="作者", goal="输出报告", backstory="擅长结构化表达")

task1 = Task(description="调研 A2A", agent=researcher)
task2 = Task(description="撰写报告", agent=writer, context=[task1])

crew = Crew(
    agents=[researcher, writer],
    tasks=[task1, task2],
    process=Process.sequential
)
result = crew.kickoff()
```

优势：

- 上手快；
- 角色分工容易理解；
- 适合线性或浅层级业务流。

代价：

- 复杂分支不够自然；
- Hierarchical 模式依赖 LLM Manager，稳定性受模型影响；
- 调试和状态可控性弱于显式图。

适合把 CrewAI 作为内容生产、调研、报告类 A2A Agent 的内部实现。

## AutoGen

AutoGen 的典型模型是多个 Agent 在一个群聊中协作，由 Manager 决定下一轮谁发言。

简化示例：

```python
from autogen import AssistantAgent, UserProxyAgent, GroupChat, GroupChatManager

coder = AssistantAgent("coder", llm_config=cfg)
reviewer = AssistantAgent("reviewer", llm_config=cfg)
user = UserProxyAgent("user", code_execution_config={"work_dir": "code"})

chat = GroupChat(agents=[user, coder, reviewer], messages=[], max_round=10)
manager = GroupChatManager(groupchat=chat, llm_config=cfg)

user.initiate_chat(manager, message="实现快速排序并测试")
```

优势：

- 适合代码生成、调试、研究类任务；
- 对话式协作自然；
- 可集成代码执行。

代价：

- LLM 决定发言顺序，存在不可预测性；
- 多 Agent 群聊容易 token 膨胀；
- 不适合强确定性业务流程。

适合暴露为“代码协作 Agent”“研究 Agent”，但要在 A2A 层限制最大轮次、最大成本和最大运行时间。

## Google ADK

Google ADK 更偏生产级 Agent 开发框架，支持层级 Agent、Workflow Agent，并和 A2A 生态关系更紧密。

简化示例：

```python
from google.adk.agents import Agent, SequentialAgent

researcher = Agent(name="researcher", model="gemini-2.0-flash", tools=[...])
writer = Agent(name="writer", model="gemini-2.0-flash")

pipeline = SequentialAgent(
    name="content_pipeline",
    sub_agents=[researcher, writer]
)
```

优势：

- 区分 LLM 路由和代码化 Workflow；
- Session State 管理较完整；
- 与 Google Cloud、Vertex AI、Gemini 集成强；
- A2A 支持更靠近官方生态。

代价：

- 对 Google 生态依赖较强；
- 中文资料相对少；
- 需要关注版本变化。

适合想直接探索 A2A 原生 Agent 暴露方式的团队。

## OpenAI Agents SDK

OpenAI Agents SDK 的核心模型比较轻：Agent 可以通过 handoff 把控制权转交给另一个 Agent。

简化示例：

```python
from agents import Agent, Runner

billing = Agent(name="Billing", instructions="处理账单问题")
support = Agent(name="Support", instructions="处理技术问题")
triage = Agent(
    name="Triage",
    instructions="判断用户问题类别后转交",
    handoffs=[billing, support]
)

result = await Runner.run(triage, "我的发票有问题")
```

优势：

- API 简洁；
- 容易入门；
- 适合客服路由和轻量任务派发；
- 平台内 tracing、guardrails、structured output 能力较完整。

代价：

- Handoff 偏单向转交；
- 不适合复杂并行协作；
- 复杂状态管理需要自行补齐。

## 调度哲学

抛开 API，常见框架可以分成三类：

| 调度哲学 | 代表 | 特点 |
|---|---|---|
| 代码编排 | LangGraph、ADK Workflow、CrewAI Sequential | 稳定、可预测、可调试 |
| LLM 编排 | AutoGen GroupChat、CrewAI Hierarchical、部分 ADK LLM Agent | 灵活，但不稳定 |
| 混合编排 | LangGraph + Tool、ADK 组合模式 | 关键路径确定，局部分支交给模型 |

经验法则：

> 业务流程越确定，越靠近代码编排；探索性越强，越可以引入 LLM 编排。生产系统通常是混合模式。

## 选型建议

| 你的情况 | 推荐 |
|---|---|
| 刚学习概念 | OpenAI Agents SDK 或 CrewAI |
| 做 Demo 或原型 | CrewAI |
| 复杂生产流程、人在回路、可恢复 | LangGraph |
| 代码生成、调试、研究 | AutoGen |
| 要生产级 A2A 互通、Google Cloud 集成 | Google ADK |
| 已深度使用 OpenAI 平台 | OpenAI Agents SDK |

## 暴露为 A2A Agent 时要回答的问题

不管内部用哪个框架，对外暴露为 A2A Agent 都要补齐：

- Agent Card 如何生成；
- Skill 如何划分；
- Message 如何映射到内部输入；
- Task Store 谁负责；
- 流式事件如何从内部框架转成 A2A 状态更新；
- cancel 如何映射到内部流程取消；
- Artifact 如何存储和返回；
- 错误如何标准化；
- trace id 如何贯穿内部节点和外部 A2A 调用。

## 参考资料

- LangGraph：<https://langchain-ai.github.io/langgraph/>
- CrewAI：<https://docs.crewai.com/>
- AutoGen：<https://microsoft.github.io/autogen/>
- Google ADK：<https://google.github.io/adk-docs/>
- OpenAI Agents SDK：<https://openai.github.io/openai-agents-python/>

