# A2A 协议深度解析

> 全面理解 Agent2Agent Protocol：背景、设计、实现、问题与生态。

---

## 一、背景与动机

### 1.1 为什么需要 A2A

2024-2025 年 Agent 爆发后，行业出现了一个明显的割裂：

- **框架割裂**：LangGraph、CrewAI、AutoGen、ADK、Bedrock Agents… 各自为政
- **厂商割裂**：Salesforce Einstein、ServiceNow Now Assist、SAP Joule、Atlassian Rovo… 都是 Agent，但互相不认识
- **能力孤岛**：每家都把 Agent 包成 API，但调用方式、能力描述、状态语义全不一样

企业的真实诉求：
- 我的"采购 Agent"能不能直接调对方公司的"开票 Agent"？
- 我用 LangGraph 写的主管 Agent，能不能调度别人用 CrewAI 写的执行 Agent？
- Agent 之间能不能像微服务那样，有标准的发现、调用、流式、回调机制？

### 1.2 A2A 的诞生

- **2025 年 4 月**：Google 在 Cloud Next 大会上发布 A2A，50+ 合作伙伴（Salesforce、SAP、ServiceNow、Atlassian、MongoDB、Cohere、Accenture、Deloitte 等）联合背书
- **2025 年中**：Google 将 A2A 捐赠给 **Linux Foundation**，成为中立开源项目
- **定位**：开放、跨厂商、跨框架的 Agent 通信标准

### 1.3 A2A 与 MCP 的关系

这是最容易混淆的一点：

| 维度 | MCP (Anthropic) | A2A (Google → Linux Foundation) |
|---|---|---|
| 解决的问题 | Agent ↔ 工具/数据 | Agent ↔ Agent |
| 类比 | USB-C（硬件接口） | HTTP（应用协议） |
| 方向 | 垂直（向下连接资源） | 水平（横向连接对等体） |
| 通信对象 | "无状态"工具调用 | 有状态、长生命周期任务 |

**两者互补，不互斥**。真实系统里：每个 A2A Agent 内部用 MCP 接工具。

---

## 二、核心概念

A2A 的模型只有 5 个核心对象，记住这 5 个就掌握了 80%。

### 2.1 Agent Card（名片）

每个 A2A Agent 在 `/.well-known/agent.json` 暴露自己的"名片"，描述：

```json
{
  "name": "Invoice Agent",
  "description": "处理发票开具、查询、作废",
  "url": "https://billing.example.com/a2a",
  "version": "1.2.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "stateTransitionHistory": true
  },
  "authentication": {
    "schemes": ["Bearer", "OAuth2"]
  },
  "defaultInputModes":  ["text/plain", "application/json"],
  "defaultOutputModes": ["text/plain", "application/pdf"],
  "skills": [
    {
      "id": "issue_invoice",
      "name": "开具发票",
      "description": "根据订单号生成发票 PDF",
      "tags": ["billing", "invoice"],
      "examples": ["给订单 #1234 开发票"],
      "inputModes":  ["application/json"],
      "outputModes": ["application/pdf"]
    }
  ]
}
```

**关键设计**：
- 通过固定路径 `/.well-known/agent.json` 实现**零配置发现**（类似 OIDC discovery）
- `skills` 是结构化的能力清单，让其他 Agent / 编排器能基于语义路由
- `capabilities` 声明可选特性，客户端按需启用

### 2.2 Task（任务）

A2A 的**最小工作单元**，有完整生命周期：

```
submitted ──► working ──► completed
                │   │
                │   └──► input-required ──► working ...
                │
                ├──► failed
                └──► canceled
```

每个 Task 有：
- `id`：全局唯一标识
- `sessionId`：可选，用来串联多个相关 Task
- `status`：当前状态 + 时间戳 + 可选消息
- `history`：消息历史
- `artifacts`：产出
- `metadata`：扩展字段

### 2.3 Message（消息）

Task 内部的一轮发言，由多个 **Part** 组成：

```json
{
  "role": "user",
  "parts": [
    { "type": "text", "text": "请给这张订单开发票" },
    { "type": "file", "file": { "name": "order.pdf", "mimeType": "application/pdf", "bytes": "..." } },
    { "type": "data", "data": { "orderId": "1234", "amount": 999 } }
  ]
}
```

**三种 Part**：
- `TextPart`：纯文本
- `FilePart`：文件（bytes 或 URI）
- `DataPart`：结构化 JSON

