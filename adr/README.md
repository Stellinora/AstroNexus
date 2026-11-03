# 架构决策记录（ADR）

本目录记录不可轻易推翻的工程决策：为什么这样选、放弃了什么、什么时候需要重审。

| 编号 | 决策 |
|---|---|
| [0001](0001-dsh-as-host.md) | DSH 作为云端宿主，domain 内核保持 DSH-free |
| [0002](0002-sqlite-single-writer.md) | 单用户阶段用 SQLite 单写者，接口隔离存储 |
| [0003](0003-knowledge-vs-experience.md) | 知识层与经验层分表、分检索、分注入 |
| [0004](0004-async-reflection.md) | 反思异步、episode 驱动、失败不推进游标 |
