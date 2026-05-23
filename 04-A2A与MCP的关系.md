# A2A 与 MCP 的关系

A2A 和 MCP 经常被放在一起讨论，但它们解决的是两类不同问题。

## 一句话区分

- **MCP**：Agent 如何连接工具、数据和外部资源。
- **A2A**：Agent 如何和另一个 Agent 通信、协作和交接任务。

也可以理解为：

```text
MCP：Agent ↔ 工具 / 数据
A2A：Agent ↔ Agent
```

## 对比表

| 维度 | MCP | A2A |
|---|---|---|
| 核心对象 | 工具、资源、提示模板 | Agent、Skill、Message、Task、Artifact |
| 通信方向 | 垂直连接能力资源 | 水平连接 Agent 对等体 |
| 典型关系 | Agent 调工具 | Agent 委托另一个 Agent |
| 状态重点 | 工具调用与上下文资源 | 长任务、多轮、状态流转 |
| 发现内容 | 有哪些工具和资源 | 有哪些 Agent 能力 |
| 适合场景 | 查数据库、读文件、调 API | 跨团队、跨系统、跨组织协作 |

## 为什么两者互补

真实系统中，一个 A2A Agent 内部很可能通过 MCP 使用工具。

例如：

```text
采购 Agent
   │
   │ A2A 调用
   ▼
供应商报价 Agent
   │
   ├── MCP 读取商品目录
   ├── MCP 查询库存系统
   ├── MCP 调用价格策略工具
   └── MCP 生成报价单文件
```

从调用方看，它只是在和一个“报价 Agent”协作；从被调用方内部看，它需要调用多个工具和数据源完成任务。

## 边界不要混淆

### 不要把工具包装成 Agent

如果某个能力只是“输入参数，返回确定结果”，例如：

- 查询订单；
- 查询库存；
- 获取用户信息；
- 读取文件；
- 调用一个固定 API。

这类能力通常更适合 MCP 或普通 API，而不是 A2A。

### 不要把复杂 Agent 当工具

如果某个能力会：

- 自己规划步骤；
- 调用多个工具；
- 需要长时间运行；
- 可能要求补充输入；
- 产出多个 Artifact；
- 需要独立审计和治理；
- 可以被其他 Agent 复用。

这类能力更适合作为 A2A Agent 暴露。

## 组合架构

一个较完整的企业 Agent 架构可能是：

```text
用户 / 业务系统
      │
      ▼
主控 Agent / Orchestrator
      │
      ├── A2A ──► 发票 Agent ── MCP ──► 财务系统
      ├── A2A ──► 合同 Agent ── MCP ──► 文档库
      ├── A2A ──► 采购 Agent ── MCP ──► ERP
      └── A2A ──► 风控 Agent ── MCP ──► 规则引擎
```

A2A 管东西向 Agent 通信，MCP 管南北向工具和数据访问。

## 混用时的关键问题

### 身份透传

当用户通过 Agent A 调用 Agent B，Agent B 又通过 MCP 访问数据库时，需要明确：

- 数据访问代表谁；
- 是 Agent B 自己的权限，还是最终用户的权限；
- 是否需要 OAuth2 Token Exchange 或 OBO；
- 审计日志里记录哪个主体。

### 权限收敛

不能因为 Agent B 被 A2A 调用了，就默认它可以访问所有 MCP 工具。建议：

- 按 Skill 限制可访问工具；
- 按用户或租户限制数据范围；
- 在 Gateway 层做策略判断；
- 在 MCP Server 层继续做资源级鉴权。

### 数据脱敏

A2A 的调用输入可能来自外部组织，不能直接传给内部工具。常见策略：

- 输入 schema 校验；
- 敏感字段过滤；
- Prompt injection 检测；
- 文件扫描；
- 只传递最小必要上下文。

### 链路观测

A2A 和 MCP 应共享 trace id 或至少建立映射关系：

```text
用户请求 trace
   └── A2A Task trace
         └── MCP Tool call trace
```

否则跨 Agent 和跨工具的错误排查会非常困难。

## 判断标准

可以用这个问题判断该用 A2A 还是 MCP：

> 我要调用的是一个“会自主完成任务的协作者”，还是一个“被动执行能力的工具”？

前者偏 A2A，后者偏 MCP。

## 参考资料

- A2A 最新规范：<https://a2a-protocol.org/latest/specification/>
- MCP 官方文档：<https://modelcontextprotocol.io/>