**多模态从协议层就被设计进去**，不是事后补丁。

### 2.4 Artifact（产出）

Task 完成时产出的结果，结构和 Message 类似（也是 Parts 数组），但语义不同：
- Message 是"对话内容"
- Artifact 是"最终交付物"

一个 Task 可以产出多个 Artifact（一个发票任务可能产出 PDF + 结构化 JSON）。

### 2.5 Push Notification Config（异步回调）

长任务不可能让客户端一直挂着，A2A 内建 webhook 回调：

```json
{
  "url": "https://my-agent.example.com/callbacks",
  "token": "opaque-correlation-token",
  "authentication": { "schemes": ["Bearer"] }
}
```

服务端在 Task 状态变更时主动 POST 通知。

---

## 三、通信机制

### 3.1 传输层

- **协议**：JSON-RPC 2.0
- **载体**：HTTPS POST
- **流式**：Server-Sent Events (SSE)
- **异步**：Webhook (Push Notification)

选 JSON-RPC 而不是 REST，因为 Agent 交互更像 RPC（方法调用），不像资源 CRUD。

### 3.2 核心 RPC 方法

| 方法 | 作用 | 同步性 |
|---|---|---|
| `tasks/send` | 提交任务，等待最终结果 | 阻塞直到 Task 终态 |
| `tasks/sendSubscribe` | 提交任务 + SSE 流式订阅 | 流式 |
| `tasks/get` | 查询 Task 状态 | 同步 |
| `tasks/cancel` | 取消 Task | 同步 |
| `tasks/pushNotification/set` | 注册 webhook | 同步 |
| `tasks/pushNotification/get` | 查询 webhook 配置 | 同步 |
| `tasks/resubscribe` | SSE 断线重连 | 流式 |

### 3.3 一次典型的流式交互

```
Client                                  Server
  │                                       │
  │── POST tasks/sendSubscribe ──────────►│
  │                                       │
  │◄── SSE: {state: "working"} ───────────│
  │◄── SSE: {state: "working", msg:...} ──│
  │◄── SSE: {state: "input-required"} ────│
  │                                       │
  │── POST tasks/send (sameTaskId) ──────►│
  │                                       │
  │◄── SSE: {state: "working"} ───────────│
  │◄── SSE: {state: "completed", artifact}│
  │                                       │
```

**关键点**：同一个 `taskId` 支持**多轮交互**，包括 `input-required` 后的补充输入。

### 3.4 认证

Agent Card 声明支持的方案，常见组合：
- `Bearer`（最常用，JWT/Opaque token）
- `OAuth2`（含 client_credentials、authorization_code 流）
- `ApiKey`
- `mTLS`（企业场景）

A2A **不规定**具体的身份提供方，只规定如何声明和携带。

---

## 四、A2A 解决了什么问题

把抽象的"互操作"展开成 6 个具体痛点：

### 4.1 能力发现（Discovery）
> 怎么知道一个 Agent 能做什么？

**A2A 方案**：标准化的 Agent Card + skills 清单 + well-known URL。

### 4.2 调用语义统一（Invocation）
> 同样是"发任务"，每家 API 都不一样。

**A2A 方案**：所有 Agent 都实现 `tasks/send`，参数结构一致。

### 4.3 长任务管理（Long-running tasks）
> Agent 任务常常要几分钟到几小时，HTTP 同步搞不定。

**A2A 方案**：Task 生命周期 + SSE 流式 + Push Notification 三件套。

### 4.4 多模态原生（Multimodal）
> 任务输入输出可能是文本、文件、结构化数据混合。

**A2A 方案**：Parts 数组，类型可扩展。

### 4.5 多轮交互（Conversational）
> Agent 可能问 "你需要哪个月的发票？" 然后等用户回复。

**A2A 方案**：`input-required` 状态 + 同 taskId 续传。

### 4.6 跨组织协作（Cross-org）
> A 公司的 Agent 调 B 公司的 Agent。

**A2A 方案**：标准认证 + Agent Card 公开 + HTTPS。

---

## 五、典型应用场景

### 5.1 企业内多框架编排

```
[主管 Agent / LangGraph]
        ├──A2A──► [研究 Agent / CrewAI]
        ├──A2A──► [代码 Agent / AutoGen]
        └──A2A──► [报表 Agent / 自研]
```

价值：每个团队用自己擅长的框架，不强求统一技术栈。

### 5.2 跨组织 B2B 集成

