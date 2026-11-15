# MyEmbedAiNativeSWE::projectAgentFramework

> Build a `newProjectAgent` from scratch, or evolve a legacy repo into a `portProjectAgent` — with one unified framework.

[projectAgent](../projectAgent/README.md) 定义了“项目智能体是什么”（`AGENT = System1 + System2 + System3`）；`projectAgentFramework` 回答**为什么要有一套标准框架来打造与迁移它（WHY）**，以及这套框架**由什么构成（WHAT）**。

## 定位与动机（WHY）

### 1. 剥离不变机制与业务载荷（Separate Invariants from Domain Payload）

无论目标是交付模块的 `ModuleSourceAgent`（micro）、打包固件的 `FirmwarePackageAgent`（macro），还是泛化的 `AnyProjectAgent`，它们都遵循同一套工程不变量：

- 角色拓扑：`DEV`、`USER`、`AGENT` 本体与 `1..N` 个 `autoExecutor`
- 三条独立通道：`devChatProto`、`userChatProto`（自然语言或 `A2A`）、`execChatProto`
- 原位入环机制：`callDEV` 与 `askDEV` 进入调用方既有会话，不新开通道
- 四段生命周期：`Stage-0`（澄清）、`Stage-1`（执行）、`Stage-1R`（重议自环）、`Stage-Final`（终结）
- 质量门禁：`selfVerify` 产出 `qualityReport(draft)`，经 `DEV` 审查签发 `qualityReport(signed)` 后方可交付

如果没有 `projectAgentFramework`，每个项目在成为 `projectAgent` 时都要把上述协议、状态机、会话持久化和底座接线重复实现一遍。框架存在的首要目的，是让项目团队**只编写和沉淀本项目的 `System1` 独有资产**，而把通用骨架交给框架提供。

### 2. 收敛“全新构建”与“存量演进”两条路径（Converge `new` and `port`）

嵌入式软件工程同时面对两类起点截然不同的项目：

- **全新项目（`newProjectAgent`）**：从零开始，天然适合正向建立 `SPEC`（`youWantR + iWhyD + aiHowC`）与 `T+O+M` Harness。
- **传统存量项目（`portProjectAgent`）**：现实中大量工程只有一份“可能可用的 `C`”，缺少结构化的 `R` 与 `D`，构建与测试散落在人工脚本或经验里。

`projectAgentFramework` 为这两条方向相反的演进路径提供统一的终点契约：无论出身是 `new` 还是 `port`，最终产出的都是结构同构、契约一致的标准 `projectAgent`，可无差别挂载到同一个 [projectAgentRunner](../projectAgentRunner/README.md) 上运行。

### 3. 支持自举演进（Self-Hosting as an `AnyProjectAgent`）

依据 `AnyProjectAgent` 的泛化定义，`projectAgentFramework` 自身同样可以演进并运行为一个 `projectAgent`（智能体工厂）：此时目标项目的 `DEV` 成为它的 `USER`，提出“新建某模块智能体”或“迁移某存量固件仓”的 `Feature`，经 `Stage-0 -> Stage-1 -> qualityReport(signed)` 闭环交付一个可验证、可运行的标准 `projectAgent`。

## 定义与边界（WHAT）

### 1. 三位一体边界（Definition vs. Framework vs. Runner）

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ projectAgent (Definition / Meta-Model)                                                 │
│   Defines: AGENT = System1(own) + System2(shared) + System3(shared)                    │
└───────────────────┬────────────────────────────────────────────────┬───────────────────┘
                    │ build / port contract                          │ runtime contract
                    ▼                                                ▼
