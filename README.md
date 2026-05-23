# learn-A2A

这是一个用于学习 A2A（Agent2Agent Protocol）的中文笔记仓库，重点梳理 A2A 的协议模型、任务生命周期、与 MCP 的关系、多 Agent 框架对比，以及生产级落地时需要补齐的工程能力。

## 阅读入口

建议从 [00-A2A文档导航.md](./00-A2A文档导航.md) 开始阅读。该文档提供完整阅读顺序和后续可扩展专题。

## 文档目录

| 文档 | 内容 |
|---|---|
| [00-A2A文档导航.md](./00-A2A文档导航.md) | 全局阅读路线和文档主线 |
| [01-A2A协议概览.md](./01-A2A协议概览.md) | A2A 解决的问题、定位和边界 |
| [02-A2A核心模型.md](./02-A2A核心模型.md) | Agent Card、Skill、Message、Task、Artifact、Part |
| [03-A2A通信机制与任务生命周期.md](./03-A2A通信机制与任务生命周期.md) | 协议绑定、流式、回调、任务状态流转 |
| [04-A2A与MCP的关系.md](./04-A2A与MCP的关系.md) | A2A 与 MCP 的边界、组合架构和混用风险 |
| [05-多Agent调度框架对比.md](./05-多Agent调度框架对比.md) | LangGraph、CrewAI、AutoGen、Google ADK、OpenAI Agents SDK 对比 |
| [06-A2A生产工程挑战.md](./06-A2A生产工程挑战.md) | 身份、成本、观测、版本、持久化、限流、错误和安全 |
| [07-A2A生态与基础设施.md](./07-A2A生态与基础设施.md) | Registry、Gateway、Identity、Observability、Marketplace、Schema Registry |
| [08-A2A学习路径与实践路线.md](./08-A2A学习路径与实践路线.md) | 从读规范到跑样例、写最小 Agent、接框架和补生产能力 |
| [09-A2A规范版本差异与待核对项.md](./09-A2A规范版本差异与待核对项.md) | A2A latest / v1.0 相关差异和后续核对清单 |

## 核心理解

A2A 不是多 Agent 编排框架，而是 Agent 之间互联互通的协议层。

- **A2A**：Agent 与 Agent 之间通信、委托任务、接收进度和结果。
- **MCP**：Agent 连接工具、数据和外部资源。
- **多 Agent 框架**：系统内部多个 Agent 如何协作和编排。

一个典型组合是：

```text
外部 Agent / Orchestrator
        │
        │ A2A
        ▼
业务 Agent
        │
        │ MCP / API / DB
        ▼
工具、数据、业务系统
```

## 参考资料

- A2A 最新规范：<https://a2a-protocol.org/latest/specification/>
- A2A GitHub：<https://github.com/a2aproject>
- A2A Samples：<https://github.com/a2aproject/a2a-samples>
- MCP 官方文档：<https://modelcontextprotocol.io/>