```
[采购公司 - 采购 Agent] ──A2A──► [供应商公司 - 报价 Agent]
                       ──A2A──► [物流公司 - 物流 Agent]
                       ──A2A──► [银行 - 付款 Agent]
```

价值：把"系统集成"从 SOAP/REST 时代推进到 Agent 时代，业务方用自然语言协作。

### 5.3 Agent 市场（Agent Marketplace）

类似 npm / app store，但卖的是 Agent 服务。买家通过 Agent Card 选型，通过 A2A 调用。

### 5.4 SaaS 厂商间的"Agent 联盟"

Salesforce 的 Einstein 直接调用 ServiceNow 的工单 Agent，不再需要专门写集成。

### 5.5 内部"Agent 微服务"架构

把企业内能力按业务域拆成独立 Agent，A2A 当作 Agent 间的"东西向通信"协议（类比 gRPC 之于微服务）。

---

## 六、参考实现与 SDK

### 6.1 官方 SDK

| 语言 | 仓库 | 状态 |
|---|---|---|
| Python | `a2aproject/a2a-python` | 主推，最完善 |
| JavaScript / TypeScript | `a2aproject/a2a-js` | 完善 |
| Java | `a2aproject/a2a-java` | 完善 |
| .NET | `a2aproject/a2a-dotnet` | 完善 |
| Go | 社区维护 | 部分能力 |

### 6.2 配套工具

- **A2A Inspector**：图形化调试客户端，能可视化 Task 生命周期、Part 内容、SSE 流
- **A2A CLI**：命令行调用 Agent
- **Sample agents**：官方仓库 `a2a-samples` 里有 LangGraph / CrewAI / ADK / Semantic Kernel 等框架的示例

### 6.3 框架集成

- **Google ADK**：原生 A2A，每个 Agent 自动生成 Agent Card
- **LangGraph**：社区适配器 `langgraph-a2a`
- **CrewAI**：通过 `crewai-tools` 暴露 A2A endpoint
- **Semantic Kernel**：微软提供 A2A 连接器

---

## 七、A2A 还没解决的问题（真正的工程挑战）

这是**最值得深入思考**的部分。协议本身只解决了"语法"，但生产部署还要解决一堆"语义/工程"问题：

### 7.1 信任与身份（Trust & Identity）

**问题**：
- 我收到一个声称是"Salesforce 发票 Agent"的请求，怎么验证？
- Agent Card 被篡改怎么办？
- Agent 在跨组织调用时如何传递最终用户身份（user-on-behalf-of）？

**目前方案 / 演进方向**：
- 签名的 Agent Card（类似 SBOM 签名）
- OAuth2 Token Exchange / OBO 流程
- 企业级 Agent Registry（受控目录）

### 7.2 成本归属（Cost Attribution）

**问题**：A 调 B，B 调 C，C 调 LLM，token 费用算谁的？

**目前方案**：
- HTTP Header 传 cost-context（非标准）
- Push Notification 回调里附带 usage 元数据
- 待标准化

### 7.3 跨 Agent 可观测性（Observability）

**问题**：一个用户请求链路跨 5 个 Agent，怎么追踪？

**目前方案**：
- W3C Trace Context（`traceparent` header）
- OpenTelemetry GenAI semantic conventions
- 每个 Task 的 `metadata` 字段携带 trace_id

### 7.4 版本演进（Versioning）

**问题**：
- Agent Card 里的 `version` 怎么用？
- Skill 的输入 schema 变了怎么平滑升级？
- 客户端怎么协商版本？

**目前现状**：协议未规定，靠各家约定。

### 7.5 任务状态持久化（State Durability）

**问题**：
- 服务端重启了，正在跑的 Task 怎么办？
- SSE 断线重连后状态怎么对齐？

**目前方案**：
- `tasks/resubscribe` 支持断线重连
- 服务端需自己实现 Task store（Redis / DB）
- 协议不规定存储后端

### 7.6 限流、配额、SLA

**问题**：
- 一个 Agent 同时被 1000 个其他 Agent 调用怎么办？
- 怎么声明 SLA（延迟、可用性）？

**目前现状**：Agent Card 里没有 SLA 字段，需要扩展。

### 7.7 错误语义

**问题**：JSON-RPC error code 表达力有限，业务级错误怎么标准化？

**目前方案**：
- 标准 JSON-RPC 错误码（-32xxx）
- 业务错误塞 `data` 字段，无统一约定

### 7.8 多 Agent 编排（Choreography vs Orchestration）