┌───────────────────────────────────────────┐    ┌───────────────────────────────────────┐
│ projectAgentFramework (Build & Port SDK)  │    │ projectAgentRunner (Impl of Sys2@R)   │
│                                           │    │                                       │
│ - Sys1 Scaffold (R+D+C SPEC + T+O+M)      │───▶│ - Hosts 1 Service + 1..N autoExecutor │
│ - chatProto & Stage Data Contracts        │pkg │ - Routes dev / user / execChatProto   │
│ - newProjectAgent & portProjectAgent Kits │    │ - Enforces O+G+V & Workspace Isolation│
│ - Sys2 (KM/MCP/CAT) & Sys3 Connectors     │    │ - Connects to CAT / MCP / modelHub    │
└───────────────────────────────────────────┘    └───────────────────────────────────────┘
```

| 模块 | 对应层级 | 核心职责（WHAT） | 作用阶段 |
| :--- | :--- | :--- | :--- |
| [projectAgent](../projectAgent/README.md) | 架构元模型 | 定义角色、协议、四段状态机与 `Sys1 + Sys2 + Sys3` 组成公式 | 顶层设计基线 |
| `projectAgentFramework` | 打造与迁移框架 | 提供 `Sys1` 标准骨架、协议契约、`Sys2/Sys3` 连接器及 `new` / `port` 工程套件 | 构建期与演进期 |
| [projectAgentRunner](../projectAgentRunner/README.md) | `Sys2@R`（`agentRunner` 实现） | 加载符合框架契约的 `projectAgent`，运行 `1 service + 1..N autoExecutor` 实例 | 运行期与治理期 |
| `modelHub` | `Sys3`（`LLM-D/P/A` 实现） | 通过受控 `ApiKEY` 对外统一提供 `[D]` / `[P]` / `[A]` 三档推理模型服务 | 全周期推理支撑 |

### 2. 核心组成（`defineFramework`）

- **`Sys1` 标准骨架（`Sys1` Manifest & Scaffold）**
  - **`R+D+C` 规格模板**：对接 [MyCaTDD](https://github.com/EnigmaWU/MyCaTDD) 活体规格体系，标准化组织 `youWantR`（需求与验收）、`iWhyD`（架构决策与质量属性）与 `aiHowC`（实现映射与代码资产）。
  - **`T+O+M` Harness 模板**：标准化声明项目自有的 `Tools / slashCommands / Skills`、`Orchestration` 工作流配置，以及短期会话上下文与长期项目记忆（含 `abortUserStory` 留存记忆）的目录契约。
- **角色通道与阶段契约（`chatProto` & Lifecycle Schemas）**
  - 定义 `chatDEV`、`chatUSER`、`chatEXECUTOR` 的标准消息载荷，兼容自然语言会话与上下游 `AGENT` 间的 `A2A` 协议。
  - 定义 `UserStory`（`todo` / `doing` / `review` / `done` / `abort` 及 `suspend` 挂起态）、`UserStorySession`、`verifiedReport` 与 `qualityReport(draft -> signed)` 的统一结构。
- **双路径工程套件（`newProjectAgent` & `portProjectAgent` Kits）**
  - **`newProjectAgent` 正向套件（Greenfield：`R -> D -> C`）**：从空白项目生成标准 `Sys1` 清单与目录骨架，绑定 `ModuleSourceAgent` / `FirmwarePackageAgent` / `AnyProjectAgent` 类型模板，配置 `selfVerify` 最小闭环后直接进入 `Stage-0`。
  - **`portProjectAgent` 迁移套件（Brownfield：Harness 先行 + `C -> D -> R` 逆向萃取）**：面向既有代码仓，先将遗留编译、烧录、静态检查与测试封装进 `T+O+M` 与 `MCP` / `CAT` 以建立 `selfVerify` 安全网，再从存量 `C` 逆向提炼 `iWhyD` 与 `youWantR` 补齐活体 `SPEC`，实现平滑纳管。
- **共享底座连接器（`Sys2` & `Sys3` Connectors）**
  - 声明式接入 `System2` 的业务支撑层（`KM/Skill` 语义检索与 `GraphRAG`、`MCP` 遗留服务、`CAT` 软硬件持续自动测试）与基础设施层（对接 `projectAgentRunner` 及 `O+G+V` 钩子）。
  - 声明式接入 `System3`（`modelHub`），按任务性质绑定 `codeGenReasoningDeepest`（`[D]`）、`codeGenReasoningLatest`（`[P]`）与 `codeGenLatest`（`[A]`）。
