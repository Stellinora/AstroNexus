# 05 · Episode 与经验

> Episode 是「事件」的一等公民。运维故障是 Episode，共同经历也是 Episode。

## 一、为什么需要 Episode

普通助手记「事实」，运维/共同经历必须记「事件」。

- 一次故障持续 3 小时，跨多个用户会话、多次 LLM 调用、用户中途离开又回来、告警持续触发
- 会话只是「事件」的切片，主体需要在**事件维度**维持状态

---

## 二、Episode 生命周期

```
告警触发
  → 创建 Episode { id, start_ts, trigger, status: Open }
  → 持续 append 观测（日志、指标、用户操作）
  → 用户 / Agent 做出处理
  → 状态变更：Open → Investigating → Mitigating → Resolved
  → 触发总结：LLM 生成 { root_cause, timeline, lessons }
  → 落 L1，同时触发 L3 模式挖掘
```

---

## 三、数据结构

```sql
CREATE TABLE episode (
  id           UUID PRIMARY KEY,
  title        TEXT,
  start_ts     TIMESTAMPTZ,
  end_ts       TIMESTAMPTZ,
  status       TEXT,            -- open / investigating / mitigating / resolved
  severity     TEXT,
  services     TEXT[],
  trigger      JSONB,
  timeline     JSONB,           -- 观测流
  root_cause   TEXT,
  resolution   TEXT,
  embedding    VECTOR(1024),
  importance   REAL,
  source_refs  JSONB,           -- 证据链：日志ID / 指标查询 / 工单
  owner        JSONB,           -- { type: person/service/agent, id } —— 归属的是「人」
  scope        TEXT             -- personal | ops | life
);
```

> 注意 `owner` 指向的是**基座（人）**，不是角色卡。Episode 归属主体；`scope` 只表示它产生于哪种角色语境。
>
> 极简路线下，编程场景的 Episode 语义改为「一次任务」：`services` → `repo/branch`，`root_cause` → `decision_reason`；只记元记忆，不记工具调用流。见 [12 §7.5](12-执行体与租约.md)。

---

## 四、召回模式

```
enum RecallMode {
    Similar  { episode_id },     // 相似 Episode：当前故障 → 找历史相似
    Semantic { query },          // 语义检索
    ByService{ service },        // 服务维度
    Recent   { days },           // 时间线回顾
}
```

「上次这个服务挂是怎么处理的」直接走 `Similar`，返回历史 Episode 的 `root_cause + resolution`——这是运维最高频、最有价值的召回。

---

## 五、经验层 / 模式挖掘

从多次 Episode 归纳模式：

```
L1 Episode 记忆（具体）：
  · "3月15日 14:20，服务A响应时间飙升到 2000ms"
  · "3月16日 09:10，服务A又出现类似问题"
  · "3月18日 22:00，服务A在流量高峰时再次异常"

L3 经验层（抽象）：
  · "服务A在流量高峰时容易出问题"
  · "排查服务A异常时，先看连接池和慢查询"
  · "服务A的异常通常在 10-15 分钟内自愈，除非流量持续"
```

**反馈闭环不可省**（见 [02 记忆架构](02-记忆架构.md) 第六节）：经验被使用时记录效果，效果差则降权、持续差则淘汰。

---

## 六、领域术语学习（黑话抽取）

```
观测到：用户说 "pg 又抖了"
  → 候选术语: pg
  → 上下文: 与 PostgreSQL、连接池、慢查询共现
  → 累计 N 次后推断: pg = PostgreSQL 服务
  → 存入术语表
  → 下次用户说 "pg" 时，Agent 自动展开
```

运维领域这类术语极多：服务代号、集群名、内部缩写、历史遗留命名。自动学习比人工维护词表更可持续。
