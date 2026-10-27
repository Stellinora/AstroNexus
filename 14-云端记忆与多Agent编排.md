# 14 · 云端记忆与多 Agent 编排系统设计总结

> 状态：**当前架构基线**。与仓库内旧文档冲突时，以本文为准。
>
> 核心目标：构建一个**独立于任何 Agent 的云端记忆与编排系统**，通过薄插件接入 OpenCode、Claude Code、手机 App、无头 QQ 等客户端。

---

## 1. 最终目标架构

```text
┌──────────────────────────── Docker 云端 ────────────────────────────┐
│                                                                     │
│  ┌──────────────────────┐      ┌──────────────────────────────┐    │
│  │ 记忆整理 Worker       │      │ DSH 云端编排服务              │    │
│  │（DSH Headless）       │      │ - 对话入口                    │    │
│  │ - 反思/总结/经验提取   │      │ - 决策与任务拆解              │    │
│  │ - 去重/矛盾检测/摘要   │      │ - 检索知识库/经验库            │    │
│  │ - 知识库/经验库生产    │      │ - 通过 ACP 插件调用本地 Agent  │    │
│  └──────────┬───────────┘      └──────────────┬───────────────┘    │
│             │                                 │                    │
│             ▼                                 ▼                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 统一存储与接口层                                              │  │
│  │ - 原始会话记录（跨框架归一化）                                 │  │
│  │ - 知识库（事实、项目背景、偏好）                               │  │
│  │ - 经验库（流程、教训、阻塞模式、有效策略）                     │  │
│  │ - 统一 API / MCP Server                                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ MCP 能力层（挂在云端编排层）                                   │  │
│  │ - 文档搜索：gitea-search / deepddw 等                         │  │
│  │ - Word 处理：office-toolkit / wordcraft / office-com 等       │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
              ▲                    ▲                    ▲
              │ 薄插件             │ HTTP/MCP           │ HTTP/MCP
      ┌───────┴───────┐    ┌───────┴───────┐    ┌───────┴───────┐
      │ OpenCode      │    │ 自有手机 App   │    │ 无头 QQ 等     │
      │（本地执行）    │    │（控制中枢）    │    │（对话渠道）    │
      └───────────────┘    └───────────────┘    └───────────────┘
```

### 一句话定位

- **Docker 是云端唯一真相源和记忆工厂**：会话同步、跨框架归一化、记忆整理、知识/经验生产全部在 Docker 内完成。
- **DSH 是 Docker 内的云端编排层**：负责对话入口、决策、记忆检索和 Agent 调度。
- **OpenCode 等是本地执行 Agent**：通过 ACP 等协议被云端委派任务。
- **Agent 插件是薄适配器**：只负责上报会话、调用检索、接收委派，不承担记忆整理和核心同步逻辑。
- **手机 App 是云端控制中枢**：类似 DS 官方 App，共享同一账户和云端对话记录。

---

## 2. 已确认的核心决策

| 决策 | 结论 |
|---|---|
| 记忆系统主体 | 独立 Docker 核心，不把主体做进 DSH 插件 |
| DSH 的角色 | Docker 内部云端编排层 + Headless 记忆整理运行时 |
| 本地执行 Agent | 当前 OpenCode，未来可加入 Claude Code 等 ACP Agent |
| 插件职责 | 薄适配：上报、检索、接收委派；不做反思和归一化 |
| 反思与总结 | 在 Docker 内异步运行，独立于对话路径 |
| 跨框架归一化 | 在 Docker 内完成，插件只上报原始数据 |
| 对话入口 | 用户直接与云端模型对话，云端模型编排本地 Agent |
| 端云对话记录 | 云端在记忆 Docker 内一起做，独立于任何 Agent |
| 手机端优先级 | 先做控制中枢，再做 root/非 root 系统接口 |
| 知识/经验建模 | 知识库与经验库分开建模，消费方式不同 |
| MCP 能力 | 挂在云端 DSH 编排层，不分散到本地 Agent 插件 |

---

## 3. 分层职责

### 3.1 Docker 记忆核心

负责：

