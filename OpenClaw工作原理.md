# OpenClaw 工作原理

OpenClaw 不是一个普通的聊天机器人，而是一个住在你电脑里的私人秘书，它能通过 WhatsApp、Telegram、Slack、Discord、WebChat 等聊天渠道接收指令（不同渠道可能由插件提供），帮你发邮件、查日历、打开浏览器、运行命令，甚至 24 小时自动做事！[参考](https://docs.openclaw.ai/channels)

```mermaid
flowchart LR
  U[你（在聊天软件里发消息）] --> C[Channel 渠道适配器]
  C --> G[Gateway 网关（常驻进程）]
  G --> A[Agent 运行时]
  A -->|需要推理| M[LLM 模型提供方（API 或本地）]
  A -->|需要动作| S[Skills / Tools]
  S --> OS[本机/节点能力：文件、命令、浏览器等]
  A <--> MEM[Memory / Sessions（磁盘上的状态）]
```

### 核心概念

```mermaid
flowchart TB
  subgraph 输入
    CH[Channels（渠道）]
  end
  subgraph 控制平面
    GW[Gateway（网关）]
  end
  subgraph 智能体
    WS[Workspace（工作空间）]
    AG[Agent（策略 + 会话 + 工具编排）]
  end
  subgraph 能力
    SK[Skills（技能：SKILL.md 说明 + 可选脚本）]
    TL[Tools（工具：exec/browser/web/fs 等）]
    LLM[LLM（模型推理）]
    ST[State（sessions / memory / auth）]
  end
  CH --> GW
  GW --> AG
  AG --> WS
  AG --> SK
  AG --> TL
  AG --> LLM
  AG --> ST
```

| 概念                      | 类比           | 作用                                       |
| :------------------------ | :------------- | :----------------------------------------- |
| **Gateway（网关）**       | 大楼的前台接待 | 接收所有外部消息，分发到正确的工作空间     |
| **Workspace（工作空间）** | 你的私人办公室 | 处理具体任务，管理对话历史和技能           |
| **LLM（大语言模型）**     | AI顾问大脑     | 理解你的意图，生成回复                     |
| **Skills（技能）**        | 工具箱里的工具 | 执行特定功能（查天气、写代码、管理日程等） |
| **Channels（渠道）**      | 通信设备       | 连接不同的消息平台                         |

### OpenClaw 到底是什么？

OpenClaw 是一个**完全开源、自己运行在你电脑上的 AI 助手**（以前叫 Clawdbot，后来改名）。
它不像 ChatGPT 那样只聊天，而是真正动手做事：

- 你在 WhatsApp 里说："帮我查一下明天航班"，它就能自动打开浏览器、登录航空公司网站、截图给你。
- 它支持几百个技能（Skills），社区还在不断增加。
- 它有长期记忆（记得你喜欢什么），还能自己生成新技能。
- 最重要的是：**你的会话、工作区文件、技能与日志主要保存在本机状态目录中**；但当你使用云端模型时，消息内容会按配置发送给对应模型提供方（你也可以选择本地模型来减少外发）。

核心思想：**把 AI 大模型（大脑） + 本地工具（手脚） + 聊天软件（嘴巴耳朵）连接起来**，让 AI 真正成为你的电脑管家。

### 整体架构：Gateway 是大脑指挥中心

OpenClaw 最核心的部分叫 **Gateway（网关）**。它就像你家里的总控台，所有东西都围绕它转。

```mermaid
flowchart TB
  subgraph Messaging
    WA[WhatsApp（Baileys）]
    TG[Telegram（grammY）]
    SL[Slack]
    DC[Discord]
    WC[WebChat]
    MORE[其他渠道/插件]
  end
  subgraph GatewayHost["Gateway（单机一份常驻网关）"]
    GW[Gateway WS/HTTP :18789]
  end
  subgraph Clients["控制面客户端（operator）"]
    CLI[CLI]
    UI[Web UI / macOS app]
    AUTO[Automations]
  end
  subgraph Nodes["节点设备（node，可选）"]
    N1[macOS / iOS / Android / headless]
  end
  WA-->GW
  TG-->GW
  SL-->GW
  DC-->GW
  WC-->GW
  MORE-->GW
  CLI<-->GW
  UI<-->GW
  AUTO<-->GW
  N1<-->GW
```

**各部分简单解释：**

- **Channel Bridge（通道桥接器）**：负责跟 WhatsApp、Telegram 等聊天软件"握手"。比如用 Baileys 库连接 WhatsApp。
- **Gateway**：核心常驻进程（默认绑定 `127.0.0.1:18789`），既承载各类渠道连接，也提供 WebSocket 控制面（CLI/Web UI/自动化/节点设备接入）。官方架构说明：一个 host 通常只运行一个 Gateway，WhatsApp 会话由 Gateway 独占管理。[参考](https://docs.openclaw.ai/concepts/architecture)
- **AI 大脑**：真正思考的是外部大模型（你提供 API Key），Gateway 只负责"叫它来干活"。
- **工具 & 技能**：AI 的手脚，比如打开浏览器、读写文件、发邮件。
- **记忆系统**：像笔记本，AI 不会忘掉你上次说的话。

------

## 一条消息是怎么变成行动的？

我们用一个真实例子："在 WhatsApp 里说：帮我把今天的邮件整理成总结发给我"。

流程图如下：

```mermaid
sequenceDiagram
  autonumber
  participant User as 你
  participant Channel as Channel
  participant Gateway as Gateway
  participant Agent as Agent
  participant Memory as Sessions/Memory
  participant LLM as LLM
  participant Tools as Skills/Tools

  User->>Channel: 发送消息
  Channel->>Gateway: 标准化入站事件
  Gateway->>Agent: 路由到 agentId + sessionKey
  Agent->>Memory: 读取会话/记忆文件（磁盘）
  Agent->>LLM: 发送上下文 + 可用工具目录
  LLM-->>Agent: 计划（可能包含 tool calls）
  Agent->>Tools: 执行技能/工具（可能在沙箱或节点上）
  Tools-->>Agent: 返回结果
  Agent->>LLM: 回传工具结果
  LLM-->>Agent: 生成最终回复
  Agent->>Gateway: 输出回复
  Gateway->>Channel: 回发消息
```

**详细拆解：**

1. **接收指令**：你发消息 → 聊天软件的 Bridge 把消息推给 Gateway。
2. **查找记忆**：Gateway 打开你的"个人档案"（Session + Memory），知道你是老用户、上次喜欢什么语气。
3. **AI 思考**：把消息 + 记忆打包发给 AI 大脑。AI 像聪明秘书："嗯，需要先读邮件，再总结。"
4. **调用工具**：AI 说"我要用 Gmail 技能"。Gateway/Agent 触发工具执行；这一步是否在沙箱中取决于你是否启用了沙箱与权限策略（可选）。[参考](https://docs.openclaw.ai/gateway/security)
5. **执行 + 反馈**：工具把结果给 AI，AI 写出总结。
6. **回复用户**：Gateway 把总结发回 WhatsApp。你就收到了！

------

## 整体架构

### 三层架构设计

OpenClaw 采用经典的三层架构，让我们从外到内逐层理解：

```mermaid
flowchart TB
  subgraph L1["外层：消息与控制面入口"]
    Channels[Channels（WhatsApp/Telegram/Slack/...）]
    Clients[CLI/Web UI/自动化（operator）]
    Nodes[节点设备（node，可选）]
  end
  subgraph L2["中层：Gateway（统一控制平面）"]
    GW[Gateway WS/HTTP]
  end
  subgraph L3["内层：Agent Runtime（会话 + 工具编排）"]
    AG[Agent（路由/策略/工具调用）]
    WS[Workspace（可编辑文件）]
    ST[Sessions/Memory/Auth（磁盘状态）]
  end
  subgraph L4["能力层：模型与工具"]
    LLM[LLM Providers / 本地模型]
    TOOLS[Tools/Skills（exec/browser/fs/web/...）]
  end

  Channels --> GW
  Clients <--> GW
  Nodes <--> GW
  GW --> AG
  AG --> WS
  AG --> ST
  AG --> LLM
  AG --> TOOLS
```

### 为什么要这样设计？

**类比**：想象一个大型公司

- **外层（用户接口）** = 客户可以通过电话、邮件、微信等多种方式联系公司
- **中层（Gateway）** = 前台接待，统一接待所有客户，然后分配到合适的部门
- **内层（Workspace）** = 不同的业务部门，各自负责不同的事务
- **底层（能力层）** = 公司的资源（专家顾问、工具设备等）

------

## 核心组件详解

### Gateway：统一的门户

**Gateway 是什么？**

Gateway（网关）是 OpenClaw 的大门，所有外部消息都必须先经过这里。

```mermaid
flowchart LR
  IN[Channels / Clients / Nodes] --> GW[Gateway]
  GW --> OUT1[路由到 Agent]
  GW --> OUT2[状态/健康/日志]
  GW --> OUT3[协议与安全策略]
```

**Gateway 的三大职责：**

#### 1. 认证（Authentication）

确保只有你授权的平台才能连接

```mermaid
flowchart TB
  X[新设备/新客户端] --> H[Gateway 握手（connect.challenge）]
  H --> A{Token/配对是否通过?}
  A -- 否 --> R[拒绝连接]
  A -- 是 --> OK["建立 WS 连接<br/>授予 role/scopes"]
```

#### 2. 路由（Routing）

把消息送到正确的工作空间

```mermaid
flowchart TB
  MSG[入站消息] --> N[标准化 Envelope]
  N --> M{"匹配 bindings?<br/>(peer/group/accountId...)"}
  M -->|命中| AG[指定 agentId]
  M -->|未命中| DEF[默认 agentId]
  AG --> SK[生成 sessionKey]
  DEF --> SK
```

#### 3. 日志记录（Logging）

记录所有交互，方便调试和审计

```mermaid
flowchart LR
  GW[Gateway] --> L[日志（console + 文件）]
  GW --> H[health/status 诊断面]
  L --> O[排障：config warnings / 插件加载 / 鉴权 / 重启原因]
```

**启动 Gateway 的命令：**

```
# 启动网关，监听 18789 端口
openclaw gateway --port 18789 --verbose
```

### Workspace：你的私人办公室

**Workspace 是什么？**

Workspace（工作空间）是实际处理任务的地方。你可以有多个工作空间，每个负责不同的事情。

```mermaid
flowchart TB
  subgraph WS["Workspace（一个 agent 一个 workspace）"]
    SOUL[SOUL.md / USER.md / AGENTS.md ...]
    MEM[MEMORY.md（可选）]
    DLOG[memory/YYYY-MM-DD.md]
    SKDIR[skills/（该 workspace 私有技能）]
  end
  subgraph AGDIR["Agent Dir（每个 agent 独立）"]
    AUTH[agent/auth-profiles.json]
    SESS[sessions/*.jsonl]
  end
  WS <---> AGDIR
```

**一个典型的 Workspace 配置示例：**

```
~/.openclaw/openclaw.json（概念示例，字段以实际版本为准）
{
  "agents": {
    "list": [
      { "id": "main", "workspace": "~/.openclaw/workspace" }
    ]
  },
  "skills": {
    "entries": {
      "weather": { "enabled": true },
      "calendar": { "enabled": true }
    }
  },
  "channels": {
    "telegram": { "enabled": true }
  }
}
```

**多工作空间使用场景：**

```mermaid
flowchart LR
  subgraph GW["同一个 Gateway"]
    B[bindings 路由规则]
  end
  TG[Telegram] --> B
  WA[WhatsApp] --> B
  B --> A1["agentId: main<br/>workspace: workspace-main"]
  B --> A2["agentId: work<br/>workspace: workspace-work"]
  B --> A3["agentId: family<br/>workspace: workspace-family"]
```

### LLM：AI 的大脑

**LLM 是什么？**

LLM（Large Language Model，大语言模型）是 OpenClaw 的智能大脑，负责理解你的意图和生成回复。

```mermaid
flowchart LR
  AG[Agent] -->|prompt + tools| LLM[模型提供方（云端或本地）]
  LLM -->|tool call / final answer| AG
```

**OpenClaw 支持的 LLM（示例，以你实际配置与版本为准）：**

模型接入与鉴权通常由 `~/.openclaw/openclaw.json` 的 `models` 配置决定。[参考](https://docs.openclaw.ai/gateway/configuration)

| 提供商    | 模型示例        | 特点                 | 适用场景       |
| :-------- | :-------------- | :------------------- | :------------- |
| Anthropic | Claude Sonnet 4 | 平衡、安全、多语言好 | 日常对话、写作 |
| OpenAI    | GPT-4           | 专业、知识广         | 专业任务、分析 |
| DeepSeek  | DeepSeek-V3     | 代码能力强、便宜     | 编程辅助       |
| 本地部署  | Ollama          | 完全私有、免费       | 隐私敏感场景   |

### Channels：连接外部世界

**Channels 是什么？**

Channels（渠道）是连接各种消息平台的"适配器"。

```mermaid
flowchart TB
  P[平台：WhatsApp/Telegram/Slack/Discord/...]
  P --> CH[Channel 适配器（协议/鉴权/收发）]
  CH --> GW[Gateway]
```

官方渠道清单与哪些是插件提供的渠道，见：[Chat Channels](https://docs.openclaw.ai/channels)

**每个 Channel 的工作：**

1. **接收消息**：从平台获取用户消息
2. **格式转换**：统一转换为 OpenClaw 内部格式
3. **发送回复**：把 OpenClaw 的回复发回平台

**示例：Telegram Channel 的工作流程**

```mermaid
sequenceDiagram
  autonumber
  participant TG as Telegram
  participant CH as Telegram Channel
  participant GW as Gateway
  participant AG as Agent

  TG->>CH: update（消息/媒体/回调）
  CH->>GW: 标准化事件 + sender/group 校验
  GW->>AG: bindings 路由 + sessionKey
  AG-->>GW: reply payload
  GW-->>CH: 发送回复
  CH-->>TG: Bot API sendMessage
```

------

## 消息流转过程

现在让我们看看一条完整的消息是如何在 OpenClaw 中流转的：

### 完整消息流程图

```mermaid
flowchart TB
  IN[入站：Channel 收到消息] --> N[标准化消息 Envelope]
  N --> AUTH[鉴权/allowlist/配对/mention 规则]
  AUTH --> ROUTE[bindings 路由到 agentId]
  ROUTE --> SESS[加载 session + memory]
  SESS --> PLAN[LLM 规划：是否调用工具]
  PLAN -->|是| TOOL[执行 Skills/Tools]
  PLAN -->|否| RESP[生成回复]
  TOOL --> RESP
  RESP --> OUT[回发到原 Channel]
```

### 详细步骤解析

**步骤 1-3：消息接收与标准化**

用户在 Telegram 发送消息后，Telegram Channel 会将其转换为 OpenClaw 的标准格式：

```
{
  "platform": "telegram",
  "channel_id": "telegram_123",
  "user": {
    "id": "user_456",
    "name": "Alice"
  },
  "message": {
    "type": "text",
    "content": "上海明天天气?",
    "timestamp": "2024-03-09T10:30:00Z"
  }
}
```

**步骤 4-5：认证与路由**

Gateway 检查这个消息：

- 来源是否已授权？
- 应该路由到哪个 Workspace？
- 用户是否有权限？

**步骤 6-8：AI 理解与决策**

Workspace 准备完整的上下文发送给 LLM：

```
[系统提示]
你是一个个人助理，可以使用以下技能：
- weather: 查询天气
- calendar: 管理日程
- ...

[对话历史]
用户: 你好
助理: 你好！有什么可以帮你的？

[当前消息]
用户: 上海明天天气?
```

**步骤 9-12：技能执行**

LLM 决定调用 weather skill，Workspace 执行并获取结果：

```
{
  "location": "上海",
  "date": "2024-03-10",
  "weather": "晴转多云",
  "temperature": "18-26°C",
  "humidity": "60%",
  "wind": "东风 3-4级"
}
```

**步骤 13-17：生成回复并返回**

LLM 根据天气数据生成自然语言回复，通过原路返回给用户。

------

## 技能系统

### 什么是技能（Skills）？

技能是 OpenClaw 执行特定任务的"能力模块"。如果把 OpenClaw 比作一个人，技能就是这个人学会的各种本领。

```mermaid
flowchart LR
  AG[Agent] -->|选择/匹配| SK[Skill（目录 + SKILL.md）]
  SK -->|教会模型“怎么用工具”| TL[Tools（exec/browser/fs/web/...）]
  TL -->|执行动作| OUT[产出：文件/截图/消息/请求结果]
```

技能的目录与优先级规则（workspace > ~/.openclaw/skills > bundled）见：[Skills](https://docs.openclaw.ai/tools/skills)

### 技能的结构

每个技能包含：

```mermaid
flowchart TB
  SKDIR[skill-folder/] --> MD[SKILL.md（frontmatter + 指令）]
  SKDIR --> FILES[支持文件（提示词、脚本、配置等）]
  SKDIR --> BIN[可选：本地可执行/脚本]
```

**示例：天气技能的定义**

```mermaid
flowchart TB
  FM["YAML frontmatter<br/>name/description/metadata"] --> INST["Markdown 指令<br/>什么时候用、怎么用、注意事项"]
  INST --> TOOLDEF[可用工具（引用系统工具或自定义工具）]
```

### 技能的调用流程

```mermaid
sequenceDiagram
  autonumber
  participant User as 你
  participant Agent as Agent
  participant LLM as LLM
  participant Tool as Tool/Skill

  User->>Agent: 需求（自然语言）
  Agent->>LLM: 上下文 + 技能/工具目录
  LLM-->>Agent: tool call（选择 skill/tool）
  Agent->>Tool: 执行（可能需审批/沙箱）
  Tool-->>Agent: 结果
  Agent->>LLM: 回传结果
  LLM-->>Agent: 最终答复
```

### 技能的安全机制

OpenClaw 对技能有严格的安全控制：

```mermaid
flowchart TB
  P[策略与边界] --> A1[DM/群聊授权：pairing/allowlist/mention]
  P --> A2[工具级审批：exec approvals / pairing]
  P --> A3[沙箱：可选隔离文件/网络/进程]
  P --> A4[插件与技能来源：只启用可信代码]
```

安全模型与注意事项见：[Security](https://docs.openclaw.ai/gateway/security)

**权限类型：**

| 权限         | 说明     | 示例             |
| :----------- | :------- | :--------------- |
| `network`    | 网络访问 | 查天气、搜索网页 |
| `filesystem` | 文件系统 | 读写文件         |
| `email`      | 邮箱访问 | 发送/接收邮件    |
| `calendar`   | 日历访问 | 管理日程         |
| `system`     | 系统操作 | 执行命令         |

------

## 完整工作流程

让我们通过一个真实场景，串联所有概念：

### 场景：定时发送每日天气报告

**需求**：每天早上 8 点，通过 Telegram 收到今日天气 + 日程提醒

```mermaid
flowchart LR
  CRON[cron/定时触发] --> AG[Agent 执行一次任务]
  AG --> W[weather skill/tool]
  AG --> CAL[calendar skill/tool]
  W --> AG
  CAL --> AG
  AG --> SEND[通过 Telegram Channel 发送消息]
```

### 配置代码

```
概念示例（用自然语言 + 技能组合描述计划）
目标：每天 08:00 触发一次
步骤：
1) 查询今天的天气（weather）
2) 列出今天的日程（calendar）
3) 组织成简短报告
4) 通过 Telegram 发送给我
```

### 用户收到的消息

```
早安！今日简报

天气情况
上海今天多云，气温 15-23°C
建议穿着：薄外套
降雨概率：10%

今日日程
- 09:00 - 10:00  团队晨会
- 14:00 - 15:30  客户演示
- 16:00 - 17:00  代码评审

温馨提示
今天有 3 个会议，建议提前准备演示材料。
```

------

## 数据流与状态管理

### 数据如何存储？

```mermaid
flowchart TB
  subgraph StateDir["状态目录（默认 ~/.openclaw，可由 OPENCLAW_STATE_DIR 改）"]
    CONF[openclaw.json]
    LOG[logs/]
    AGD["agents/&lt;agentId&gt;/<br/>- agent/auth-profiles.json<br/>- sessions/*.jsonl"]
  end
  subgraph Workspace["Workspace（每个 agent 一个）"]
    WFILES[SOUL.md/USER.md/AGENTS.md/...]
    WMEM[MEMORY.md（可选）]
    WDAY[memory/YYYY-MM-DD.md]
    WSK[skills/]
  end
  StateDir --- Workspace
```

记忆的“源数据是 Markdown 文件”与 daily log / MEMORY.md 的规则，见：[Memory](https://docs.openclaw.ai/concepts/memory)

**数据类型及存储方式：**

| 数据类型     | 存储方式     | 保留时间         | 示例                 |
| :----------- | :----------- | :--------------- | :------------------- |
| 当前会话状态 | 内存         | 直到会话结束     | 正在进行的对话上下文 |
| 对话历史     | 本地数据库   | 可配置（如30天） | 过去的聊天记录       |
| 用户配置     | 配置文件     | 永久             | API密钥、偏好设置    |
| 技能数据     | 技能自己管理 | 取决于技能       | 日程、邮件草稿       |
| 系统日志     | 日志文件     | 可配置           | 错误、调试信息       |

------

## 扩展性与插件生态

### 如何添加新功能？

OpenClaw 的设计允许轻松扩展：

```mermaid
flowchart LR
  NEED[新需求] --> CHOOSE{扩展方式}
  CHOOSE -->|装技能| SK[Skills（SKILL.md）]
  CHOOSE -->|装插件| PL[Plugins（新增 tools/channels/skills）]
  SK --> USE[Agent 在对话中调用]
  PL --> USE
```

### 技能安装流程

```mermaid
flowchart TB
  FIND[在 ClawHub 搜索/浏览技能] --> INSTALL["clawhub install &lt;slug&gt;"]
  INSTALL --> DIR["写入 &lt;workspace&gt;/skills/&lt;name&gt;/"]
  DIR --> PICKUP[新会话/下一次 agent turn 自动发现]
  PICKUP --> USE[在任务中使用该技能]
```

ClawHub 工作方式与默认安装目录策略见：[ClawHub](https://docs.openclaw.ai/tools/clawhub)

 
