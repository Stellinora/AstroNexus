# AstroNexus · 云端记忆与多 Agent 编排

> 从 Vesperine 拆出的助手项目。

AstroNexus 的目标是构建一个**独立于任何 Agent 的云端记忆与多 Agent 编排系统**，通过薄插件接入 OpenCode、Claude Code、手机 App、无头 QQ 等客户端。

- 来源仓库：https://github.com/Stellinora/Vesperine
- 兄弟项目：生命项目（Vesperine 的 `docs/life/`）
- 当前架构基线：[14-云端记忆与多Agent编排](14-云端记忆与多Agent编排.md)
- 本仓库文档为助手项目私有副本；共享文档修改时需与 Vesperine 同步。

---

## 核心架构

- **Docker 是云端唯一真相源和记忆工厂**：会话同步、跨框架归一化、记忆整理、知识/经验生产全部在 Docker 内完成。
- **DSH 是 Docker 内的云端编排层**：负责对话入口、决策、记忆检索和 Agent 调度。
- **OpenCode 等是本地执行 Agent**：通过 ACP 等协议被云端委派任务。
- **Agent 插件是薄适配器**：只负责上报会话、调用检索、接收委派，不承担记忆整理和核心同步逻辑。
- **手机 App 是云端控制中枢**：共享同一账户和云端对话记录。
- **无头 QQ 等是对话渠道**：只做入口/出口，不做记忆整理。

---

## 文档

### 当前基线

- [00-定位与核心命题](00-定位与核心命题.md)
- [14-云端记忆与多Agent编排](14-云端记忆与多Agent编排.md)
- [15-DSH依赖策略](15-DSH依赖策略.md)
- [16-Web界面](16-Web界面.md)
- [13-双项目拆分](13-双项目拆分.md)
- [11-未决问题与待办](11-未决问题与待办.md)
- [术语表](术语表.md)

### 历史与背景

以下文档保留为背景参考；与 14 冲突时，以 14 为准。

- [02-记忆架构](02-记忆架构.md)
- [05-Episode与经验](05-Episode与经验.md)
- [07-安全模型](07-安全模型.md)
- [09-人与人的通信](09-人与人的通信.md)
- [10-开源参考与借鉴](10-开源参考与借鉴.md)
- [12-执行体与租约](12-执行体与租约.md)

---

## 跨项目

- [Vesperine 生命项目](https://github.com/Stellinora/Vesperine/blob/main/docs/life/README.md)