- 接收所有 Agent 上报的原始会话数据。
- 按 `agent_id` / `framework_tag` 做跨框架归一化。
- 存储原始记忆、知识库、经验库。
- 运行异步反思流水线：
  - 评分
  - 去重
  - 矛盾检测
  - 摘要
  - 偏好提取
  - 高价值记忆晋升
  - 陈旧记忆衰减
- 暴露统一检索 API / MCP Server。
- 维护端云同步状态、设备映射、权限隔离。

### 3.2 DSH 云端编排服务

负责：

- 作为用户对话入口。
- 检索知识库/经验库，把相关记忆注入决策上下文。
- 拆解任务，决定委派给哪个本地 Agent。
- 通过 ACP 插件调用本地 Agent。
- 维护任务状态：`pending / running / done / failed`。
- 汇总结果并返回给用户。
- 可选：把委派运行元数据回写为原始信号，供反思 Worker 消费。

> **边界**：DSH 的版本与插件 API 不构成 AstroNexus 的对外契约。核心逻辑放在独立服务中，DSH 通过 MCP / HTTP 调用；升级与 patch 策略见 [15-DSH依赖策略](15-DSH依赖策略.md)。

### 3.3 Agent 薄插件

负责：

- 监听本地会话事件，例如会话空闲、每轮结束。
- 向 Docker 上报原始消息、工具调用、文件变更等。
- 调用 Docker 的 `knowledge.search` / `experience.search`。
- 接收云端委派并触发本地 Agent 执行。
- 回传执行结果和运行元数据。

不负责：

- 反思。
- 总结。
- 跨框架归一化。
- 同步冲突解决。
- 记忆存储策略。

### 3.4 手机 App

负责：

- 作为云端控制中枢，体验对标 DeepSeek 官方 App。
- 同一账户下查看跨设备、跨 Agent 的会话列表。
- 实时查看消息流、任务状态、审批请求。
- 远程发消息、接管审批、继续会话。
- 后续叠加 root / 非 root 系统自动化能力。

---

## 4. 关键推理过程

### 4.1 为什么不要为每个 Agent 写完整插件？

不同 Agent 插件机制差异大：OpenCode、Claude Code、Codex、手机端、QQ 机器人各不相同。如果每个都写完整记忆插件，反思、检索、去重、同步、权限逻辑会重复多份，后续修改一个策略要改所有插件。

**结论：统一核心 + 中间夹层 + 薄适配器。**

### 4.2 为什么反思必须独立于对话路径？

反思是离线的、重量的，需要专门“复习时间”；对话是在线的、低延迟的。业界成熟方案都把记录和整理分开：

- Claude Code 的睡眠整合在后台子 Agent 异步执行。
- claw-reflect 是独立记忆蒸馏引擎。
- neobrain 有独立 daemon 生命周期。
- agent-dreams 的认知睡眠周期跑在服务端。

**结论：反思 Worker 放在 Docker 内异步运行，不阻塞 Agent 对话。**

### 4.3 为什么 DSH 适合做 Docker 内的编排/整理层？

- DSH 有 Headless 模式，适合容器内无界面运行。
- 有插件生态可复用记忆整理能力。
- 有 ACP 相关插件可调用本地 Agent。
- 插件化设计适合把“记忆整理”和“编排调度”做成独立服务。

**边界：DSH 是 Docker 内部运行时，不是记忆系统主体，也不是本地执行 Agent。**

### 4.4 为什么跨框架归一化必须放在 Docker 内？

不同 Agent 的会话结构不同。如果让每个插件自己适配 Docker 写入格式，每接一个新 Agent 都要重写归一化逻辑。

**正确做法：Docker 定义框架无关的会话 schema，插件只上报原始数据 + 框架标签，Docker 内部用解析适配器归一化。**

### 4.5 为什么实时对话同步不经过记忆整理链路？

实时消息需要低延迟、流式、原生支持；记忆整理是异步、重量、可失败重试的。如果让记忆整理同时承担实时同步，会出现双重写入、延迟叠加、冲突复杂度上升。

**结论：实时对话走云端会话服务/SSE；记忆整理异步消费，不参与实时同步。**

### 4.6 为什么知识库和经验库要分开？

