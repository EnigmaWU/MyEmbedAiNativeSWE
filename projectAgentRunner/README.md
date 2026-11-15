# MyEmbedAiNativeSWE::projectAgentRunner

> Concrete implementation of `agentRunner` (`Sys2@R`) — the shared runtime engine for every `projectAgent`.

正如 `modelHub` 是 `System3`（`LLM-D/P/A`）的落地形态，`projectAgentRunner` 是 [projectAgent](../projectAgent/README.md) 中 `System2` 基础设施层核心构件 **`agentRunner`（`Sys2@R`）的具体实现**。它负责加载由 [projectAgentFramework](../projectAgentFramework/README.md) 打造或迁移的标准 `projectAgent`，将其真正运行为可会话、可并发执行、受治理的在线智能体。

## 定位与动机（WHY）

### 1. 为什么通用 Agent 运行时不够用（Why Not a Generic Agent Runtime）

市面上的通用智能体执行器大多围绕“单会话 Prompt + Tool 调用循环”设计，无法原生满足嵌入式软件工程中 `projectAgent` 的核心约束：

- **本体与执行体分离**：一个 `projectAgent` 需要常驻编排会话的服务本体，同时按 `todoUserStory` 并发拉起多个互不干扰的 `autoExecutor`，而非把会话接待与长耗时构建验证揉在同一个单线程循环里。
- **三通道隔离与原位入环**：`devChatProto`、`userChatProto`（含上下游智能体间的 `A2A` 调用）与 `execChatProto` 必须严格分流；同时 `callDEV` 与 `askDEV` 必须把 `DEV` 拉进**调用方既有会话**，而不是另起脱节的新通道。
- **跨挂起周期的有状态交付**：嵌入式研发常遇到需求边界待澄清或硬件依赖阻塞（`1-1R`），运行引擎必须原生支持单次 run 的 `suspend` 挂起、`UserStorySession` 回流重议（`Stage-1R`）以及 `resume`（等同重新 `open`）续跑。

### 2. 兑现“基座全池共享、项目各自自治”（Shared Engine, Isolated `System1`）

命名中的 `projectAgentRunner` 应读作 `(projectAgent)Runner`：它是**所有 `projectAgent` 共享的 `System2` 运行底座**，而非每个项目私自维护一套运行引擎。通过统一的运行底座，每个挂载其上的 `ModuleSourceAgent`、`FirmwarePackageAgent` 或 `AnyProjectAgent` 只需携带自身的 `System1`（`R+D+C` + `T+O+M`），即可获得多实例并发调度、协议网关与跨智能体 `A2A` 协同能力。

### 3. 将质量与治理约束硬化到运行层（Runtime-Enforced `O+G+V`）

在严肃的嵌入式工程中，关键红线不能只靠提示词约束，而必须由运行引擎在机制层强制落地：

- 多个并行 `autoExecutor` 绝不能直接踩踏同一份 `Sys1` 主工作区；
- 未附带验证证据（`verifiedReport`）或未经 `DEV` 签发为 `qualityReport(signed)` 的产物，绝不能从 `userChatProto` 流向 `USER`；
- 每一次阶段跃迁、入环决策与测试执行都必须留存可追溯的观测与审计记录。

## 定义与架构（WHAT）

### 1. 运行时拓扑（Runtime Topology）