**问题**：A2A 本身只定义"两两通信"，多 Agent 协作的编排逻辑写在哪？

**答案**：**A2A 不管编排**。编排是上层框架（LangGraph / ADK / 自研 orchestrator）的事。A2A 是"传输层"，编排是"应用层"。

### 7.9 安全与对抗

**问题**：
- 恶意 Agent 注入 prompt injection
- 数据外泄（Agent A 把敏感数据传给不该看的 Agent B）
- Agent 互调死循环

**目前方案**：
- 输入 sanitization 在应用层
- 调用图限制（最大深度、最大 fan-out）
- Agent Gateway 做策略管控

### 7.10 与 MCP 的边界与混用

**问题**：Agent 既是 A2A server，又用 MCP 接工具，能不能复用 transport / auth？

**目前方向**：
- 部分实现复用底层 HTTP 框架
- 上层语义保持分离

---

## 八、生态相关与对比

### 8.1 类似协议

| 协议 | 来源 | 定位 | 与 A2A 的关系 |
|---|---|---|---|
| **MCP** | Anthropic | Agent ↔ 工具 | 互补 |
| **ACP** | IBM Research | Agent ↔ Agent | 直接竞品，理念类似，社区较小 |
| **AGNTCY** | Cisco / Outshift | Agent Network 层 | 更底层（网络发现 / 路由） |
| **OpenAI Agents** | OpenAI | 框架 + 内部协议 | 闭源生态 |
| **LangChain Hub** | LangChain | Agent 分发 | 不是协议是平台 |

### 8.2 A2A 的"基础设施层"配套

随着 A2A 普及，下面这些会成为生态拼图：

1. **Agent Registry**：企业级目录服务，管控允许调用的 Agent 列表
2. **Agent Gateway / Proxy**：策略、限流、审计、安全过滤
3. **Agent Identity Provider**：颁发 Agent 身份证书
4. **Agent Observability**：跨 Agent 链路追踪、成本分析
5. **Agent Marketplace**：发现 + 计费 + SLA
6. **Agent Schema Registry**：skill 输入输出 schema 版本管理

这些目前还没有事实标准，是创业 / 内部平台建设的机会窗口。

---

## 九、学习路径建议

如果想真正掌握 A2A，建议按这个顺序：

1. **读 Spec**：<https://a2aproject.github.io/A2A/specification/>（一遍就够，主要看 Task / Message / Agent Card / RPC methods）
2. **跑 Sample**：克隆 `a2a-samples`，跑一个 hello world
3. **写最小客户端 + 服务端**：用 Python SDK 实现一个能开发票的 Agent + 一个调用它的客户端
4. **接上流式**：用 `tasks/sendSubscribe` 实现流式输出
5. **接上 Push Notification**：实现真正的长任务
6. **接入框架**：把现有的 LangGraph / CrewAI Agent 包成 A2A server
7. **解决一个生产问题**：trace、auth、错误处理任选其一

---

## 十、参考资料

- **Spec**：<https://a2aproject.github.io/A2A/>
- **GitHub Org**：<https://github.com/a2aproject>
- **Sample agents**：<https://github.com/a2aproject/a2a-samples>
- **Python SDK**：<https://github.com/a2aproject/a2a-python>
- **A2A Inspector**：<https://github.com/a2aproject/a2a-inspector>
- **Linux Foundation announcement**：A2A 项目托管页
- **MCP Spec（互补阅读）**：<https://modelcontextprotocol.io/>

---

## 附录：一张图理解 A2A 全貌

```
┌─────────────────────────────────────────────────────────┐
│                     Client Agent                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  1. 发现：GET /.well-known/agent.json           │    │
│  │  2. 选 skill                                     │    │
│  │  3. tasks/sendSubscribe                          │    │
│  └─────────────────────────────────────────────────┘    │
└────────────┬────────────────────────────────────────────┘
             │  JSON-RPC over HTTPS + SSE
             ▼
┌─────────────────────────────────────────────────────────┐
│                     Remote Agent                         │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────┐  │
│  │ Agent Card   │   │ Task Manager │   │  Executor   │  │
│  │ (skills,     │──▶│ (lifecycle,  │──▶│ (LLM +      │  │
│  │  auth, caps) │   │  history)    │   │  MCP tools) │  │
│  └──────────────┘   └──────────────┘   └─────────────┘  │
│                            │                             │
│                            ▼                             │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Push Notification → Client webhook (long tasks)   │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```