- 知识库：事实性、相对稳定，例如项目技术栈、用户偏好，适合会话开始时主动注入。
- 经验库：过程性、情境相关，例如某类迁移的阻塞模式、API 限流退避策略，适合遇到相似情境时按需检索。

**结论：两个库分开建模，插件侧对应两个检索入口和不同注入策略。**

### 4.7 为什么手机端先做控制中枢，再做系统接口？

手机端首先是“远程续聊、随时决策”的控制中枢，类似 DS 官方 App。root / 非 root 系统接口是能力扩展层，不应拖累核心体验。

系统接口建议按能力层级设计，而不是按用户类型分叉：

- Accessibility：零门槛保底。
- Shizuku：非 root 用户的准系统权限。
- ROOT：最高优先级，自动启用。

统一控制器对外暴露相同工具接口，内部按当前设备能力路由。

---

## 5. 接口设计建议

### 5.1 统一数据模型

建议至少包含：

```text
user_id
device_id
agent_id
framework_tag
project_id
conversation_id
memory_type: preference | decision | blocker | workflow | reflection | fact
content
source_message_ids
confidence
visibility: private | project | global
created_at / updated_at / expires_at
embedding（可选）
sync_version
```

### 5.2 核心 API / MCP 工具

```text
knowledge.search(query, project_id, type)
experience.search(query, context)
memory.remember(content, type)
session.ingest(session_id, messages)
memory.forget(id)
memory.reflect()        # 可选，手动触发
sync.status()
delegate(agent_id, task, mode="async" | "sync")
task.status(task_id)
```

### 5.3 编排委派接口

```text
delegate(agent_id, task, mode)
  -> task_id

task.status(task_id)
  -> pending | running | done | failed

task.output(task_id)
task.kill(task_id)
```

底层协议可以是 ACP、SSH、HTTP，由插件适配层封装。编排服务只面对统一 `agent_id`。

---

## 6. 可参考的插件与实现

> 以下为讨论中提到的参考实现，不作为硬依赖，需按实际成熟度和版本验证。

### 6.1 DSH 调用本地 Agent

- `dsh-subagent-opencode`：本机/SSH 远端，异步 job id，完成通知。
- `dsh-opencode-acp`：OpenCode 作为 ACP 子代理，记录 session id、模型、耗时、token。
- `dsh-subagent-acp-ui`：多 ACP Agent 配置页、握手测试、路由 skill。
- `@acpus/dsh`：DSH 作为 Supervisor，编排 Codex、Claude、OpenCode 等。
- `dsh-agent-bridge`：桥接已有 coding-agent CLI。

### 6.2 记忆整理与反思

- `meow-memory`：dream 机制、空闲触发、分层记忆、SQLite 零原生依赖。
- `dsh-mneme`：自动巩固、去重、合并、冲突裁决、Markdown 镜像。
- `claw-reflect`：独立记忆蒸馏引擎，多阶段流水线。
- `opencode-memory-dream`：会话空闲提取、系统提示注入、显式记忆工具。
- `opencode-semantic-memory`：全文 + 向量混合检索。
- `opencode-mcp-memory`：多种记忆类型。

### 6.3 对话树与归档

- `@mtayfur/opencode-chat-tree`：轻量树形导航和分支。
- `opencode-context-tree`：Git 式分支、合并、时间线。
- `hivemind-session-archive`：整棵会话树持久化为 JSONL + SQLite，供反思消费。

### 6.4 端云同步与手机控制

- `@opencode-sync`：私有 Git 仓库同步会话/配置，向量时钟冲突解决，离线友好。
- `@pugi/plugin-cloud-sync`：基于 Yjs CRDT 的实时跨端接续。
- `opencode-sync-plugin`：云端存档与仪表盘。
- `opencode-webui`：可深度扩展的 Web 前端。
- `Pixcode`：多 CLI 统一控制室，文件/终端/Git/编排集成。
- `Portal` / `OpenChamber`：移动优先/美观的远程访问。
- `Otto` / `Armada` / `opencode-pilot` / `entangle` / `pocketpilot`：手机控制参考。

### 6.5 无头 QQ

