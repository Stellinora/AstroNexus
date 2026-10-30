# 16 · Web 界面

> 状态：**已定方向，Phase 0 落地中**。
> 相关：[14 §3.4](14-云端记忆与多Agent编排.md)、[15-DSH依赖策略](15-DSH依赖策略.md)。

## 一、决策

| 问题 | 结论 |
|---|---|
| 定位 | 开源项目，用户量 10 以内；**先单用户** |
| 范围 | **core 只管云端**；APK / PC 客户端另立项目 |
| 相似度 | 只求**视觉像** DeepSeek 官方网页版；布局沿用 DSH |
| 演进 | UI 跟 DSH 版本走，不独立 fork |
| 实现路线 | 官方 `web` profile + 主题（必要时少量客户端插件） |
| 不做 | fork `apps/web`、独立前端、多租户网关 |

## 二、架构

```text
浏览器 ──HTTPS──▶ Caddy（TLS / 域名）──▶ dsh --profile astronexus-web
                                              └─ memory / reflect / orchestrate 插件
```

- 一个 DSH 实例、一个 `DSH_HOME`、一份 SQLite；单用户先跑通。
- 端口 3080 只在 compose 网络内暴露；对外由 Caddy 终结 TLS。
- 同一用户的 Web / 未来的 APK / PC 都连接这一个实例，天然共享会话与记忆。

## 三、Profile

`astronexus-web` = `@deepseek-ai/dsh-base` + `@deepseek-ai/dsh-web-app` + `@astronexus/dsh-bundle`。

隐私覆盖（`profile-web/cordis.patch.yml`）：

- `session-log-deepseek` 关闭：不上传会话到 DeepSeek。
- 环境变量 `DSH_TELEMETRY_MODE=DISABLED`、`DSH_TELEMETRY_DISABLED=1`：关闭 OTel。

## 四、安全边界

- 启动 URL 里的 token 是**进程凭据**，只能给单个用户；不得贴到公开渠道。
- 对外必须 TLS；`--trusted-host` 与 `--public-url` 必须与反代域名一致。
- 未来的多用户/APK 需要单独的登录与实例路由层；不在 core 内实现。

## 五、分期

1. **Phase 0**：web profile、镜像、compose、Caddy 示例、隐私开关。
2. **Phase 1**：视觉主题（覆盖 `--dsw-*` token、logo、文案），先做客户端插件构建 spike，失败则用反代注入 CSS 兜底。
   已落地：`services/dsh/client-theme` 手写 client bundle（无需构建链），覆盖品牌蓝 `#4d6bfe` 与明暗底色；后续按截图调色。
3. **Phase 2**：记忆检索面板、`memory_*` 工具卡片（`tool.call.toolview`）。
4. **出界**：APK、PC 壳、gateway、多用户、iOS。

## 六、验收

```bash
dsh --profile astronexus-web --dump-config   # 有 webserver，session-log-deepseek disabled
docker compose -f docker/compose.yml config
```

本机 `dsh --profile astronexus-web --no-open --port 3080` 打开返回 200。

已验证（WSL + Docker Desktop）：`docker compose build` 成功，容器 `3080` 返回 200，
`session-log-deepseek` 为 disabled。