```text
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│ System2 Infrastructure: projectAgentRunner (= agentRunner, Sys2@R)                            │
│                                                                                               │
│  ┌─ Mounted projectAgent #1 ─────────────────────────┐   ┌─ Mounted projectAgent #M ───────┐  │
│  │                                                   │   │                                 │  │
│  │  [Service Instance x 1] (projectAgent body)       │   │  [Service Instance x 1]         │  │
│  │   - Loads Sys1 (R+D+C SPEC + T+O+M Harness)       │   │   - Loads Sys1                  │  │
│  │   - Hosts chatDEV / chatUSER / chatEXECUTOR       │   │   - Hosts 3 chatProto endpoints │  │
│  │   - Persists UserStorySession & Stage-0..Final    │   │   - Persists UserStorySession   │  │
│  │   - In-place topOfAgent routing (callDEV/askDEV)  │   │                                 │  │
│  │                       │                           │   │                 │               │  │
│  │                 execChatProto                     │   │           execChatProto         │  │
│  │           ┌───────────┴───────────┐               │   │                 ▼               │  │
│  │           ▼                       ▼               │   │  [autoExecutor #1 .. #K]        │  │
│  │  [autoExecutor #1]       [autoExecutor #N]        │   │   - Isolated Workspaces         │  │
│  │   - Story #1 Sandbox      - Story #N Sandbox      │   │                                 │  │
│  └───────────────────────────────────────────────────┘   └─────────────────────────────────┘  │
│                                                                                               │
│  ┌─ Built-in Infrastructure Hooks (Sys2@O+G+V) ────────────────────────────────────────────┐  │
│  │  [O] Observation (Telemetry/Trace)  [G] Governance (Policy/Conflict)  [V] Verification  │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────┬───────────────────────────────────────────────────┬────────────────────┘
                       │ Sys2 business calls                               │ Sys3 LLM calls (ApiKEY)
                       ▼                                                   ▼
       ┌───────────────────────────────┐                   ┌───────────────────────────────┐
       │ Sys2 Business: KM + MCP + CAT │                   │ Sys3: modelHub ([D] [P] [A])  │
       └───────────────────────────────┘                   └───────────────────────────────┘
```

> 注：`leftOfAgent` 中 `DEV` 使用的 `baseCodeAgent`（如 Copilot / Codex 等编码智能体）运行在 `projectAgentRunner` 之外；运行器内的 `chatDEV` 只负责会话接入与编排，不重复实现底层编码工具。

### 2. 核心子系统（`defineRunner`）

- **双层实例托管引擎（Dual-Tier Instance Hosting）**
  - **单服务实例（`1 Service Instance` / `projectAgent`）**：每个挂载的 `projectAgent` 对应唯一常驻服务实例，装载该项目的 `Sys1@T+O+M` 与主工作区视图，维护 `UserStory` 账本与持久化 `UserStorySession`。
  - **弹性执行实例池（`1..N autoExecutor Instances`）**：默认至少承载 1 个执行体，并随活跃 `todoUserStory` 数量动态扩缩至 $N$ 个；每个 `autoExecutor` 专注闭环单个 `UserStory`（`todo -> doing ⇄ review -> done`，支持任意中间态 `abort`）。
- **三通道协议网关与原位入环路由（Tri-Channel Gateway & In-Place Routing）**
  - 独立暴露并隔离 `devChatProto`（面向真人 `DEV` 的自然语言会话）、`userChatProto`（面向真人 `USER` 的自然语言会话，或面向父级 `FirmwarePackageAgent` 的 `A2A` 协议）与 `execChatProto`（本体与 `autoExecutor` 间的内部 `A2A` 或私有协议）。
  - 原生实现 `topOfAgent` 会话叠加：收到 `/callDEV` 时将 `DEV` 桥接进当前 `userChatProto` 会话；收到 `/askDEV` 时将 `DEV` 桥接进当前 `execChatProto` 会话，确保上下文零割裂。
- **阶段与 Run 生命周期调度器（Stage & Run State Machine）**
  - 驱动 `Stage-0`（澄清共识）、`Stage-1`（派发执行与审查闭环）、`Stage-1R`（重议自环）与 `Stage-Final`（交付或判弃终结）的确定性状态跃迁。
  - 管理单次 run 边界：当 `askDEV` 无法解决触发 `1-1R` 时，安全冻结（`suspend`）对应 `autoExecutor` 上下文并将会话汇总回流 `USER`；待 `1R-1` 确认修改后执行 `resume`（等同重新 `open`）恢复执行。
- **并发工作区隔离与 `O+G+V` 底座协同（Workspace Isolation & `O+G+V` Enforcement）**
  - **工作区隔离与 `Governance`（`G`）**：为并行运行的多个 `autoExecutor` 分配独立沙箱/工作树副本，禁止直接并发改写共享 `Sys1` 工作区；合并冲突、越权工具调用及未签发 `qualityReport(signed)` 的交付动作均由 `G` 策略强制拦截。
  - **可观测性 `Observation`（`O`）与验证 `Verification`（`V`）**：全链路采集三通道会话轨迹、`modelHub`（`[D]` / `[P]` / `[A]`）推理开销与 `CAT` 软硬件测试流水线结果，自动汇聚验证证据生成 `verifiedReport`，支撑 `qualityReport(draft -> signed)` 签发闭环。
