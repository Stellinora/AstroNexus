# 15 · DSH 依赖策略

> 状态：**已定**。适用于 `core`（独立仓库 `AstroNexus-core`）。
> 相关：[14 §3.2 / §9](14-云端记忆与多Agent编排.md)、[11 #24](11-未决问题与待办.md)、[13 §2.3](13-双项目拆分.md)。

## 一、事实基线

| 项 | 值 |
|---|---|
| 全称 | DeepSeek Harness（`dsh`） |
| 来源 | https://github.com/deepseek-ai/deepseek-harness |
| 许可 | MIT |
| 技术栈 | TypeScript + Cordis 微内核，「一切皆插件」 |
| 成熟度 | **developer preview，官方声明会有破坏性变更** |
| 本仓库锁定 | `@deepseek-ai/dsh@0.2.1-rc.2`（exact，不含 `^` / `~`） |
| 组成方式 | profile（`$DSH_HOME/profiles/<name>`）+ bundle（`package.json` 的 `dsh.bundle.patch`）+ 分层 `cordis.patch.yml` |

DSH 的官方应用面是 `web / headless / sdk / sdk-minimal / acp`。AstroNexus 不新增应用面，只在既有 profile 上叠加自己的 bundle。

## 二、决策

1. **DSH 是外部运行时依赖，不 fork。**
2. **不 vendor 源码**（不把上游仓库拷进 `core`）。
3. 需要对上游做小改动时，走 **`pnpm patch`**，patch 文件进仓库。
4. **`@deepseek-ai/*` 的 import 只允许出现在 `core/services/dsh/bundle/src/{memory,reflect,orchestrate}` 插件入口**；
   `domain/`、`packages/`、`plugins/`、`mcp/` 保持运行时无关。

反向决策（什么时候才 fork）见第六节。

## 三、依赖边界

| 目录 | 可否引用 `@deepseek-ai/*` | 说明 |
|---|---|---|
| `services/dsh/bundle/src/{memory,reflect,orchestrate}` | 允许 | 插件入口：只做 `apply(ctx)` 注册与适配 |
| `services/dsh/bundle/src/domain` | **禁止** | 存储、检索、反思算法的纯 TS 内核 |
| `services/dsh`（profile / Dockerfile / bundle manifest） | 允许 | 组装与打包，不含业务逻辑 |
| `packages/protocol` | **禁止** | 契约层，不感知任何运行时 |
| `packages/schema` | **禁止** | 框架无关会话 schema |
| `plugins/` | **禁止** | Agent 侧薄适配器，不给本地 Agent 下发 DSH 依赖 |
| `mcp/` | **禁止** | MCP 能力层按配置挂载，与 DSH 解耦 |

守卫方式：`core/.dependency-cruiser.cjs` 的 `no-dsh-outside-adapter` 规则，CI 中 `pnpm run guard` 失败即红。

## 四、版本与升级

- 依赖一律 **exact pin**，提交 lockfile（`pnpm-lock.yaml`）。
- DSH 版本双重锁定：`services/dsh/Dockerfile` 的 `ARG DSH_VERSION` 与
  `services/dsh/bundle/package.json` 的 `engines.dsh` 必须是同一个精确版本。
- 容器基础镜像按 digest 固定（已固定 `node:24-bookworm-slim@sha256:c86ecead…`，linux/amd64）。
- 升级 checklist：
  1. 读上游 release notes，标记破坏性变更；
  2. 在分支上升 DSH 版本，保留旧版本对比；
  3. 跑契约测试（`packages/protocol` 一致性）与 `pnpm run check`；
  4. 跑 `dsh --profile astronexus --dump-config`，确认组合仍成立；
  5. 记录版本、变更点、回滚方式。

## 五、patch 流程

- patch 用 `pnpm patch @deepseek-ai/dsh@<version>` 生成，`.patch` 文件放 `core/patches/`。
- 镜像里的 DSH 由 `npm -g` 安装，因此 patch 由 `services/dsh/Dockerfile`
  在安装后统一应用（`patch -p1`），不走 `pnpm.patchedDependencies`。
- 每个 patch 必须在 `core/patches/README.md` 登记：**原因 / 影响面 / 移除条件 / 验证方式**。
- 顺序：先提上游 PR；上游不接或等不及，才落 patch。
- 每次升级 DSH 后，先尝试删除 patch 再跑测试；能删就删。

## 六、fork 门槛

以下四条**全部满足**才 fork：

1. 必须修改 DSH 内核语义（agent loop、session 持久化、插件加载顺序等）；
2. 现有 plugin seam / bundle 层无法表达该需求；
3. 上游明确拒绝或长期不合并；
4. patch 冲突已连续跨两个 rc 版本。

fork 的硬性约束：

- 独立仓库（如 `AstroNexus-dsh`），`upstream` remote 指向官方；
- 业务逻辑仍然放在 `core/services/dsh` 的 bundle / 插件里，**不进 fork**，保证可随时切回官方；
- patch 以序列维护，每个版本可重放。

## 七、与 13 / 14 的关系

[13 §2.3](13-双项目拆分.md) 要求「不让插件承担反思、总结、归一化、同步」。本策略把它落到依赖层面：

- **工程主体**是 DSH 进程：memory / reflect / orchestrate 三个插件同进程运行，共享 `ctx` 与一个 SQLite 连接。
- **内核**是 `services/dsh/bundle/src/domain` 与 `packages/protocol`、`packages/schema`：不依赖 DSH，可单独测试，也可换运行时复用。
- 插件入口只做注册，因此 DSH 预览版波动被限制在三个入口文件，不传导到记忆算法与对外 API。

## 八、待办

- [ ] 首次 `docker build` 后补基础镜像 digest 与 DSH 版本校验脚本
- [ ] `packages/protocol` 契约测试
- [ ] 上游 rc 升级的定期检查（手动或 CI 定时任务）