- NapCat：无头 QQ 协议端。
- AstrBot / NoneBot2 / ncatbot：机器人框架。
- `qqcat`：支持 MCP Server 接入。
- `dsh-channel-qq`：QQ 深度接入 DSH 的参考。

### 6.6 MCP 文档与 Word

- `dsh-mcp-oauth-client`：图形化管理 MCP，Streamable HTTP / stdio。
- `dsh-plugin-kit` MCP 卡片：保存即热加载。
- `dsh-workspace-mcp`：项目级 MCP 配置，随项目隔离。
- `dsh-gitea-search`：SQLite FTS5 + Jieba，代码/文档全文检索。
- `deepddw`：知识库 + 文档检索 MCP。
- `dsh-office-toolkit`：纯 JS 读写 docx/xlsx/csv，零 Office 依赖。
- `dsh-wordcraft`：侧边栏真实 Word 实时编辑。
- `@eqman00003/dsh-office-com`：Windows COM 驱动真实 Office，支持 VBA/宏/公式。

---

## 7. 手机端设计要点

### 7.1 第一层：控制中枢

- 设备配对与访问控制。
- 跨机器会话列表。
- 会话切入与继续。
- 远程审批与提问接管。
- 状态可见性：进度、工具结果、长任务通知。
- 同一账户下共享云端对话记录。

### 7.2 第二层：系统接口

按能力层级统一控制器：

```text
ROOT > Shizuku > Accessibility
```

- Accessibility：零门槛保底。
- Shizuku：非 root 用户通过 ADB 授权获得 shell 级权限。
- ROOT：自动检测，最高优先级。
- 对外暴露统一工具：`tap`、`input`、`open_app`、`screenshot` 等。

### 7.3 与云端关系

- 手机 App 是 Docker 云端的直接客户端。
- 对话记录以云端为唯一真相源。
- 手机不需要知道本地 OpenCode 的存在。
- 系统自动化工具是手机本地执行器，由云端下发的指令驱动。

---

## 8. 推荐落地顺序

1. **定义统一协议和数据模型**：会话、记忆、知识、经验、任务状态。
2. **Docker 核心最小闭环**：会话上报、归一化、存储、检索 API/MCP。
3. **接入第一个本地 Agent**：OpenCode 薄插件，验证上报 + 检索 + 委派。
4. **DSH Headless 记忆整理**：挂载记忆插件，验证异步反思和知识/经验生产。
5. **DSH 云端编排服务**：对话入口、记忆检索注入、ACP 委派本地 Agent。
6. **手机 App v1**：同一账户、会话列表、消息流、审批接管。
7. **无头 QQ 渠道**：NapCat + 框架 + MCP 接入。
8. **MCP 能力层**：文档搜索、Word 处理按需挂载。
9. **对话树与归档**：前端导航 + 归档供反思消费。
10. **多 Agent 扩展**：Claude Code 等 ACP Agent 纳入委派目标。

---

## 9. 风险与边界

| 风险 | 应对 |
|---|---|
| DSH 插件 API 变更（preview 破坏性升级） | 核心逻辑不放进 DSH 插件；锁定 exact 版本 + pnpm patch，升级先跑契约测试（见 [15](15-DSH依赖策略.md)） |
| 插件重复建设 | 统一核心 + 薄适配器；归一化在 Docker 内 |
| 反思阻塞对话 | 反思异步 Worker，独立队列和限流 |
| 实时同步与记忆整理冲突 | 实时对话走会话服务/SSE；记忆整理异步消费 |
| 手机端权限复杂 | 先控制中枢，后按能力层级叠加系统接口 |
| MCP 工具过多干扰编排 | 项目级 MCP 隔离，按需挂载 |
| 多框架会话结构差异 | Docker 内框架无关 schema + 解析适配器 |
| 编排与反思抢 LLM 预算 | 分开部署、分开限流、可降级模型 |

---

## 10. 一句话总结

**Docker 是云端唯一真相源、记忆工厂和编排中枢；DSH 在 Docker 内负责记忆整理与云端编排；OpenCode 等本地 Agent 通过薄插件被委派执行；手机 App、无头 QQ 等只是云端的控制/对话客户端。所有跨框架差异、反思总结、同步归一化都封装在 Docker 内部，插件永远保持薄。**
