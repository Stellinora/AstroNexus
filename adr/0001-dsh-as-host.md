# ADR 0001 · DSH 作为云端宿主，domain 内核保持 DSH-free

- 状态：已采纳
- 日期：2026-10

## 背景

需要一个云端长期进程，承载对话入口、记忆整理、多 Agent 编排；同时要能承受 DSH 预览版的破坏性变更。

## 决策

- 云端只跑 DSH 进程，memory / reflect / orchestrate 以插件形式挂载。
- 业务算法、存储、schema 放在 `services/dsh/bundle/src/domain/`，不 import `@deepseek-ai/*`。
- 插件入口只做 `apply(ctx)` 注册与适配，不写算法。

## 放弃的方案

- **全自研 harness**：要重建 loop、工具管线、会话格式、ACP/MCP、沙箱，成本远高于差异化收益。
- **fork DSH**：预览版迭代快，rebase 成本高；插件 seam 已能表达需求。

## 后果

- DSH 升级的爆炸半径被限制在三个入口文件。
- 插件入口是运行时适配层，必须靠测试覆盖，不能靠类型检查兜底。
- 如果 DSH 无法表达某项需求，先走 patch；连续两个 rc 都冲突才考虑 fork。

## 重审条件

多租户、横向扩展、DSH 入口连续破坏、反思抢占对话资源。
