# Cluster-Cooperation 方案（v2 · 源码核实版）

> 在 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（**dsh**）之上做二次开发，
> 参照 [wangzuke/open-Agent-Team](https://github.com/wangzuke/open-Agent-Team)（**OAT**）的协作范式，
> 做一个**靠配置驱动模型、角色、拓扑与编排策略的 Agent 集群运行时**。

**基线已落地（本机）**

| 项 | 值 |
|---|---|
| 上游基线 | `dsh-0.1.6-alpha.2`（2026-09-17 的发布合并提交） |
| 本地路径 | `deepseek-harness/`（工作区子树） |
| 工具链 | Node **v24.21.0**（`C:\Program Files\nodejs`，未进 PATH）/ npm 11.19.0 / **pnpm 11.7.0**（`%APPDATA%\npm`）/ git 2.45.1 |
| 安装 | `pnpm install` ✅ 12m52s |
| 构建 | `pnpm run build` ✅ EXIT=0（含 tsdown Host+Client、Typert、Web 前端 248 个产物） |
| `--dump-config` | ✅ 五个 profile 均可导出插件树 |

> **每个新 shell 需先执行**：`$env:PATH="C:\Program Files\nodejs;$env:APPDATA\npm;$env:PATH"`
> （或一次性 `setx` 写入用户 PATH，待你确认。）

---

## 1. 结论先行：这活比想象中小，但价值集中

调研前我以为要从零搭多 Agent 内核；核实后结论相反——**dsh 上游已经把 OAT 的骨架几乎全做完了**，
而且做得比 OAT 更彻底（状态是落在 Session 事件日志里可重放的，不是落 JSON 文件）。
但有一件事它明确**没有做**，而这恰好是你的核心诉求：

> **每个 Agent 用哪个模型，dsh 没有"按角色配置"的说法；多 Agent 编排策略也没有一层声明式配置。**

所以本项目的真实定位是：

| 不做 | 要做 |
|---|---|
| roster / mailbox / 任务 DAG / 9 个工具 / Web UI（**上游已有**） | **配置层**：一份 `cluster.yml` 声明模型、角色、拓扑、预算 |
| 自己写 OpenAI/Anthropic adapter（**`dsh-llm-pi-ai` 已有**） | **模型路由层**：按角色/阶段/预算把请求改道到指定模型，含降级链 |
| 重写 agent loop | **编排层**：briefing、依赖自动解锁、轮次调度、评审回路、压缩与预算 |
| 从零写聊天 UI（**`client-ui-agent-team` 已有**） | **对话层**：广播 / 圆桌 / 辩论 / 投票等群对话协议 |

---

## 2. 能力盘点：上游已提供 vs 我们要建

### 2.1 上游已提供（直接拿来用，不改）

| 能力 | 实现位置 | 说明 |
|---|---|---|
| Team 域服务 | `packages/experimental/agent-team` → `ctx.agentTeams` | `membership / listMembers / spawnTeammate / sendMessage / createTask / getTask / listTasks / updateTask / waitForChange / interrupt / tryMembership` + 3 个 Remote 方法 |
| 持久化三件套 | 同上 | `TeamMemberSnapshot`（roster）、`TeamMessageSnapshot`+`TeamMessageSource`（去重邮箱）、`TeamTaskSnapshot`（`revision` CAS + `blockedBy` DAG + `writeScopes`） |
| 重放 | `foldTeam()` | 从根 Session 日志重放出 roster / 看板 / 未投递邮箱 |
| 模型可用工具（9 个） | `packages/experimental/tool-agent-team` | `spawn_teammate`、`send_message`、`list_agents`、`wait_agent`、`interrupt_agent`、`team_task_create/list/get/update` |
| Profile 层 | `packages/experimental/agent-team-profile` | patch：禁用 4 个 legacy subagent 工具行，插入 team service + tool 行 |
| Web UI 层 | `packages/experimental/client-ui-agent-team` | 浏览器端 roster / 任务板（可被 Cluster 面板复用扩展） |
| **多供应商模型** | `packages/llm/llm-pi-ai` | `openai-completions` / `openai-responses` / `anthropic-messages` 三种协议，cordis.yml 里声明 route + `apiKeyEnv` 即可，**不用自己写 adapter** |
| **单 Agent 绑定模型** | `packages/core/agent/src/model-selection.ts` → `installModelSelection(agentCtx, ref)` | 官方实现，ACP 已在生产用。注册 `system-prompt/assemble` + `agent/request` + `agent/pre-step` 三个 scoped 监听，返回单一 disposer |
| 凭据 | `ctx.credentials` | `resolve(ref)` 每次请求重新解析；**配置只放环境变量名，禁放字面量** |

### 2.2 上游明确没有 / 做不到（我们的落点）

| 缺口 | 证据 | 我们的做法 |
|---|---|---|
| **per-teammate 模型绑定** | `SpawnTeammateRequest`（`agent-team/src/types.ts:144`）只有 `name/description/prompt/context/provider/signal`，**没有 `agentOptions`**；子 Agent 默认继承父 route | 见 §4.2：监听 `agent/created` + `installModelSelection` |
| **声明式团队配置** | 全靠 cordis.yml 手写 patch 行 | `cluster/config` + `cluster.yml` |
| **结构化 briefing** | `spawn_teammate` 只有 name/description/prompt | `cluster/orchestrator` 生成 mission / deliverables / definition_of_done / quality_bar |
| **依赖自动解锁** | 有 `blockedBy` DAG，但**没有**"解锁后通知 owner" | orchestrator 监听 CAS 提交 → 重算 ready → 自动 `sendMessage` |
| **群对话协议** | 只有 1:1 `send_message` | `cluster/dialogue` 做广播/圆桌/辩论/投票 |
| **预算与成本账** | 无集群级预算 | `cluster/telemetry` + 预算硬闸 |
| **自动组队** | 官方明令："普通任务不触发 delegation，除非用户显式要求" | 我们的 profile 默认就跑 team，绕过这条限制 |

### 2.3 必须接受的硬约束（写进设计）

- `maxMembers: 8`、`maxTasks: 256`、`maxPendingMessagesPerMember: 64`、`maxMessageBytes: 65536`、`disposalTimeoutMs: 5000`（可在 patch 里调）。
- **共享工作目录**：所有 teammate 看同一个 checkout，**无 worktree 隔离、无文件锁**；`writeScopes` 是 advisory，挡不住 Bash。
- 实验包**无稳定性承诺**，schema 可能随时变 → 我们只依赖其公开服务与类型，不 fork 源码。
- **禁止在 `llm/stream` 改道**：`packages/llm/llm/src/index.ts:1036` 有硬守卫，改 provider/model 会抛 `INVALID_PREPARED_CALL`。唯一合法改道层是 **`agent/request` 瀑布**。

---

## 3. 总体架构

```
                        cluster.yml  (+ cordis.patch.yml 分层覆盖)
                                    │
                     ┌──────────────▼───────────────┐
                     │   cluster/config  (Service)  │  schemastery 校验 · 分层覆盖 · 热重载
                     │        ctx.clusterConfig     │
                     └──────────────┬───────────────┘
        ┌───────────────┬───────────┼───────────────┬─────────────────┐
        ▼               ▼           ▼               ▼                 ▼
 cluster/router   cluster/orchestrator  cluster/dialogue  cluster/tools  cluster/telemetry
        │               │                  │               │                 │
 agent/created    updateTask CAS     sendMessage 封装   agent.ctx scoped  SessionEventMap
 +installModel-   → 自动解锁        广播/圆桌/辩论    工具注册          cluster/* 事件
  Selection       轮次/重试/评审                                        成本记账
        │               │                  │                             │
        ▼               ▼                  ▼                             ▼
   ctx.llm          ctx.agentTeams    ctx.agentTeams              session log（唯一真相源）
 (llm-pi-ai 多供应商)  (roster/board/mail)
                                    │
                          Lead ──spawn──▶ Teammate Sessions

 cluster/bundle ──▶ profile "cluster"：dsh --profile cluster "任务描述"
```

---

## 4. 关键技术实现（已核实可行）

### 4.1 模型资产：一张 cordis 片段搞定多供应商

不写 adapter，直接用上游 `dsh-llm-pi-ai` 声明 route：

```yaml
- id: llm-pi-ai
  name: '@deepseek-ai/dsh-llm-pi-ai'
  config:
    providers:
      coder-gateway:
        displayName: OpenAI 兼容中转
        apiKeyEnv: CODER_API_KEY          # 只放环境变量名
        api: openai-completions
        baseURL: https://gateway.example/v1
        models: [{ id: qwen3-coder, contextWindow: 131072, maxTokens: 8192 }]
      critic-anthropic:
        displayName: Anthropic 兼容
        apiKeyEnv: CRITIC_API_KEY
        api: anthropic-messages
        baseURL: https://gw.example/anthropic
        models: [{ id: claude-opus-4, contextWindow: 200000, maxTokens: 16384 }]
```
DeepSeek 官方走既有 `llm-deepseek`（provider `deepseek-official`）。

### 4.2 per-teammate 模型绑定：不 fork 实验包的实现路径 ⭐

`spawnTeammate` 不透传 `agentOptions`，但有另一条干净的路：

```ts
// cluster/router 核心逻辑（伪码）
ctx.inject(['agentTeams', 'clusterConfig'], ...)
ctx.on('agent/created', ({ agent }) => {            // ① 串行且早于首次 prompt 装配（已核实）
  const m = ctx.agentTeams.tryMembership(agent)     // ② 拿到该 member 的 name / role
  if (!m) return                                    //    非 Team 成员不管
  const route = ctx.clusterConfig.routeFor(m.name)  // ③ 查 yml：provider + model (+ fallback 链)
  const dispose = installModelSelection(agent.ctx, { current: route, assembled: undefined })
  // ④ 用 agent.ctx 承载，随 scope 自动 dispose，HMR/teardown 安全
  agent.ctx.effect(() => dispose)
})
```

**为什么这条路成立（源码证据）**

- `agent/created` 是 **serial 且被 await 的**：`packages/core/agent-loop/tests/scope-lifecycle.spec.ts:100-107` 证明监听器执行完之前，排队的首轮工作不会开始（断言 `order` 为空）；`agent-loop` 文档也写明 "awaits serial `agent/created` initialization before starting queued work"。
- `installModelSelection` 的三个监听挂在 **`agent.ctx`**（scoped），在 `system-prompt/assemble` 里快照 `{provider,model}`、在 `agent/request` 里替换返回的 `LlmCallConfig`。它在 `llm.prepareCall` **之前**，绕开 `INVALID_PREPARED_CALL` 守卫。
- 换成 Flavors：`CreateAgentOptions.setup` 是"无竞态槽位"——但 teammate 由 `agent-team` 内部创建，我们拿不到 `setup`，故退而用 `agent/created`，风险已被上述测试覆盖。

**若后续 ProveELS 发现竞态**，兜底方案（代价递增）：
① 在 `cluster/router` 里于 `agent/pre-step` 首发也插一次 route；
② fork `packages/experimental/agent-team`，给 `SpawnTeammateRequest` 加 `agentOptions` 字段（约 20 行，须同步更新类型文档与 snapshot）。

### 4.3 配置模型 `cluster.yml`

不变的一期三件套（资产 / 角色 / 策略），但**增加**已核实的字段：

```yaml
version: 1
clusters:
  default:
    lead:  { model: planner, fallback: [cheap], max_rounds: 12 }
    topology: mesh            # star | mesh | pipeline | debate
    limits:                   # 对齐上游硬约束
      maxMembers: 8
      maxTasks: 256
    members:
      - name: coder-backend
        route: { provider: coder-gateway, model: qwen3-coder, temperature: 0.0 }
        fallback: [{ provider: deepseek-official, model: deepseek-chat }]
        context: fresh        # fresh | fork
        write_scopes: ["src/server/"]
        token_budget: 400000
    orchestration:
      briefing: structured
      dependency_auto_unlock: true
      review: { enabled: true, reviewer: reviewer, max_retries: 2 }
      consensus: { protocol: majority, max_rounds: 3 }
    budget:
      total_tokens: 2000000
      total_rounds: 60
      wall_clock_ms: 1800000
      on_exceed: stop_and_summarize
```

`agentOptions` 的合法字段只有 `provider / model / reasoningEffort / maxTokens`（`AgentOptions` 定义），
**`temperature` 不在其中** → 只能由 adapter 侧 route 定义或我们自己扩展，M1 先只用这四个字段，**采样参数退化为 route 级配置**（记入 Known Limitations）。

---

## 5. 包拆分（最终，`packages/cluster/*`）

| 包 | ctx key | 职责 | 主要落点 |
|---|---|---|---|
| `cluster/config` | `ctx.clusterConfig` | `cluster.yml` 解析 + schemastery 校验 + 分层覆盖 + 热重载 | Service |
| `cluster/router` ⭐ | — | **按角色把请求改道到指定模型**，含降级链 | `agent/created` + `installModelSelection` |
| `cluster/orchestrator` | `ctx.clusterOrchestrator` | briefing、依赖自动解锁、轮次、重试/评审、压缩/预算 | `updateTask` CAS、`agent/turn-stopping`、`agent.inject()` |
| `cluster/dialogue` | `ctx.clusterDialogue` | 广播 / 圆桌 / 辩论 / 投票 | `ctx.agentTeams.sendMessage` 之上 |
| `cluster/tools` | （注册到 `ctx.tools`） | 集群增强工具（带 briefing 的 delegate、ask、review） | `agent.ctx` scoped 注册 |
| `cluster/telemetry` | `ctx.clusterTelemetry` | token/成本/轮次记账 | 扩展 `SessionEventMap`（`cluster/*`） |
| `cluster/bundle` | — | profile `cluster` + patch：吃进上游 `agent-team-profile` 再叠我们的层 | `dsh.bundle` |
| `cluster/web` | — | **扩展** `client-ui-agent-team`：加配置/成本/消息图面板 | Client 插件 |

命名与门禁全部遵循 `AGENTS.md`：`@deepseek-ai/dsh-*`、`private: true`、ESM、`cordis` 进 peer+dev、
注册进 `tsconfig.host.json` / `tsconfig.client.json`、README 必含 Model Experience + Known Limitations。

---

## 6. OAT 机制 → dsh 落地对照（最终版）

| OAT 机制 | 上游是否已有 | 我们要做什么 |
|---|---|---|
| 上下文隔离 worker | ✅ `spawnTeammate`，`fresh`/`fork` | 无 |
| 文件任务板 → 持久化看板 | ✅ 更强（Session 日志 + CAS + DAG 无环校验） | 无 |
| 文件邮箱 | ✅ Steer 投递（步边界/起新轮/冷恢复）+ 去重键 + no-retry 规则 | 无 |
| Inbox 自动投递 | ✅ 内建 | 无 |
| Worker 预等待 `wait_agent` | ✅ 返回 `noProgress` 提示先唤醒 | orchestrator 据此编排唤醒顺序 |
| 上下文压缩 | ✅ `packages/compaction/compaction` | per-member 预算策略 |
| **per-角色模型**（你的核心诉求） | ❌ | ⭐ `cluster/router` |
| **结构化 briefing** | ❌ | `cluster/orchestrator` |
| **依赖自动解锁通知** | ❌ | `cluster/orchestrator` |
| **群对话协议** | ❌ | `cluster/dialogue` |
| **预算硬闸 / 成本账** | ❌ | `cluster/telemetry` |
| **跨供应商交叉评审** | ❌ | router + orchestrator 配合 |

---

## 7. 里程碑（修订）

| 阶段 | 产出 | 验收 |
|---|---|---|
| ✅ **M0 基线** | 环境、源码、安装、构建、`--dump-config` | 已完成 |
| ✅ **M0.5 能力盘点** | 能力矩阵 + 差距表 + 验证 `agent/created` 时序 | 已完成（本文档 §2） |
| **M1 最小闭环** | `cluster/config` + `cluster/router` + `cluster/bundle` | **改一处 yml 的模型，实际调用随之改变**（核心验收） |
| **M2 编排** | `cluster/orchestrator`：briefing / 自动解锁 / 轮次 / 预算 / 评审 | 带依赖的 3 任务 DAG 无需人工干预跑完 |
| **M3 对话 + 遥测** | `cluster/dialogue` + `cluster/telemetry` | 两种 topology 各跑通一场景；对话可回放；成本可见 |
| **M4 可观测 + 文档** | CLI 命令族、扩展 Web 面板、snapshot 测试、README | `doc-sync` + `typecheck` + `lint` 通过 |

**立即开工项**：M1 的三个包 + `cluster.yml` 骨架。

---

## 8. 风险台账（动态更新）

| 风险 | 等级 | 现状 | 对策 |
|---|---|---|---|
| `agent/created` 装 model selection 的竞态 | 中 | 已由 `scope-lifecycle.spec.ts` 证实串行+await | 兜底：`agent/pre-step` 补一次；最坏 fork 加 `agentOptions` |
| `temperature` 不在 `AgentOptions` | 低 | 已确认 | 采样参数下沉到 adapter route 级；记入 Known Limitations |
| 上游实验包 breaking change | 中 | 已锁 commit | 只依赖公开服务；每包 README 记录依赖版本；升级走单独 PR |
| `writeScopes` 无强制力 | 中 | 上游明示 advisory | `tools/pre-execute` 加 guard 做拦截+告警 |
| Windows 构建链 | 低 | 构建已通过 | 保持全链路在 E:，不改 WSL |
| E: 剩余 35 GB | 中 | install 已耗 1.5 GB | 如需腾挪，把 pnpm store 指到 D:/F: |
| Node 未进 PATH | 低 | 每 shell 手动注入 | 建议 `setx` 固化（待你确认） |

---

## 9. 待你确认

1. 是否把 `C:\Program Files\nodejs` 与 `%APPDATA%\npm` 用 `setx` 固化进用户 PATH（免得每次手动注入）？
2. M1 先做**纯 Host 侧**（config + router + bundle，用 headless profile 验证走通的模型后的日志），还是直接带上一个最小可用的 `cluster.yml` 示例再跑？我倾向前者，1 天出结果。
3. `cluster/web` 是否等 M3 之后再做（先看清真跑起来缺什么），还是现在就把 `client-ui-agent-team` 的复用方案定下来？

---

## 10. M1 实测进展与阻塞（2026-09-19）

### 10.1 已交付代码（均已构建通过 + 单测通过）

| 位置 | 内容 |
|---|---|
| `packages/cluster/config` | `ctx.clusterConfig` 服务：`cluster.yml` 严格校验（未知键/未知别名/错误类型一次性全部报出）、别名与内联路由、`routeFor` / `fallbacksFor` / `member` / `defaultClusterName` |
| `packages/cluster/router` | `cluster-router` 插件：`agent/created` + `ctx.agentTeams.tryMembership` + `installModelSelection(agent.ctx, ref)`，随 `agent/disposed` 与插件 effect 释放 |
| `packages/cluster/bundle` | profile 层：禁用 4 个 legacy subagent 工具行，插入 agent-team / tool-agent-team / cluster-config / cluster-router |
| `packages/cluster/cluster.example.yml` | 可跑通并被测试当作 fixture 的示例文档 |
| `packages/boot/app-boot/src/profile.ts` | 把 `@deepseek-ai/dsh-cluster-bundle` 加入 `OPTIONAL_BUNDLES`（Web 插件页可见） |
| 门禁改动 | `tsconfig.base.json` 加 3 条 paths、`tsconfig.host.json` 加 3 条 references |

测试：`packages/cluster` 共 **10 个测试全部通过**；`pnpm run build` EXIT=0。

### 10.2 搭建的端到端验证台（可复用）

- `.dev/mock-openai.mjs`：脚本化 OpenAI 兼容网关，按 `model` 字段分流；Lead 依次发出
  `spawn_teammate → send_message → wait_agent → 结论文本`，队友返回纯文本。
  每次请求把 `model` 写入 `.dev/mock-requests.log`，**用于证明"哪个角色用了哪个模型"**。
- `.dev/verify-upstream.patch.yml`（上游 Team 对照）+ `insert-config.patch.yml` / `insert-router.patch.yml`（增量挂载我们两个插件）。
- 运行方式：`pnpm dsh --profile headless --patch A --patch B --patch C "<任务>"`（`--patch` 可重复）。

### 10.3 验证台的两个陷阱（都踩过，务必避免）

1. **mock 网关必须每轮重启**。脚本用请求计数决定"第几轮回什么"，进程不重启就会跨运行累积，
   导致后续运行第一步就直接吐结论文本、看起来"通过"——**前几轮我据此得出的"队友已跑通"结论全部作废**。
   每轮验证前必须重启 mock（并重新写日志）。
2. **mock 必须发 `finish_reason`**，否则每轮报 `TRANSPORT: Stream ended without finish_reason`，
   会伪装成"队友没被创建"。
3. 顺带记录一个组合陷阱：`session-log-deepseek` 依赖 `deepseekLlmApiExtensions`。
   若禁用后者，前者会 `pending` 且启动时打印 `1 entry did not activate`。
   验证时保留 `deepseek-llm-api-extensions`，只禁用 `llm-deepseek` 与 `plugin-package-inventory-deepseek`。

### 10.4 已实施的上游改动（方案 A 的落地）

调研发现关键事实：`SubagentContinuationManager.startContinuable()` **已经完整支持**
`request.agentOptions`（`packages/subagent/subagent/src/continuation.ts:115`
`resolveChildAgentOptions(parent, request.agentOptions, childDepth)`，并一路传到子 Agent 创建）。
缺的只是 `agent-team` 的 `roster.spawn()` 没把它透传出去。因此改动面比预估更小：

| 文件 | 改动 |
|---|---|
| `packages/experimental/agent-team/src/types.ts` | `SpawnTeammateRequest.agentOptions?: TeamSpawnRoute`；新增 `TeamSpawnRoute`（provider / model / reasoningEffort / maxTokens） |
| `packages/experimental/agent-team/src/roster.ts` | 把 `request.agentOptions` 透传给 `ctx.subagents.startContinuable(...)`（Host 侧做一次品牌收敛） |
| `packages/experimental/tool-agent-team/src/index.ts` | `spawn_teammate` 执行时按 member 名从 `ctx.get('clusterConfig')` 解析路由并作为 `agentOptions` 传入（**结构性查找，不引入包依赖**；服务缺失时静默回退到继承 Lead 路由） |
| `packages/boot/app-boot/src/profile.ts` | 把 `@deepseek-ai/dsh-cluster-bundle` 加入 `OPTIONAL_BUNDLES` |

设计要点：**路由在"创建时"就确定**，不依赖事件时序，也不依赖模型自觉传参——`cluster.yml` 里
`members[].route` 是唯一真源。Lead 自身仍由 `cluster-router` 在 `agent/created` 时绑定
（已实测：Lead 的创建事件可以到达插件监听器）。

**踩坑记录（重要）**：`TeamSpawnRoute` 刻意**不 import `@deepseek-ai/dsh-agent`**。
`agent-team/src/types.ts` 同时喂给浏览器侧（`./client` 导出），一旦把 Host 的 Agent 包拖进 Client 程序，
`ctx.sessions` 就会被解析成 Host 的 `SessionStore`，导致 `client-ui-agent-team`
报 `Property 'binding'/'refreshSubagents'/'retainInfo' does not exist` 三连错。
改为结构等价的本地类型 + Host 侧一次 `as AgentOptions` 收敛后，全量构建通过。

### 10.5 仍然存在的阻塞：`spawn_teammate` 稳定失败

**现象**：只要 `spawn_teammate` 真正被调用，整个 turn 就以
`dsh: UNKNOWN: Cannot read properties of undefined (reading 'prepare')` 结束（EXIT=1），
队友从未被创建（`agent/created` 不会出现第二行）。

**已排除**：
- **不是我们的插件引起的**——用完全不含 cluster 插件的「仅上游」补丁 + 全新 mock 复现，现象一致（连跑两次稳定复现）。
- **不是 mock 的锅**——已修复 `finish_reason` 与跨轮状态问题后仍然复现。
- **不是凭据/禁用行的锅**——`deepseek-llm-api-extensions`、`session-log-deepseek` 均已恢复启用。

**已定位的边界**：工具体（`execute`）从未进入（临时诊断未打印），说明失败发生在
**工具调度阶段**而非工具内部。agent-loop 的调度入口是
`ctx.tools[TOOL_RUNTIME_SCHEDULER].prepare(call.exec)`
（`packages/core/agent-loop/src/tool-calls.ts:170`），
报错形态与该表达式取到 `undefined` 完全吻合；下一步应在该行插入诊断，确认
`ctx.tools` 上的内部 symbol 是否在 scoped 视图中丢失。

**复现命令**（约 1 分钟）：

```powershell
# 1. 启动 mock（每次验证前必须重启！）
node .dev/mock-openai.mjs
# 2. 运行
$env:DSH_HOME='.dsh-home'; $env:MOCK_API_KEY='sk-mock'; $env:DEEPSEEK_API_KEY='sk-mock'
pnpm dsh --profile headless --patch ../../.dev/verify-upstream.patch.yml "<任务>"
```

### 10.6 M1 验收结论

- ✅ 配置层、模型路由层、profile 层代码完成，全量构建通过，10 个单测通过
- ✅ 上游缺口已按方案 A 打开（`agentOptions` 可从 `cluster.yml` 确定性地下发到队友）
- ❌ **端到端"改 yml 的模型 → 实际调用随之改变"未验收**，被上游 `spawn_teammate` 的
  `.prepare` 失败挡住；修复点在 `tool-calls.ts:170`，与我们的改动无关

---

### 10.7 阻塞已解开（根因：symbol 身份分裂）

**根因**：`TOOL_RUNTIME_SCHEDULER` 原本用 `Symbol('…')` 定义，而 `@deepseek-ai/dsh-tools`
在同一个进程里被**加载了两次**——插件加载器按包名走构建产物 `lib/`，CLI 自身经 tsx 走源码 `src`。
两个模块实例各自求值一次 `Symbol()`，于是 `ToolRuntime` 实例上存的 symbol 与 agent-loop
查表用的 symbol **身份不同**，`ctx.tools[TOOL_RUNTIME_SCHEDULER]` 恒为 `undefined`，
报错就落在 `.prepare` 上。

**证据**：在 `runGroup` 插桩后打印出 `ctx.tools` 是 `ToolRuntime` 实例，且其自有 symbol 里
存在同名 `Symbol(@deepseek-ai/dsh-tools.scheduler)` —— **同描述、不同身份**，正是双实例特征。

**修复**（1 处，1 行）：

```ts
// packages/core/tools/src/index.ts
export const TOOL_RUNTIME_SCHEDULER: unique symbol = Symbol.for('@deepseek-ai/dsh-tools.scheduler')
```

改为全局注册表 symbol，使这条内部缝对"模块被求值多次"免疫。
`unique symbol` 类型仍然成立（TS 允许 `Symbol.for()` 用于 `unique symbol` 声明）。

**端到端验收（已通过）**：

```
cluster 补丁挂载时：  lead-model, lead-model, tester-model, lead-model, lead-model   EXIT=0
仅上游补丁（对照）：  lead-model, lead-model, lead-model, ...                        EXIT=0
```

`cluster.yml` 中 `members[tester].route = mock-tester/tester-model`，
队友的请求确实带上 `tester-model`；移掉 cluster 插件后队友又退回继承 Lead 模型。
**"改 yml 的模型 → 实际调用随之改变"这条 M1 核心验收成立。**

---

## 11. 交付与仓库状态

### 11.1 已推送

仓库 **[HMUG12/Cluster-Cooperation](https://github.com/HMUG12/Cluster-Cooperation)**，分支 `main`，
基线为上游 `dsh-0.1.6-alpha.2` 发布提交。推送状态随时变化，以仓库的提交列表为准。

**仓库形态**：`Cluster-Cooperation` 本身就是 **deepseek-harness 的 fork**（保留了上游全部提交历史），
本地 `deepseek-harness/` 的 `origin` 指向该仓库，原上游已改名为 `upstream`。
因此后续同步上游只需 `git fetch upstream && git merge upstream/master`，
提交 PR 也直接可用。本轮方案文档以 `CLUSTER.md` 随代码入库。

提交内容：

| commit | 说明 |
|---|---|
| `feat(cluster): declarative Agent Cluster configuration and role-based model routing` | 3 个新包 + 上游 `agentOptions` 透传 + `OPTIONAL_BUNDLES`（28 个文件） |
| `chore: join the Cluster-Cooperation initial commit` | 与远端初始提交的历史合并（`-s ours`，内容以 fork 为准） |

推送前门禁：`pre-commit`（lint / whitespace / vendor guard / third-party notices）与
`pre-push`（`pnpm run typecheck`，59.8s）**全部通过**。

### 11.2 本地遗留（均已被 `.gitignore` 忽略，不影响仓库）

- `.dev/`：可复用的验证台（`mock-openai.mjs`、`dump-session.mjs`、三份 patch），
  以及本轮调试产生的若干 `*.log` 与失效脚本 `build.bat`/`build.ps1`（删除时审批弹窗超时，未清理）。
- `.dsh-home/`：本地 Harness home，内含验证用的 `cluster.yml` 与会话目录。
- 外层目录 `E:\新创意构思\Cluster-Cooperation`（`docs/` + 嵌套的 fork 克隆）现在只是**工作区**，
  它自身不是被推送的仓库。

### 11.3 环境提示

IDE 进程环境在启动时冻结，新开 shell 看不到 `node`。`PATH` 已用 .NET 直写注册表还原并追加
`%APPDATA%\npm`，**重启 IDE 后生效**；在那之前所有命令都需显式注入
`$env:PATH="C:\Program Files\nodejs;$env:APPDATA\npm;$env:PATH"`。

---

## 11.4 M2 第一刀：依赖自动解锁（已完成）

上游原文承认这个缺口：**"Task readiness never starts an owner."** —— 任务板会算出 `ready`，
但没有任何事件告诉 owner 可以开工了，只能等它自己重新 `team_task_list`。

新增包 `packages/cluster/orchestrator`（`@deepseek-ai/dsh-cluster-orchestrator`）：

| 文件 | 职责 |
|---|---|
| `src/ready.ts` | **纯函数** `readyNotices(tasks, completedTaskId)`：一次完成事件应该唤醒哪些 owner；以及 `readyMessage()` 的消息文本 |
| `src/index.ts` | 插件：订阅 `session/event` 过滤出 durable `team/task` 提交 → 解析 Lead 与会话 → 串行投递 `[TASK READY]` 给每个被解锁的 owner |

设计要点：

- **不轮询**。唤醒搭在"改变了任务板的同一个事实"上——`team/task` 的 durability commit。
- **判定逻辑全在纯函数里**，因此规则本身可以脱离活的 Team 被测；投递层只是薄适配。
- **幂等**：只对"刚刚完成、且在被解锁任务的 `blockedBy` 里"的完成发通知，
  重放同一条事件、或重新读一遍没动过的板子，都不会产生第二次唤醒。
- **串行投递**：所有投递排在一条 promise 链上，慢消息不会让后一次完成超越前一次。
- 投递失败只记 warning，绝不让"完成任务的 turn"失败。

`config.dependencyAutoUnlock` 默认 `true`；bundle 的 patch 已把它一并挂载
（insert 顺序：agent-team → tool-agent-team → cluster-config → cluster-router → cluster-orchestrator）。

---

## 11.5 M2 第二刀：结构化 briefing（已完成）

OAT 的核心机制之一是"结构化任务派发"——`spawn_agent` 不是传一句话，而是传
`mission / deliverables / definition_of_done / quality_bar`。dsh 的 `spawn_teammate`
只有 `name / description / prompt`，这些约束**每次派发都要靠 Lead 重新打一遍**，且很容易漏。

这一刀把它变成**配置即契约**：写进 `cluster.yml` 一次，每次派生都自动带上。

`cluster.yml` 新增四个可选字段（member 级）：

```yaml
- name: coder-backend
  model: coder
  mission: Turn the agreed interface into working server-side code.
  deliverables:
    - Implementation under src/server/
    - Unit tests covering the new endpoints
  definitionOfDone:
    - The design document's interface is implemented without signature drift
  qualityBar:
    - No new lint or type errors
```

落点：

| 文件 | 改动 |
|---|---|
| `cluster/config/src/briefing.ts` | **新增纯函数** `renderBriefing({clusterName, member})` |
| `cluster/config/src/document.ts` | 成员的 allowed keys 增加 4 项并读取 |
| `cluster/config/src/index.ts` | 新增 `briefingFor(clusterName, memberName)` |
| `experimental/tool-agent-team/src/index.ts` | `configuredRoute` 升级为 `configuredMember`，把 briefing 插在**身份提醒之后、模型任务之前** |

**关键语义决定**（由测试逼出来的）：briefing 的判据是**问责**
（`mission` / `deliverables` / `definitionOfDone` / `qualityBar` 至少有一项），
而被写作用域和 token 预算只是"顺带"——只声明后两者的成员**不产生 briefing 块**，
保持"队友拿到的就是 Lead 原样那句任务"的既有行为。这条规则写进了 `renderBriefing` 的契约注释。

渲染形状：

```
<cluster-briefing member="coder-backend" cluster="default">
Mission: …
Deliverables:
- …
Definition of done:
- …
Quality bar:
- …
Advisory write scopes: src/server/
Token budget: 400000
</cluster-briefing>
```

**验证**：`packages/cluster` **21 个测试通过**（其中 4 个覆盖 briefing：完整渲染、
只声明路由或 Lead 时静默、省略未声明小节、非字符串数组要报错），
`tool-agent-team` 的 **22 个测试全部通过**（身份前缀契约未被破坏）。

---

## 11.6 M2 第三刀：评审回路（已完成）

配置里早就声明了 `orchestration.review`，但一直是空头支票——**"完成的活没人把关"**：
任务一旦 completed 就被当成交付，Lead 只能自己再读一遍 diff。

现在它变成真行为，仍然是事件驱动 + 纯函数判定。

`cluster.yml`：

```yaml
clusters:
  default:
    orchestration:
      review:
        enabled: true
        reviewer: reviewer   # 必须是本 cluster 已声明的成员
        maxRetries: 2        # 被驳回几次后升级给 Lead
```

两条规则（都挂在同一个 durable `team/task` 提交上）：

1. **完成即送审**：非评审任务一旦 `completed`，自动建一个评审任务
   （`Review: <原标题>`、`cluster-review-of: <taskId>` 标记、继承被审任务的写作用域），
   由 Lead 身份 `reassign` 给配置的 reviewer，并投递 `[REVIEW]` 通知。
2. **驳回即计数**：当某任务重新变成非 completed 状态、且它已有评审记录时，
   驳回次数 = **为它开过的评审任务数**（板上即全部状态，不需要隐藏计数器）。
   未超预算投 `[RETRY k/max]` 给原 owner；超预算投 `[ESCALATE]` 给 Lead，不再自动重试。

四条**刻意的拒绝条件**（都在 `reviewRequest()` 里）：

- 不评审"评审任务"本身（否则无限递归）；
- 不评审 reviewer 自己的产出（自审没有意义）；
- 同一完成事件已有评审记录时不再重复开（这是事件重放安全的保证）；
- 未配置 reviewer / `enabled: false` 时整条规则不生效。

**幂等键用的是 `taskId::revision`**——revision 每次变更自增，所以事件重放会被抑制，
而任何真实变更都能通过；集合有上限（512）避免无界增长。

落地：`cluster/config/src/document.ts`（`orchestration.review` 解析 + 交叉校验 reviewer
必须是本 cluster 声明过的成员）、`cluster/config/src/index.ts`（`reviewFor()`）、
`cluster/orchestrator/src/review.ts`（新增纯函数 `reviewRequest` / `retryNotice` / 三个渲染器）、
`cluster/orchestrator/src/index.ts`（接线）。

**验证**：`packages/cluster` **35 个测试通过**（新增 9 个评审回路 + 5 个评审配置）。

---

## 11.7 M2 第四刀：用量记账与预算执行（已完成）

`cluster.yml` 从第一个版本起就有 member 级 `tokenBudget`，**至今没有任何消费者**——和评审回路之前的状态一模一样。
这一刀把它变成真行为，仍然沿用"事件驱动 + 纯函数判定"的同一套骨架。

落地：

| 文件 | 改动 |
|---|---|
| `cluster/config/src/index.ts` | 新增 `budgetFor(clusterName, memberName)`（Lead 不声明预算） |
| `cluster/orchestrator/src/spend.ts` | **新增纯模块**：`addUsage` / `billableTokens` / `overBudget` / `budgetNotices` |
| `cluster/orchestrator/src/index.ts` | 订阅 `assistant/message` 折算花费，越线时通知成员与 Lead，新增 `budgetWatch` 配置项 |

四个关键设计决定：

1. **计费口径是"四种 token 之和"**。`TokenUsage` 的计数是**互斥**的：`inputTokens` 只算未命中缓存的部分，
   缓存读写单独上报。所以 `billable = input + cacheRead + cacheWrite + output`，
   且**刻意不用** adapter 可选的 `totalTokens`（它可能缺省）。
2. **`overBudget` 用严格大于**。正好花完预算的成员算"在预算内"——这才是写下那个数字的人对"预算"的理解。
3. **恰好一次靠 seq 单调性**，不靠去重集合。任务板那条规则用 `taskId::revision` 作键，
   但花费不行：只折算 `seq` 大于该会话"上次已折算 seq"的事件，天然幂等，且不依赖会因上限被淘汰的键集合。
4. **成员未上榜时不缓存**。teammate 的第一次用量可能在其 roster 行尚未落定时到达，
   把这次 miss 缓存成"无预算"会永久性废掉该成员的预算，所以只有解析成功才缓存。

投递策略：**成员收到收尾通知（可执行的那条），Lead 收到超额代价**，两条都是 best-effort、
各自独立捕获异常——Lead 那条如果被拒（向自己发消息），不能连带丢掉成员那条。
通知按**每会话一次**发送。

**验证**：`packages/cluster` **47 个测试通过**（新增 10 个用量 + 2 个预算），
`tsc -b tsconfig.host.json` EXIT=0，oxlint 0 warning 0 error，
四项文档门禁（配对 1011 / 模型体验 295 / 链接 2011 / 折行 2035）与 `doc-standard.spec` 22 项全绿。

**已知边界**：只通知不强停（没有取消成员这一轮），且每会话只通知一次——
两条都写进了 README 的 Known Limitations。

## 11.8 M2 第五刀：评审裁决成为解锁闸门（已完成）

之前的行为有个语义漏洞：任务一到 `completed` 就唤醒下游，**评审形同虚设**——
下游在裁决出来之前就开工了。这一刀把"释放下游"的事实从"交付"改成"裁决"。

实现只有两处：`ready.ts` 新增纯函数 `releaseDecision(tasks, completedTaskId, reviewOwed)`，
`index.ts` 在唤醒前先判断"这次完成是否仍欠评审"。规则三条：

- 完成事件**仍欠评审** → 谁都不唤醒（下游等裁决）；
- **评审任务完成**（即批准）→ 反查 `reviewedTaskOf`，唤醒**被审任务**的下游；
- 未启用评审（`reviewLoop: false`）→ 维持原样，立刻唤醒。

顺带把 `openReview` 的签名从"自己算 request"改成"接收已算好的 request"：这样
"释放判定"与"实际开出的评审任务"读的是同一块板子，不会各自算出不同结论。

**验证**：`packages/cluster` **51 个测试通过**（`ready.spec.ts` 新增 4 个，含一条
"交付不释放 / 批准才释放"的组合断言），`tsc -b tsconfig.host.json` EXIT=0，
四项文档门禁（配对 1011 / 模型体验 295 / 链接 2011 / 折行 2035）全绿。

**已知边界**：始终不批准的 reviewer 会让下游永久阻塞，只能靠人工或 Lead 打破——
已写进 README 的 Known Limitations。

## 11.9 M2 第六刀：声明期一致性校验（已完成）

`document.ts` 的模块注释写着它存在的理由是"让拼写错误在加载期失败而不是静默路由到错误的模型"，
但有三处静默歧义漏网：

1. **`tokenBudget` / `maxConcurrency` 未做正整数校验**。`tokenBudget: 0` 或负数会被接受，
   而 §11.7 的判定是 `billable > budget`——于是该成员**第一次调用就越线**。
   也就是说这一刀护住的正是刚做出来的功能。
2. **重复成员名**。路由、briefing、预算全部**按名解析**，重名时静默落到第一行，
   第二行声明的模型意图被无声丢弃。
3. **`enabled: false` 仍强制要求 reviewer**。关掉评审反而比打开评审更难写。

三处都在 `readClusterDocument` 内修掉，并新增 4 个测试覆盖：0/-1/1.5/`'many'` 四种非法预算、
重名成员、非正整数并发上限、以及"显式关闭评审可以不写 reviewer"。

**验证**：`packages/cluster` **55 个测试通过**（config 19 → 23），`tsc -b tsconfig.host.json` EXIT=0。

## 11.10 M3 前置：修正"通知 Lead"的通道，并定下 M3 的状态表达（已完成 + 设计结论）

### 缺陷（我上一版自己引入的）

`mailbox.ts:121` 有一条硬约束：**成员不能给自己发消息**（`TEAM_SELF_MESSAGE`）。
而 orchestrator 只持有 Lead 这一个凭据，于是它有两条通知的目标写了 `'lead'`——
发送者正是 Lead 自己，**必然抛错、被 `tryDeliver` 吞掉，Lead 永远收不到**：

1. 评审重试耗尽时的 `[ESCALATE]`（原设计"交给 Lead 决定"）；
2. 预算越线时那半条"告诉 Lead 超额代价"。

修法不是绕过约束，而是**顺着唯一的合法边**：只有队友能对 Lead 说话。
两条通知都改为**发给能转达的那个成员**，并在文本里明确要求它自己 `send_message target "lead"`；
无 owner 的升级没有任何合法收件人，改为 `logger.warn` 交给运维，
不再尝试注定被拒的投递。已知边界写进了 README。

### M3 设计结论（先想清楚再写代码）

`TeamService` 的公开面只有 `sendMessage / createTask / getTask / listTasks / updateTask /
interrupt / waitForChange / listMembers`。由此定下三条 M3 的设计前提：

1. **任何面向 Lead 的通知，只能"由队友转达"或"落在任务板上"**，没有第三种通道。
   这是所有对话协议的共用约束。
2. **投票不需要新状态，但需要新语义**。任务板能表达的只有"行 + 状态"。上一轮试多人评审时
   撞到的正是"旧评审行同时充当计数凭证与幂等守卫"，于是"本轮已满"与"已审过"无法区分。
   可行解法是**把轮次编号显式化**（如 `Review #2:` 前缀），让轮次从"行数的推论"变成"板上事实"。
3. **辩论/投票的裁决必须落成一次 `createTask` / `updateTask`**，否则 Lead 看不到结论、
   下游也无法据此解锁。

三条合起来意味着 M3 第一期应当选**广播**：它只需"Lead → 多个成员"这条**已存在**的合法边，
不需要新状态，而且是圆桌/辩论/投票三种协议的公共基础设施。

**验证**：`packages/cluster` **55 个测试通过**（`retryMessage` 与 `budgetNotice` 的断言
改为校验"由成员转达"的措辞），`tsc -b tsconfig.host.json` EXIT=0。

## 11.11 诚实性修正：`topology` 与 `maxConcurrency` 是空转的（已完成）

复查 config 包时发现一处比"未消费"更糟的状态：`topology` 与 `maxConcurrency` 被解析、
被枚举/范围校验、存进 `ClusterDocument`……然后**连访问器都没有**
（`grep topology packages/cluster/config/src/index.ts` 零命中）。

而 `README` 的示例里赫然写着 `topology: mesh`——**这会让人以为拓扑已经生效**。

它们真实的效力是零：谁能和谁说话完全由 Team 邮箱的邻接规则决定，
同时运行多少个成员没有任何东西限制。这一刀不发明消费者，只**把假象改成事实**：
在 config README 的 Known Limitations 里写明两者不改变任何运行时行为，
并说明"声明它们只是为了让文档能写下意图，有消费者之前会一直空转"。
双语同步，行数仍逐行对齐。

**验证**：四项文档门禁全绿（配对 / 模型体验 / 链接 / 折行）。

## 11.12 门禁债：fork 在全量门禁下并非全绿（一项已修，一项待做）

跑此前**从未跑过**的目录类门禁，发现 fork 并不全绿，责任都在我们新增的 cluster 包：

| 门禁 | 结果 | 原因 / 处置 |
|---|---|---|
| `verify-tool-catalog` | ✅ | 未被影响（我们没有 `tool-*` 包） |
| `verify-config-catalog` | ❌ → ✅ | `docs/config-catalog.md` 过期（我们改过配置 schema）；**已重新生成并提交** |
| `verify-cordis-catalog` | ❌ 仍红 | `service ctx.clusterConfig … has no SERVICE_PAGE entry`，另有 3 个类型未分类 |

`gen-cordis-catalog.ts` 是 fail-closed 的双向校验：每个 `ctx.<key>` 服务必须映射到**恰好一个**
subsystems 页（`SERVICE_PAGE`），生成器再把 API 区段注入该页；每个服务方法引用的类型必须归类到
`linkedTypePages`（带文档页）、`foundationTypeNames` 或 `typeLinkExemptions`。

我先加了 3 条 `typeLinkExemptions` 把类型违规消掉，随后**主动撤掉**：`SERVICE_PAGE` 要求的
那份子系统页无论如何都得建，而页面一旦存在，这 3 个类型就该走 `LINK_MAP` 指向它
（`docs/subsystems/agent-team.md` 里那些 `ts type-equiv` 块正是这类页面的写法），例外只是兜底。
落下一个马上要被替换的捷径是浪费。

**已修复（本次）**：新建 `docs/subsystems/cluster.md` 与中文对照页（各 108 行，已进
`docs/subsystems/README.md` 导航表），在 `gen-cordis-catalog.ts` 里登记 `LINK_MAP`
（`ClusterSpec` / `MemberSpec` / `ReviewSpec` → `cluster.md`）与 `SERVICE_PAGE`
（`clusterConfig: 'cluster.md'`），并在 `gen-doc-graphs.ts` 里补上它**自己的** fail-closed
服务角色分类 `SERVICE_ROLES`（`mode: 'core'`）。生成器随后重写了四份产物：
`docs/config-catalog.{md,zh}`、`docs/capability-seams.{md,zh}`、
`docs/event-producer-consumer.{md,zh}`、`packages/extensions/tool-cordis/src/api-catalog.ts`。

**一条重要的协作机制（差点漏掉）**：生成器**只写英文侧**。英文生成源一变，中文对照就必须由
**同一次改动**补齐（`docs/i18n/README.md` 的规则），否则配对门禁变红——而它校验的是**结构性签名**
（标题层级、逐字代码块、表格行列数、列表种类与项数、链接目标），**不是行数**。
我那个 `pairlines` 助手只适用于手写对照，用在生成文档上会误报。
另一个坑：`config-catalog` 的重新生成在**上一轮提交**里就已产生，但当时没跑配对门禁，
债留到了这一轮才被发现——**改了生成物就要立刻跑配对门禁**。

**验收**：`verify-translation-pairing` 1012 对全一致；`md-links` / `md-wrap` /
`cordis-catalog` / `cordis-api` / `doc-graphs` / `config-catalog` / `type-equiv` **七项全 0**。

**八项门禁探完，全景如下**：

| 门禁 | 结果 |
|---|---|
| `verify-tool-catalog` / `verify-dependency-catalog` / `verify-export-jsdoc` | ✅ |
| `verify-config-catalog` | ✅（本次修复） |
| `verify-cordis-catalog` / `verify-cordis-api` / `verify-doc-graphs` | ❌ → ✅（三者同根因，本次一并修复） |

关键发现：`gen-cordis-api` 与 `gen-doc-graphs` **都复用** `projectCordisCatalog`，
因此三者报的是**同一批**三个类型违规（`cordis-catalog` 额外多一条 SERVICE_PAGE）。
这一刀的收益因此被放大：**一份 `cluster.md` 子系统页 + `LINK_MAP` + `SERVICE_PAGE`
可一次性让三个红门禁同时转绿**。

## 11.13 M3 第一期：`broadcast_message`（已完成并真机验证）

设计结论（§11.10）指向广播：它只需已有的"Lead → 多成员"合法边，不需要新的任务板状态，
而且它是圆桌/辩论/投票的共同基础设施。**落点我自己拍了：放进 `cluster-orchestrator`。**

为什么不新建 `tool-*` 包：两笔真实成本——一次完整 `pnpm install`（本会话实测 15 分 32 秒），
**加**改上游的目录收割器（`gen-tool-catalog.ts` 有 78 个显式 import、伪造附件存储/工作流引擎/
subagent provider，还要为"从不全局注册的工具"mint Agent 作用域）。两种落点的工具行为完全一致，
所以先出功能；"提升为 `tool-*` 包以进入工具目录"记为待办，并写进了 README 的 Known Limitations。

| 文件 | 改动 |
|---|---|
| `cluster/orchestrator/src/broadcast.ts` | **新增纯模块**：`broadcastTargets(requested, members)` |
| `cluster/orchestrator/src/index.ts` | 只在"调用者是 Lead 且组合挂载了工具运行时"时注册该工具 |
| `package.json` | 新增 `@deepseek-ai/dsh-tools` peer/dev 依赖 |

四个决定：

1. **不把 `tools` 写进 `inject`**，沿用本文件已有的"结构化可选服务"模式（`ctx.get('tools')`）。
   board 策略只需要 `agents` + `agentTeams`；让一个编排插件因为它附带的一个工具而提高加载门槛，
   会破坏那些刻意不挂载工具运行时的组合。
2. **省略 `targets` 即面向全队**；点名时逐个规范化（去空白、去重）并校验。
3. **拒绝必须带理由**（名字不存在 / 就是 Lead 自己 / 派生已失败），而不是静默丢弃——
   "以为已经通知了 5 个人"是不能接受的错误。写测试时我发现自己断言错了：默认广播时
   failed 成员**应当**出现在 `skipped` 里，实现是对的、测试是错的。
4. **串行投递**，因此返回顺序等于调用方点名的顺序，单点拒绝不会中断其余投递。

**真机验证（不只跑单测）**：用脚本化模拟网关跑 headless 集群，并从网关记录的请求体里确认——
Lead 的 26 个工具里 `broadcast_message` 排第一，而 teammate 的 29 个工具里**没有**它
（"只给 Lead 注册"的隔离性成立）。整条链路（config → router → orchestrator）依然正常：
teammate 被路由到 `tester-model` 并正常收尾。

**验证**：`packages/cluster` **65 个测试通过**（新增 10 个广播），`tsc -b tsconfig.host.json` EXIT=0，
门禁全绿（配对 1013 / 模型体验 / 链接 / 折行 / config-catalog / tool-catalog / export-jsdoc）。

**连带教训（第三次遇到）**：`package.json` 的 `description` 与源码行号都会进生成物
`docs/config-catalog.md`——生成物一变，中文侧与配对记录就要同步。这次是照着 §11.12 的教训做的。
另外记一条工具用法：**`pnpm install --lockfile-only` 是更新锁文件的正确姿势**——44 秒、不链接依赖、
不跑 postinstall，因此完全绕开那个 lefthook 死结（对比完整 install 的 15 分钟）。

## 11.14 M3 设计：三种对话协议能否落在现有基底上（含本轮补上的断链）

先设计、后动手。我把"圆桌 / 投票 / 辩论"逐一对到任务板能表达的东西上；
设计过程中撞出一处**真实断链**，本轮把它补掉了。

### 基底能表达什么

任务板的全部状态是 `TeamTaskSnapshot { id, revision, subject, description, status, ownerId, blockedBy, writeScopes }`，
而唯一写入方式是 CAS 变更，动作集为 claim / release / edit / set_dependencies / complete / reopen /
reassign / delete。**没有值槽**：除 `description`（自由文本）外，没有任何地方能存一个"立场"。

| 协议 | 需要什么 | 结论 |
|---|---|---|
| 圆桌（并行征询 + 汇总） | N 个"作答"任务 + 一个被它们阻塞的"汇总"任务 | 只需已有状态，**本轮补上了它缺的那一环** |
| 投票 | 每票需要一个值槽 | 需要新的编码约定（见下），延后 |
| 辩论 | "谁在第几轮发言"的顺序状态 | 最难，延后 |

### 设计撞到的两处硬约束

1. **`reassign` 拒绝被阻塞的任务**（`task-board.ts:188`，`TEAM_TASK_BLOCKED`），
   而 `createTask` 只产出无主行。于是"等所有答案到齐再汇总"**断链**：被阻塞的汇总任务在放行前
   没有主人，而放行时的就绪通知**要求有主人**，否则无人可唤。
   → **本轮修复**：任务用描述里的 `cluster-owner: <name>` 行声明主人，
   orchestrator 在**放行那一刻先指派、再唤醒**（`src/handoff.ts`）。
   这把 `blockedBy` 从"只能表达顺序"升级为"能表达扇出后收敛"——圆桌因此才可用。
2. **Lead 无法被唤醒**（§11.10）：邮箱拒绝自发自收。所以"汇总"的归属必须落在**某个队友**身上，
   或者由 Lead 主动轮询；而轮询与"自动协作"的初衷相悖。圆桌因此把汇总指派给队友（显式参数），
   这是当前唯一能闭环的选择。

### 投票的状态表达（结论，待实现）

三态（赞成/反对/弃权）无法用"一行 + 状态"表达。可行编码：**每张选票一行**，
立场由 orchestrator 在**创建时写进描述的机器可读标记**（如 `cluster-ballot: <motion> for`），
投票人 `complete` 表示"我投出这一票"、`delete` 表示弃权；一个被全部选票阻塞的
`Tally:` 任务在最后一张票落地时放行——**正好复用本轮做出来的 handoff 机制**。
已知代价：投票是**公开且可归属**的（任何成员都能 `team_task_list` 看到谁投了什么），
且成员理论上能 `edit` 自己票上的标记（tally 时必须校验标记并以 revision 佐证）。
辩论还需要"回合顺序"状态，那要求把轮次显式化（§11.12 的结论），排在投票之后。

### 本轮交付

`src/handoff.ts`（纯模块）+ orchestrator 放行时的指派步骤 + 中英文档：
`packages/cluster` **75 个测试通过**（新增 10 个指派），`tsc -b tsconfig.host.json` EXIT=0。
下一步是在这个闭环之上做**圆桌工具**（一次调用完成"扇出 → 汇总"两件事）。

## 11.15 M3 第二期：圆桌工具（已完成并真机验证）

在 §11.14 补齐的 handoff 闭环之上，`roundtable` 用一次调用完成"扇出 → 汇总"。

| 文件 | 改动 |
|---|---|
| `cluster/orchestrator/src/roundtable.ts` | **新增纯模块**：`roundtablePlan` 与一轮圆桌的全部文本 |
| `cluster/orchestrator/src/index.ts` | 与 Lead 的广播工具同址注册；一次调用建 N 个作答任务 + 1 个汇总任务 |

四个决定：

1. **汇总任务最后创建**。这样一次 `createTask` 就能同时带上它的阻塞列表**和**被收集的作答任务名；
   否则得先建再 `edit` / `set_dependencies` 改两次——而"任务被阻塞时还能否 `edit`"我没验证过，
   不去踩这个未验证的假设。
2. **收集者必须是队友，且校验排在第一位**。扇出之后汇总不了，等于烧掉每位参与者的整个回合，
   所以拒绝必须发生在**创建任何一个任务之前**。
3. **收集者不参与作答**（自动从名单中剔除），且"无人可问"时直接拒绝：没有参与者的汇总任务
   没有阻塞项、会立刻就绪，而就绪本身不触发任何完成事件——**永远没有人会被唤醒**。
4. **答案写在任务描述里**（末行 `answer: <内容>`），不额外发消息。这样合成者只读任务板，
   不必把 N 条消息塞进自己的上下文；也让"谁的答案是什么"成为可审计的板上事实。

**已知交互（写进了 README）**：圆桌的作答与汇总都是普通完成事件，所以启用评审时它们会被一并评审；
且作答任务对所有成员可读——**圆桌的私密性上限就是任务板的可见性**。

**真机验证**：Lead 的工具清单里出现 `roundtable`（3 次请求），teammate 的清单里 **0 次**；
整条链路（config → router → orchestrator）仍正常收尾。

**验证**：`packages/cluster` **87 个测试通过**（新增 12 个圆桌），`tsc -b tsconfig.host.json` EXIT=0，
13 项门禁全 0（配对 1012 对一致）。

## 11.16 M3 第三期：投票（motion）（已完成并真机验证）

按 §11.14 定下的编码实现：**每位投票人恰好一行选票**，立场写在描述里（末行
`vote: for` / `vote: against` / `vote: abstain`），`complete` 即"投出这一票"。

| 文件 | 改动 |
|---|---|
| `cluster/orchestrator/src/motion.ts` | **新增纯模块**：投票规划、**确定性计票**、全部文本 |
| `cluster/orchestrator/src/broadcast.ts` | 抽出 `collectorRefusal`，与圆桌共用"收集者校验" |
| `cluster/orchestrator/src/handoff.ts` | 通知可携带计数 |
| `cluster/orchestrator/src/index.ts` | 注册 `motion`；放行 tally 时把计数算好一并送出 |

四个决定：

1. **"一行一人"是关键，不是风格选择**。`Tally:` 任务的 `blockedBy` 因此**恰好就是选民名册**，
   而"所有阻塞项已完成"= "所有选票已投出"——**任务板自己的就绪算术就把票关上了**，
   不需要新状态、也不需要轮询。这是整个设计里最省的部分。
2. **计票由 orchestrator 确定性完成**，不交给模型数数：`readTally` 读的是票行，
   结果随指派通知一起送达，计票人核对的是一个数字。
3. **不可读的立场记为 `unrecorded` 并在通知里作为附加说明列出**，绝不当成弃权静默吞掉——
   宁可给出"计数与名册对不上"，也不给一个谁也核对不了的结论。
4. **弃权是一等立场**（`vote: abstain`），而不是"删除选票"。删除掉的行不算"已完成"，
   会让 `Tally:` **永远不被放行**——这是我设计时差点踩进去的坑，现在它是测试里的一条断言。

**真机验证**：Lead 的工具清单里三个工具齐全（`broadcast_message` / `roundtable` / `motion`），
teammate 的清单里**没有** `motion`；整条链路正常收尾。

**验证**：`packages/cluster` **106 个测试通过**（新增 18 个投票 + 1 个指派），
`tsc -b tsconfig.host.json` EXIT=0，13 项门禁全 0。

至此 M3 的三个协议里**广播、圆桌、投票都已落地**，只剩**辩论**——它是唯一还需要新状态表达的
（"谁在第几轮发言"必须能被表达，否则辩不起来）。

## 11.17 M3 修正：触达规则与"半途失败"（已完成）

三个工具交付后，我做了一次**溯源复查**——不是加功能，而是验证已交付的东西是否真的成立。
结论：闭环在 API 层面成立，但**触达规则漏了一类成员**，而三个工具都会因此**半途失败**。

### 复查确认成立的两条

1. **`createTask` 接受 `blockedBy`**（`task-board.ts:66`）——圆桌与投票的闭环在 API 层面成立，
   此前只是推断，现在是事实。
2. **`delete` 拒绝删除仍阻塞他人的任务**（`TEAM_TASK_HAS_DEPENDENTS`）——这**独立证实**了
   §11.16 里"弃权不能靠删除表达"的决定：选票正阻塞着 tally，任务板会直接拒绝删除。

### 挖出的缺陷：`provisioning` 成员"名单里有、却触达不到"

`TeamMemberPhase` 只有三态：`provisioning | active | failed`。
`TeamMemberView.status` 则是**运行时状态**：`failed` / `provisioning` / 否则取活体状态、缺失时为 `'inactive'`。
所以 `'inactive'` 的真实含义是"phase 已 active、只是当前没活起来"——**可以指派**
（我起初担心的那条**不成立**，这个澄清本身也值钱）。

真正触达不到的是 **`'provisioning'`**：`resolveActiveMember`（`roster.ts:51`）要求 `phase === 'active'`，
而**邮箱（`mailbox.ts:120`）与任务板（`task-board.ts:189`）走的是同一个解析器**。于是发消息会抛
`TEAM_MEMBER_NOT_FOUND`，指派任务同样抛。而 `listMembers` 从成员创建那一刻就把它列出来——
因此最自然的用法（**刚 `spawn_teammate` 完就发起扇出**）正好踩中：建行成功、指派抛错、
**工具中途失败、板上留下无主行**。

### 修法（两层）

1. **纯层**：`broadcastTargets` 与 `collectorRefusal` 增加 provisioning 判定与各自的原因。
   三个工具都建立在 `broadcastTargets` 之上，一处修正覆盖三者。
2. **执行层**：新增 `attempt()`，把每个目标的"建行 + 指派 + 投递"包成一次独立尝试，
   失败**记入 `skipped` 并附原因**，而不是抛错丢弃其余目标。若一个目标都没成功则明确抛错——
   因为**没有阻塞项的汇总/计票任务不会被任何就绪事件唤醒**，静默留下它等于永久卡住。

**验证**：`packages/cluster` **110 个测试通过**（新增 4 个），`tsc -b tsconfig.host.json` EXIT=0，
oxlint 0 warning 0 error，文档门禁全绿（配对 1012 / 模型体验 295 / 链接 2013 / 折行 2037），
`config-catalog` 未受影响（`Config` 接口行号未变）。双语 186:186 对齐。

**这一轮的真正价值**：没有交付新功能，而是**证实了两条此前只是推断的结论、修掉一个只在真跑时才暴露的缺陷**。
`provisioning` 这类目标纯函数测试**永远发现不了**——因为它不在"成员是谁"的语义里，
而在"成员此刻能不能被触达"的语义里。

## 11.18 M3 验证：让协议真的跑一遍，并修掉"让成员先 claim"（已完成）

上一轮我承认了最大的缺口：**三个工具的 `execute` 从未执行过**。这一轮把它补上，方法是
**进程外 E2E**——`.dev/mock-motion.mjs` 是一个会判别的脚本网关（按模型分别应答），
配合 `motion-cluster.yml` + `--patch` 叠加，驱动**真实 CLI + 真实 Agent Teams 服务**跑完一轮投票。

### 跑出来的结果（每一条都有日志证据）

- `motion` 真执行：返回 `tally: task-3`、`ballots: [coder→task-1, tester→task-2, status: accepted]`、`skipped: []`
- tally 建立时 **`"blockedBy":["task-1","task-2"]`**——**两张票都是它的阻塞项**
- 两张票各自由真实 teammate 读、写 `vote: for`、完成 ✓
- 最后一张票落地后，`handoff` 把 tally 指给 reviewer，通知里带着 **`2 for, 0 against, 0 abstain of 2 ballots`**
  ——**屏障成立，且计数是从选票行上读出来的**，不是谁报的
- 计票人被唤醒时 tally 为 `in_progress` + `ownerName: reviewer`，完成后 `"status":"completed"` ✓
- 断言脚本 `assert-motion.mjs`：**`all claims hold across 29 scripted calls`**（可复跑）

### 途中修掉的 mock bug（不是产品问题）

1. 会话标题那次辅助模型调用**吃掉了一个脚本步**，导致 `tester` 没被派生 → 改为识别标题调用、不推进回合计数。
2. 从请求体里取 revision 的正则匹配不到**被转义的** JSON（`\"revision\":`）→ 改为解析最新 tool 结果并递归找数字字段。
3. teammate 脚本在完成选票后无限重复 `send_message` → 改为发一次再以文本收尾。

### 挖出的**真产品缺陷**：让已经指派过的成员"先 claim"

任务板对 `claim` 的判定是 `status === 'pending'`（`task-board.ts:137`），
而编排器**在建行后立刻用 `reassign` 指派**（选票、答卷、汇总、计票、评审任务全都如此）
→ 成员读到的行已是 `in_progress`。于是**五处指令里的"先 claim"必然抛 `TEAM_TASK_BLOCKED`**：

| 位置 | 原指令 | 处置 |
|---|---|---|
| `handoff.ts` 交接通知 | "then claim using that revision" | 改为"已指派给你，无需 claim；完成后 complete" |
| `motion.ts` 选票描述 | 同上 | 同上 |
| `roundtable.ts` 答卷描述 | 同上 | 同上 |
| `roundtable.ts` 汇总描述 | "Claim it with the current revision" | 改为"释放时即指派给你" |
| `review.ts` 评审通知 | 同上 | 同上 |

**保留 claim 的两处**是正确的、也必须保留：`ready.ts` 的 `[TASK READY]` 通知（任务可能仍未指派）
与 `review.ts` 的驳回重试（`reopen` 会清空 owner，回到 pending）——前者改为条件措辞
（"若仍无主则 claim，否则直接完成"）。

**验证**：`packages/cluster` **110 个测试通过**；`tsc -b tsconfig.host.json` EXIT=0；
E2E 断言 29 次调用全绿。**这是本项目第一次"跑出来"而不是"读出来"的缺陷**。

**遗留缺口（诚实记录）**：E2E 三件套目前在 `.dev/`（被 gitignore），因此**不可从仓库复现**；
且只驱动了 `motion` 一条链，`broadcast_message` 与 `roundtable` 尚未被 E2E 跑过
（三者共用 `broadcastTargets` 与 `attempt`，但各自的 execute 仍需实跑）。

## 11.19 M3 验证（二）：把 E2E 收进仓库（已完成）

上一轮的缺口是"E2E 三件套在 `.dev/`（被 gitignore），本机可复跑但无法从仓库复现"。
这一轮按**仓库自己的 E2E 公约**把它收进仓库：

| 文件 | 作用 |
|---|---|
| `apps/cli/tests/cluster-motion.e2e.ts` | 起真实 CLI，断言 **session 事件日志** |
| `apps/cli/tests/profiles/headless/tests/fixtures/cluster-motion-llm.mjs` | 确定性、无密钥的 `LlmAdapter` fixture |

结果：**1 passed，15 秒**，无密钥、无网络、确定性执行。

### 三个关键决定

1. **不新建位置，沿用仓库范式**。E2E 放 `apps/cli/tests/*.e2e.ts`（由 `vitest.e2e.config.ts` 收集），
   模型替换沿用 `profiles/headless/tests/fixtures/*.mjs` 的 fixture 适配器约定（对照 `team-llm.mjs`）：
   **靠 teammate 身份提醒区分角色，靠对话内容选择下一步**。
2. **不改 `apps/cli` 的依赖清单**。profile 无法按包名解析 `@deepseek-ai/dsh-cluster-bundle`
   （bundle 先按 dsh 安装锚点解析，而该包不是本 app 的依赖）。我没有加依赖、也没有复制补丁，
   而是**直接 `--patch` 引用 cluster bundle 自己的 `cordis.patch.yml`**：补丁里的插件名从补丁自身位置解析，
   于是 **bundle 仍是"集群挂载了什么"的唯一事实来源**。
3. **断言 session 事件，而不是我自己的痕迹**：两张 `Ballot:` 均 completed、
   `Tally:` 的 `blockedBy` 等于两张票的 id、描述含 `cluster-owner: reviewer`、名册为三个 teammate、
   `tool/call` 含 `motion`，以及**计票通知里带着 `0 abstain of 2 ballots` 与 `claim nothing`**。

### 过程中修掉的两个 fixture 缺陷（都是我的）

1. **角色无关的守卫**：我照抄 `team-llm.mjs` 的"工具齐全性"前置检查，但 **teammate 没有 `motion`**
   （Lead 专属）→ 每个 teammate 请求都抛错 → Lead 无限等待 → CLI 挂到被 90 秒超时杀掉。
   改为**按角色检查**（Lead 要 `motion`，teammate 要任务板工具）。
2. **给 Lead 的读板循环加界**：超过 12 次 `team_task_list` 就输出 `CLUSTER_MOTION_STUCK`。
   把"挂住 90 秒后被杀、报错里什么都没有"变成"可读的失败"——这是这次能快速定位的直接原因。

**验证**：E2E **1 passed（15 秒）**；`packages/cluster` 110 个测试通过；`tsc -b tsconfig.host.json` EXIT=0；
oxlint 0 warning 0 error；**15 项门禁全 0**（含全部目录生成器与 `verify-application-entrypoints`）。
`scripts/repo-files.spec.ts` 有 9 项软链遍历用例在本机失败，与本次改动无关（新增的是普通文件，用例自身要建软链）。

**仍然缺的**：只覆盖 `motion` 一条链，`broadcast_message` 与 `roundtable` 的 execute 尚未被 E2E 跑过。
harness 现在可复用，补两条链属于增量工作。

## 11.20 M3 验证（三）：三条协议全部跑通（已完成）

上一轮只覆盖了 `motion`。这一轮补齐另外两条，**三种对话协议的 execute 现在都有仓库内的 E2E**：

| 文件 | 覆盖的链路 | 用时 |
|---|---|---|
| `apps/cli/tests/cluster-motion.e2e.ts` | 投票：两张选票 → 屏障 → 带计数的交接 → 计票完成 | 15s |
| `apps/cli/tests/cluster-roundtable.e2e.ts` | 圆桌：两份答卷 → 屏障 → 汇总任务（携带答卷清单）→ 汇总完成 | 18s |
| `apps/cli/tests/cluster-broadcast.e2e.ts` | 广播：一次**默认全体**扇出 → 三人各自应答 | 13s |

各配一份确定性 fixture 适配器，由 `vitest.e2e.config.ts` 的 include 自动收集，
因此与既有 E2E 一起在 e2e 套件里执行（**无密钥、无网络**）。
`vitest run --config vitest.e2e.config.ts apps/cli/tests/cluster-*.e2e.ts` → **3 passed / 20.7s**。

**广播这一条额外覆盖了"默认全体目标"路径**——motion 与 roundtable 用的都是显式名单，
`broadcastTargets` 的默认分支此前从未被执行过。它的断言直接读工具结果自身的 JSON：
`{"delivered":[coder accepted, tester accepted, reviewer accepted],"skipped":[]}`
——**"没人被跳过"成了被断言的事实**，正是 §11.17 触达修复生效的位置。

### 又一次"断言比被测代码更天真"

broadcast 首跑即失败，原因在我自己：断言里找的是未转义的 `"delivered"`，
而**工具结果的 JSON 是双重转义的**。改为**解析工具结果自身的 JSON** 再断言，而不是比字符串。
同一形态的错误在三个 fixture 里各出现一次（猜日志形状），结论一样：**能解析的就该解析**。

**验证**：三条 E2E **3 passed / 20.7s**；**15 项门禁全 0**；oxlint 0 warning 0 error。

**M3 到这里的真实状态**：三种对话协议（广播 / 圆桌 / 投票）**都已实现、有单元测试、并各自被真实 CLI 跑通**。
仍未做的只剩**辩论**——唯一需要新任务板状态的协议（回合顺序），以及把"三条链"写进 CI 的常规门禁说明。

## 11.21 M3 第四刀：辩论（已完成）

M3 的四种对话协议到这里齐了。辩论正是我在 §11.14 判定"可能需要先扩状态"的那一个——
这一轮的**设计结论是：不需要新状态**。

### 设计：轮次就是阻塞列表

`task-graph.ts` 对任务图的拒绝只有三种：`missing | duplicate | cycle`。
于是"第 r 轮的每个发言被第 r-1 轮的**全部**发言阻塞"既合法又充分：

- **开门**由就绪判定负责——上一轮全部完成后，下一轮的发言才 ready；
- **指派**由 `handoff` 负责——每个发言用 `cluster-owner:` 声明辩手，放行那一刻被指派并唤醒；
- **开场轮**创建即 ready，因此由工具自己指派并投递 `[DEBATE]` 通知（与圆桌的答卷同形）；
- **裁决**被终轮阻塞，"最后一位辩手说完"就是它的放行事实，而通知里携带**终轮有多少论证可读**
  （`debateSummary`，与 tally 携带计数同形）。

也就是说：**辩论 = 分层屏障**，完全落在 M2 已有的两个机制上，没有一行新的任务板状态。

### 落地

| 文件 | 改动 |
|---|---|
| `cluster/orchestrator/src/debate.ts` | **新增纯模块**：`debatePlan` / `readStatements` / `debateSummary` / 轮次语汇与三种文本 |
| `cluster/orchestrator/src/index.ts` | 注册 `debate` 工具；交接分支识别裁决并携带读数 |
| `cluster/orchestrator/src/handoff.ts` | 参数 `tally` → `carried`（现在也承载裁决读数） |

**准入规则**（全在纯层，全有测试）：少于 2 轮拒绝（"一轮作答是圆桌，不是辩论"）；
超过 5 轮、或发言总数超过 32 拒绝——**提前**拒绝，避免建到一半才撞 `maxTasks` 而留下半截任务板；
裁判先于辩手校验（与圆桌先校验收集者同理：唤不醒收集者的扇出会白花所有人的回合）；
**点名裁判当辩手则拒绝并说明**（"辩论需要一位没有参辩的裁判"）——这是辩论与圆桌唯一的语义差别；
默认名单是"除裁判外的全体"。

**验证**：`packages/cluster` **126 个测试通过**（新增 16 个）；`tsc -b tsconfig.host.json` EXIT=0；
oxlint 0 warning 0 error；**15 项门禁全 0**（含重记录的双语配对 203:203，
以及因 `Config` 接口行号位移而重新生成的 `docs/config-catalog.md`）。

### 辩论的 E2E（同日补上）

`apps/cli/tests/cluster-debate.e2e.ts` + `fixtures/cluster-debate-llm.mjs`：**1 passed，17 秒**。
它断言的核心是**分层屏障**，板上证据长这样：

```
task-1 Debate 1/2 [completed] blockedBy=[]
task-2 Debate 1/2 [completed] blockedBy=[]
task-3 Debate 2/2 [in_progress] blockedBy=["task-1","task-2"]
task-4 Debate 2/2 [in_progress] blockedBy=["task-1","task-2"]
task-5 Verdict:   [pending]    blockedBy=["task-3","task-4"]
```

即"第二轮被第一轮的**两个**发言共同阻塞、裁决被第二轮两个发言阻塞"，
且第二轮是**在第一轮全部完成之后**才被开门与指派的——这正是分层屏障要证明的东西。

**首跑暴露的是我 fixture 的缺陷（产品是对的）**：我用**整段对话**判断"是否已 `team_task_get`"，
于是辩手第二轮跳过 get，拿第一轮的陈旧 revision 去 edit/complete → 必然抛
`TEAM_TASK_STALE_REVISION` → 发言永远完不成。这正是"给 Lead 读板循环加界"换来的好处：
**17 秒就定位，而不是挂到 90 秒被杀、报错里什么都没有**。

**仍未做**：把四个工具提升为 `tool-*` 包以进入工具目录。

## 11.22 M4 第一刀：`/cluster` 命令（已完成）

M4 的形态在调查后确定：**斜杠命令**，而不是 `dsh` 子命令。理由：集群状态是**会话内的**，
而 `/cluster` 恰好运行在同一个会话里、能直接读 `agentTeams`；`dsh cluster status` 反而要先定位会话。
仓库里的 `/goal`、`/feedback`、`/compact` 也都是这个形态，`packages/<域>/command-*/` 是既有公约。

新增 `packages/cluster/command-cluster`，四条只读子命令：

| 子命令 | 输出 |
|---|---|
| `/cluster`（默认）/ `status` | 集群名、名册按状态计数、可触达 teammate 数、任务板按状态计数与 pending 的 ready/waiting |
| `/cluster tasks` | 每行一个共享任务：id、状态、标题、owner、就绪状态与全部阻塞项 |
| `/cluster agents` | 每个成员一行（Lead 在前）：名字、状态、角色、模型、诊断 |
| `/cluster graph` | 每条依赖一行：谁在等、等谁 |

`cost` 需要把编排器内部的花费折算暴露成服务接口，**留给下一刀**（已写进 README 的 Known Limitations）。

### 顺带修掉两处既有缺陷（都是这次才暴露的）

1. **4 个 cluster 别名被放进了 `tsconfig.base.json` 的"生成区"内部**——生成器重新生成时不会包含它们，
   于是 `gen-tsconfig-paths` **每次都失败**（我此前的门禁清单里恰好没有这一项，所以直到这次才暴露）。
   已移到 marker 之外的手写区并注明原因，该门禁现在通过。
2. **生成的中英 config 目录需要同步**：新包进入目录后，中文侧必须补上对应行，
   否则双语配对会在"第 211 个链接目标"处分歧。已补齐并重记录。

**验证**：新包 **10 个测试通过**；`tsc -b tsconfig.host.json` EXIT=0；oxlint 0 warning 0 error；
双语 README **109:109** 对齐；配对 **1013 对**一致；模型体验门禁 **296 个 README** 全通过（`explained none` 103）。

## 11.23 M4 第二刀（测试）：`/cluster` 接线测试，以及一处必须停下来的既有缺陷

新包此前只有纯渲染器被测，**注册与分派路径从未执行**——正是协议工具上反复出现的同一缺口。
补 `tests/command.spec.ts`：挂真实 `CommandRuntime`，用一个只提供 `listMembers/listTasks` 的 Team 服务替身，
经注册表执行 `/cluster`，断言（1）loader 看到的注册形状、（2）每条子命令各自抵达自己的报告、
（3）未知子命令用 usage 拒绝、（4）Team 的拒绝变成 **error 结果**而不是抛穿 dispatch。

**验证**：包内 **15 个测试通过**；`packages/cluster` **141 个测试通过**。

### 但这次多跑的一项门禁，报出了 19 条既有违规

为了改 devDeps，我这次多跑了一项此前**从未跑过**的门禁：`check-workspace-constraints`。
它一口气报出 **19 条违规**，全部属于 cluster 包族：

1. **发布成员形状**（bundle / config / orchestrator / router）：`private: true` 不该设、缺 `publishConfig.access`、
   `repository.url` 与 `files` 不符合本仓库的发布约定。
2. **非实验包不得依赖实验包**：bundle（2 条）、orchestrator、router，以及**本次新增的 `command-cluster`**。

也就是说：**这 4 个包从建立那天起就没有符合工作区约束**，而历次门禁清单里**恰好都没有这一项**——
与 §11.22 的 `gen-tsconfig-paths` 属同一类盲区（**清单不完整**，而不是运气）。
新包沿用了同样的形状，因此带上了同一个问题。

**处置**：不再往违规上叠代码。下一刀先**把 cluster 包族对齐工作区约束**：
逐个修正 package.json 的发布形状，并解决"实验依赖"那一条
（先确认 `experimental-package-policy` 的判定依据，再决定是改名、登记，还是改依赖类型）。

---

## 12. 下一步（按优先级）

1. **M2 编排收尾**：只剩轮次调度与上下文压缩策略。结构化 briefing、依赖自动解锁、评审回路、
   用量记账与预算执行都已完成并验证。
2. **M3 第四期：辩论**。三个协议里唯一还需要**新状态表达**的：圆桌是并行、投票是一次性收口，
   而辩论需要"谁在第几轮发言"的顺序。可行方向是把轮次显式化（§11.12 的结论）——
   例如发言任务带 `round: <n>` 标记并按轮次建立阻塞边，让任务板自己排出顺序；
   但"谁先发言""每轮几人"这些规则需要先想清楚，**不要先写代码**。
   另有：把三个工具提升为 `tool-*` 包以进入工具目录；
   确认 `topology` / `maxConcurrency` 是否要有真实消费者；轮次调度与上下文压缩策略。
3. **M4 可观测**：`dsh cluster status/tasks/agents/graph/cost` 命令族。用量记账（§11.7）已经把
   `cost` 需要的数据折好了，CLI 层可以直接读；再复用上游 `client-ui-agent-team` 扩展 Web 集群面板。
4. **纪律性提醒**：本项目反复出现的模式是"配置先声明、消费者后补"——`tokenBudget`、
   `orchestration.review`、`briefing` 都经历过这个阶段。新增任何配置字段时，要么同批给出消费者，
   要么在 README 的 Known Limitations 里写明它当前无人读取。
