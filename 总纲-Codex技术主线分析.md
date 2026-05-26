# 总纲 — OpenAI Codex 技术主线分析

> 这是《Codex 源码深度研究》的总纲。后续 25 章是按子系统展开的“局部深挖”；本文只负责“骨架 + 设计哲学 + 火爆原因 + 差异化”四件事。  
> 写作口径：先给事实，再给判断；判断均附源码路径行号或外部公开链接。

---

## 1. 项目坐标

### 1.1 规模快照：Codex 是一个“多层产品工程”，不是单点 CLI

按 2026-05-26 的可复核口径，Codex 的规模可以用四个数字刻画：

- **crate 数：113**（直接来自 workspace members 枚举，`codex-rs/Cargo.toml:L2-L116`）。
- **代码行数（本地统计口径）：1,168,717**；**tracked files：4,655**（`git ls-files` + 本地逐文件计行脚本，同日复核）。
- **GitHub Stars：85,789**（`stargazers_count`，见 [openai/codex repo API](https://api.github.com/repos/openai/codex)）。
- **GitHub contributors（分页累计）：454**（见 [contributors API page1](https://api.github.com/repos/openai/codex/contributors?per_page=100&page=1) 并按页累加）。

如果只看安装命令，Codex 像一个 CLI；如果看这四个量级，它更像“一个统一产品壳 + 多执行面 + 多协议层 + 多安全后端”的平台型工程。

### 1.2 三大入口：Rust 主体 / npm 包装 / TS + Python SDK

Codex 的入口不是单点，而是三类面向不同用户群体的入口。

1. **Rust 主体（主执行面）**  
   `codex-rs/cli/src/main.rs` 同时暴露 `app-server`、`mcp-server`、`cloud`、`tui`、`exec` 等命令面（`codex-rs/cli/src/main.rs:L437-L560`），说明 CLI 不是薄封装，而是全局功能路由。

2. **npm 包装（跨平台分发面）**  
   npm 包名是 `@openai/codex`，`bin` 指向 `bin/codex.js`（`codex-cli/package.json:L2-L8`）。  
   `codex.js` 先做 OS/arch 识别，再映射并拉起平台二进制包，最终 `spawn` native 程序（`codex-cli/bin/codex.js:L24-L36`, `L15-L22`, `L184-L187`）。  
   这层核心价值是“统一安装体验 + 二进制分发兼容”。

3. **TS / Python SDK（程序化入口）**  
   TypeScript SDK 文档明确其通信方式是 CLI 子进程 + stdin/stdout JSONL 事件流（`sdk/typescript/README.md:L5-L6`）。  
   Python SDK 明确 `app-server` JSON-RPC v2 over stdio（`sdk/python/README.md:L3-L4`），并要求 Python 3.10+（`sdk/python/pyproject.toml:L14`）。  
   这意味着 SDK 面向的是“把 Codex 接到系统里”，而不仅是“人工在终端里对话”。

从入口分层看，Codex 把“安装分发”“本地运行”“程序集成”拆成了不同责任域，但保持一套核心执行语义，降低了跨入口行为漂移。

### 1.3 开源治理特殊性：可见源码 + 受控贡献

`docs/contributing.md` 明确写出两点：  
- 外部贡献是 **invitation-only**（`docs/contributing.md:L3`）；  
- 提交 PR 前必须签署 **CLA**（`docs/contributing.md:L80-L92`）。

这是一种“开放代码可读 + 贡献流程收敛”的治理模型。它的直接后果是：

- 社区可见度高（Star、fork、issue 讨论活跃），但核心演进节奏主要由官方路线控制；
- 对外部读者，源码研究价值更高，因为主线设计常能保持一致性；
- 对外部贡献者，参与门槛更接近“生态协作”而非“大规模社区共管”。

这不是好坏二元，而是治理目标不同：Codex 更像“产品化开放核心”，而不是“自治式社区工程”。

---

## 2. 设计哲学（至少 6 条）

这一节不展开到实现细节，而是从源码组织、协议边界、运行模型中抽出“反复出现的设计取向”。每条先列事实，再给判断。

### 2.1 本地优先、云端补位（Local-first with Cloud Extension）

**事实**：

- 官方 README 把 Codex 定位成“lightweight coding agent you can run in your terminal”，并强调可在本地工程目录直接工作（`README.md:L1-L8`）。
- TUI 和 CLI 都是本地可运行面（`codex-rs/cli/src/main.rs:L507-L560`, `codex-rs/tui/src/lib.rs:L863-L946`）。
- 同时又有 `cloud-tasks` 作为补位能力，提供 `create/list/status/diff/apply` 等云端任务接口（`codex-rs/cloud-tasks/src/lib.rs:L157-L182`, `L492-L585`）。

**判断**：  
Codex 没有把“云端代理”当唯一答案，而是采用“本地执行主循环 + 云端承担异步/远程场景”的双轨设计。这种取向更利于开发者从零迁移：先本地使用，再按需上云，不必一次重构工作流。

### 2.2 Rust 二进制是单一可信执行入口（Single Trusted Runtime）

**事实**：

- npm 启动器最终总是落到平台 native 二进制（`codex-cli/bin/codex.js:L184-L187`）。
- Rust CLI 主程序统一承接各执行面（`codex-rs/cli/src/main.rs:L437-L560`）。
- `arg0` 机制允许同一二进制按 `argv[0]` 分发到不同子能力（`codex-rs/arg0/src/lib.rs:L152-L175`），减少“多可执行文件行为偏差”。

**判断**：  
这种模式强化了“单运行时语义一致性”：不论是 npm 用户、直接二进制用户还是某些包装入口，最后都尽量进入同一 Rust 执行域，降低跨入口 bug 的不确定性。

### 2.3 协议层是产品边界，不是内部细节

**事实**：

- `app-server-transport` 明确支持 `Stdio / UnixSocket / WebSocket / Off`（`codex-rs/app-server-transport/src/transport/mod.rs:L67-L72`）。
- `app-server` 运行入口可注入 transport options（`codex-rs/app-server/src/lib.rs:L420-L433`）。
- `app-server-protocol` 在 `common.rs` 里显式区分客户端方法定义，并导出 v1/v2 结构（`codex-rs/app-server-protocol/src/protocol/common.rs:L437-L465`, `codex-rs/app-server-protocol/src/protocol/v2/mod.rs:L1-L54`）。
- 仓库 AGENTS 文档明确：“active API development should happen in v2; do not add new API surface area to v1”（`AGENTS.md:L186-L190`）。

**判断**：  
Codex 把协议层当“可演进的契约产品”：transport 可替换，协议版本可并存迁移，调用方面向 SDK/IDE 稳态。这是典型平台工程思路，不是单体应用思路。

### 2.4 沙箱矩阵优先于“单一安全方案”

**事实**：

- `SandboxManager` 在统一入口里分派不同平台策略（`codex-rs/sandboxing/src/manager.rs:L23-L28`）。
- macOS 通过 `sandbox-exec` + seatbelt profile（`codex-rs/sandboxing/src/seatbelt.rs:L25-L29`）。
- Linux 组合了 `bubblewrap` 文件系统隔离与 Landlock/seccomp 限制（`codex-rs/linux-sandbox/src/bwrap.rs:L58-L67`, `codex-rs/linux-sandbox/src/landlock.rs:L57-L69`, `L119-L169`）。
- Windows 侧包含 restricted token 与网络拦截判定（`codex-rs/windows-sandbox-rs/src/lib.rs:L255-L279`, `codex-rs/windows-sandbox-rs/src/resolved_permissions.rs:L33-L101`）。

**判断**：  
Codex 的安全策略不是“最强单点”，而是“跨 OS 的最小可行共识 + 各平台增强”。这使得产品可以统一叙事（都有 sandbox），同时接受底层实现异构。

### 2.5 Plugin / MCP 不只是扩展点，而是增长接口

**事实**：

- `core-plugins` 有安装、启停、状态与技能信息聚合（`codex-rs/core-plugins/src/manager.rs:L194-L224`）。
- 启动时会同步 OpenAI 插件仓（支持 git/HTTP fallback）（`codex-rs/core-plugins/src/startup_sync.rs:L66-L99`）。
- `codex-mcp` 维护 MCP server 连接生命周期与工具发现（`codex-rs/codex-mcp/src/connection_manager.rs:L71-L111`）。
- `mcp-server` 可直接作为独立 server 运行（`codex-rs/mcp-server/src/lib.rs:L59-L151`）。

**判断**：  
这套结构使 Codex 的能力边界可以由“核心团队写代码”扩展为“外围生态提供工具”。在产品增长上，这通常比单体内建功能更可持续。

### 2.6 持久化是“第一等公民”，不是日志副产物

**事实**：

- `rollout::Recorder` 会落 `RolloutItem`，支持 `persist / flush / load`（`codex-rs/rollout/src/recorder.rs:L758-L814`）。
- `state` crate 固定了 SQLite 状态库文件名（`codex-rs/state/src/lib.rs:L78-L80`）。
- `state::runtime::threads` 中有线程元数据 upsert/get 等 SQL 操作（`codex-rs/state/src/runtime/threads.rs:L494-L584`, `L820-L886`）。
- `core::state_db_bridge` 在会话侧初始化状态库（`codex-rs/core/src/state_db_bridge.rs:L6-L7`）。

**判断**：  
Codex 的会话设计目标不是“临时聊天”，而是“可恢复、可追踪、可复盘”的执行轨迹系统。对工程团队而言，这决定了它更容易进入审计、调试、团队协作场景。

### 2.7 AGENTS.md 替代“散落 prompt”的团队协作方式

**事实**：

- `core/src/agents_md.rs` 说明 `AGENTS.md` 会在仓库树中向上查找并按顺序合并（`codex-rs/core/src/agents_md.rs:L8-L16`）。
- `docs/agents_md.md` 也强调其为“对人类与 agent 都可读的仓库级指导文件”（`docs/agents_md.md:L1-L12`）。

**判断**：  
这代表 Codex 把“提示词”从个人临时输入，转化为版本化、可审查的仓库工件。对团队研发来说，这是从“人脑记忆”转向“工程约束”的关键一步。

### 2.8 事件驱动主循环：把 agent 行为拆成可控操作集

**事实**：

- 协议层定义 `Submission` 与 `Op`（`UserInput`、`ExecApproval`、`Compact`、`Shutdown` 等）（`codex-rs/protocol/src/protocol.rs:L126-L168`, `L479-L540`）。
- `submission_loop` 对这些操作做统一分发（`codex-rs/core/src/session/handlers.rs:L708-L856`）。
- `session/mod.rs` 在会话上下文里整合 model、工具、沙箱、持久化与状态（`codex-rs/core/src/session/mod.rs:L1-L300`）。

**判断**：  
Codex 的核心不是“一个大 prompt 调一次模型”，而是“一个持续运行的、可中断/可批准/可压缩上下文的状态机”。这也是它能承载长会话与复杂工具调用的基础。

---

## 3. 整体架构图

先给骨架，再看细节。下面这张图只表达“层级与责任边界”，不展开具体实现。

<div bgcolor="#ffffff">

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"background": "#ffffff","primaryColor": "#f5f5f5","primaryTextColor": "#000000","primaryBorderColor": "#333333","lineColor": "#444444","secondaryColor": "#f8f8f8","tertiaryColor": "#ffffff","clusterBkg": "#fafafa","clusterBorder": "#888888","edgeLabelBackground": "#ffffff","fontFamily": "Helvetica"}, "themeCSS": "svg { background: #ffffff !important; } .label, .nodeLabel, .edgeLabel, text { fill: #000000 !important; color: #000000 !important; }"}}%%
flowchart LR
  subgraph L1[入口层]
    NPM[npm CLI<br/>@openai/codex]
    RCLI[Rust CLI<br/>codex-rs/cli]
    SDK[TS/Python SDK]
    MCPIN[MCP Server Entry]
  end

  subgraph L2[协议层]
    APS[app-server]
    APST[app-server-transport<br/>Stdio/UDS/WS]
    APROTO[app-server-protocol<br/>v1 + v2]
  end

  subgraph L3[Agent 层]
    CORE[core]
    SESSION[session/submission_loop]
    TOOLS[tools + hooks]
  end

  subgraph L4[服务层]
    MODEL[model-provider + login]
    NP[network-proxy]
    ROLL[rollout]
    STATE[state + thread-store]
    MEM[memories]
    PLUG[core-plugins + core-skills]
    MCP[codex-mcp]
  end

  subgraph L5[沙箱层]
    SBM[sandboxing::Manager]
    MAC[macOS Seatbelt]
    LNX[Linux bwrap + Landlock]
    WIN[Windows RestrictedToken + WFP]
    EXP[execpolicy]
  end

  subgraph L6[UI/交付层]
    TUI[tui]
    CLOUD[cloud-tasks]
    OUTCLI[cli surfaces]
  end

  NPM --> RCLI
  SDK --> APS
  MCPIN --> APS
  RCLI --> APS
  APS --> APST
  APS --> APROTO
  APS --> CORE
  CORE --> SESSION
  SESSION --> TOOLS
  SESSION --> MODEL
  SESSION --> NP
  SESSION --> ROLL
  SESSION --> STATE
  SESSION --> MEM
  SESSION --> PLUG
  SESSION --> MCP
  TOOLS --> SBM
  SBM --> MAC
  SBM --> LNX
  SBM --> WIN
  SBM --> EXP
  APS --> TUI
  APS --> CLOUD
  APS --> OUTCLI
```

</div>

这张图对应的源码锚点如下：

- 入口层：`codex-cli/package.json:L2-L8`, `codex-cli/bin/codex.js:L24-L36`, `codex-rs/cli/src/main.rs:L437-L560`, `sdk/typescript/README.md:L5-L6`, `sdk/python/README.md:L3-L4`。
- 协议层：`codex-rs/app-server/src/lib.rs:L420-L433`, `codex-rs/app-server-transport/src/transport/mod.rs:L67-L72`, `codex-rs/app-server-protocol/src/protocol/common.rs:L437-L465`。
- Agent 层：`codex-rs/core/src/lib.rs:L12-L48`, `codex-rs/core/src/session/handlers.rs:L708-L856`。
- 服务层：`codex-rs/rollout/src/recorder.rs:L758-L814`, `codex-rs/state/src/runtime/threads.rs:L494-L584`, `codex-rs/core-plugins/src/manager.rs:L194-L224`, `codex-rs/codex-mcp/src/connection_manager.rs:L71-L111`。
- 沙箱层：`codex-rs/sandboxing/src/manager.rs:L23-L28`, `codex-rs/sandboxing/src/seatbelt.rs:L25-L29`, `codex-rs/linux-sandbox/src/bwrap.rs:L58-L67`, `codex-rs/windows-sandbox-rs/src/lib.rs:L255-L279`, `codex-rs/execpolicy/src/parser.rs:L57-L61`。
- UI 层：`codex-rs/tui/src/lib.rs:L863-L946`, `codex-rs/cloud-tasks/src/lib.rs:L157-L182`。

---

## 4. 数据流：一次 `codex` 命令背后发生了什么

这一节只回答“主路径发生了什么”，不讨论边缘分支。以最常见路径为例：用户通过 npm 安装，终端输入 `codex`，进入 TUI 会话并触发工具调用。

### 4.1 主路径分解（事实）

1. **命令入口**：shell 调到 `codex`，若来自 npm 安装，先进入 `bin/codex.js` 做平台判断（`codex-cli/bin/codex.js:L24-L36`）。
2. **native 交接**：启动器确定平台包后，`spawn` Rust 二进制（`codex-cli/bin/codex.js:L184-L187`）。
3. **CLI 路由**：Rust CLI 解析子命令并进入 TUI/app-server 等分支（`codex-rs/cli/src/main.rs:L437-L560`）。
4. **in-process app-server 启动**：`app-server-client` 作为“进程内门面”封装 `codex_app_server::in_process`，统一握手、请求分发、事件回传（`codex-rs/app-server-client/src/lib.rs:L1-L16`, `L67-L71`）。
5. **会话主循环**：`submission_loop` 接收 `Op::UserInput` 等操作，驱动模型请求、工具调用、审批与上下文管理（`codex-rs/core/src/session/handlers.rs:L708-L856`）。
6. **工具执行与沙箱**：工具侧命令最终受 sandbox + policy 约束执行（`codex-rs/sandboxing/src/manager.rs:L23-L28`, `codex-rs/execpolicy/src/policy.rs:L28-L41`）。
7. **流式事件回写**：in-process event stream 对关键通知采用“必须投递”策略，避免 TUI 文本损坏或 turn completion 丢失（`codex-rs/app-server-client/src/lib.rs:L151-L186`）。
8. **界面渲染**：TUI 订阅事件并刷新 UI（`codex-rs/tui/src/lib.rs:L863-L946`）。
9. **轨迹与状态持久化**：`rollout` 写 JSONL，`state` 写 SQLite 线程/配置元数据（`codex-rs/rollout/src/recorder.rs:L758-L814`, `codex-rs/state/src/runtime/threads.rs:L494-L584`, `codex-rs/core/src/state_db_bridge.rs:L6-L7`）。

### 4.2 这条链路的意义（判断）

Codex 的关键不是“单次请求成功”，而是“链路可持续”：  
命令入口一致、协议事件可回放、关键通知尽量无损、会话状态能恢复。对于长任务和多回合 agent 来说，这比一次 patch 是否写对更重要。

<div bgcolor="#ffffff">

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"background": "#ffffff","primaryColor": "#f5f5f5","primaryTextColor": "#000000","primaryBorderColor": "#333333","lineColor": "#444444","secondaryColor": "#f8f8f8","tertiaryColor": "#ffffff","edgeLabelBackground": "#ffffff","fontFamily": "Helvetica"}, "themeCSS": "svg { background: #ffffff !important; } .messageText, text { fill: #000000 !important; color: #000000 !important; }"}}%%
sequenceDiagram
  autonumber
  participant U as User
  participant N as npm launcher (codex.js)
  participant C as Rust CLI
  participant A as in-process app-server
  participant S as core session loop
  participant M as Model API
  participant T as Tool runtime
  participant B as Sandbox/Policy
  participant UI as TUI
  participant R as rollout(JSONL)
  participant DB as state(SQLite)

  U->>N: codex "task..."
  N->>C: spawn platform binary
  C->>A: init app-server client
  A->>S: Submission(Op::UserInput)
  S->>M: request model turn
  M-->>S: stream deltas + tool calls
  S->>T: execute tool
  T->>B: sandbox + execpolicy/network policy check
  B-->>T: allow/deny/ask
  T-->>S: tool result
  S-->>A: server notifications/events
  A-->>UI: streamed deltas + completions
  S->>R: persist rollout items
  S->>DB: upsert thread metadata
  UI-->>U: render turn result
```

</div>

---

## 5. 关键子系统盘点（25 章映射）

这一节做“目录级导航”，避免把单章内容提前写完。映射来源为 `scripts/chapters.yaml`（`scripts/chapters.yaml:L28-L311`）。

### 5.1 Part I / II / III 全量清单（每章一行）

| Part | 章节 | 定位 | 关键源码路径（示例） | 章节文档 |
|---|---|---|---|---|
| Part I | ch01 项目全景与设计哲学 | 总览入口、治理模式、哲学抽取 | `codex-rs/Cargo.toml` | [01-项目全景与设计哲学.md](01-项目全景与设计哲学.md) |
| Part I | ch02 多入口与启动分发 | npm→Rust→arg0 分发链 | `codex-cli/bin/codex.js` | [02-多入口与启动分发.md](02-多入口与启动分发.md) |
| Part I | ch03 配置系统与企业要求 | 配置树、企业约束、fail-closed | `codex-rs/core/src/config/mod.rs` | [03-配置系统与企业要求.md](03-配置系统与企业要求.md) |
| Part I | ch04 初级使用方法 | install/login/exec/TUI 基础流 | `docs/getting-started.md` | [04-初级使用方法.md](04-初级使用方法.md) |
| Part I | ch05 高级使用方法 | cloud/fork-resume/plugin/mcp | `codex-rs/cloud-tasks/src/lib.rs` | [05-高级使用方法.md](05-高级使用方法.md) |
| Part I | ch06 Agent 核心循环 | session/turn 状态机主线 | `codex-rs/core/src/session/mod.rs` | [06-Agent核心循环.md](06-Agent核心循环.md) |
| Part I | ch07 Prompt 组装与 Skill 注入 | prompt 家族 + skills 注入链 | `codex-rs/core-skills/src/injection.rs` | [07-Prompt组装与Skill注入.md](07-Prompt组装与Skill注入.md) |
| Part I | ch08 Provider 与 Responses/Realtime API | provider 抽象 + SSE/WS/Reatime | `codex-rs/model-provider/src/provider.rs` | [08-Provider与API模式.md](08-Provider与API模式.md) |
| Part II | ch09 工具系统总览 | tools 注册、schema、handler 分层 | `codex-rs/tools/src/lib.rs` | [09-工具系统总览.md](09-工具系统总览.md) |
| Part II | ch10 命令执行与 unified_exec | 执行通道、shell 解析、安全校验 | `codex-rs/core/src/unified_exec/mod.rs` | [10-命令执行与unified_exec.md](10-命令执行与unified_exec.md) |
| Part II | ch11 apply_patch 工具 | Lark 语法、流式 parser、受约束编辑 | `codex-rs/apply-patch/src/parser.rs` | [11-apply_patch工具.md](11-apply_patch工具.md) |
| Part II | ch12 macOS 与 Linux 沙箱 | Seatbelt + bwrap + landlock | `codex-rs/linux-sandbox/src/bwrap.rs` | [12-macOS与Linux沙箱.md](12-macOS与Linux沙箱.md) |
| Part II | ch13 Windows 沙箱与 WFP | token/ACL/WFP 组合权限模型 | `codex-rs/windows-sandbox-rs/src/lib.rs` | [13-Windows沙箱与WFP防火墙.md](13-Windows沙箱与WFP防火墙.md) |
| Part II | ch14 执行策略 Starlark | execpolicy 规则引擎 | `codex-rs/execpolicy/src/policy.rs` | [14-执行策略Starlark.md](14-执行策略Starlark.md) |
| Part II | ch15 网络代理与策略 | HTTP/SOCKS/MITM + network policy | `codex-rs/network-proxy/src/network_policy.rs` | [15-网络代理与策略.md](15-网络代理与策略.md) |
| Part II | ch16 Hook 与生命周期事件 | PreToolUse/PostToolUse/Compact/Stop | `codex-rs/hooks/src/schema.rs` | [16-Hook与生命周期事件.md](16-Hook与生命周期事件.md) |
| Part II | ch17 Plugin 市场系统 | 市场拉取、远程包、开关与同步 | `codex-rs/core-plugins/src/startup_sync.rs` | [17-Plugin市场系统.md](17-Plugin市场系统.md) |
| Part II | ch18 MCP 双向集成 | MCP client/server 双向桥接 | `codex-rs/codex-mcp/src/connection_manager.rs` | [18-MCP双向集成.md](18-MCP双向集成.md) |
| Part II | ch19 会话与轨迹持久化 | rollout JSONL + state SQLite | `codex-rs/rollout/src/recorder.rs` | [19-会话与轨迹持久化.md](19-会话与轨迹持久化.md) |
| Part II | ch20 记忆系统 | read/write/mcp memory 与注入路径 | `codex-rs/memories/write/src/storage.rs` | [20-记忆系统.md](20-记忆系统.md) |
| Part II | ch21 App-Server JSON-RPC 协议层 | protocol、daemon、client、transport | `codex-rs/app-server-protocol/src/protocol/common.rs` | [21-AppServer协议层.md](21-AppServer协议层.md) |
| Part III | ch22 TUI 渲染管线与 Code Mode V8 | TUI 事件渲染 + V8 runtime | `codex-rs/tui/src/lib.rs` | [22-TUI与CodeMode.md](22-TUI与CodeMode.md) |
| Part III | ch23 Cloud Tasks 与外部 Agent 迁移 | 云任务编排 + 外部会话导入 | `codex-rs/cloud-tasks/src/lib.rs` | [23-CloudTasks与外部Agent迁移.md](23-CloudTasks与外部Agent迁移.md) |
| Part III | ch24 Codex vs Claude Code / Opencode 对比 | 架构与工作流横向比较 | `codex-rs/app-server/src/lib.rs` | [24-Codex与同类对比.md](24-Codex与同类对比.md) |
| Part III | ch25 沙箱与权限模型对比 | 三 OS 沙箱与策略层横评 | `codex-rs/sandboxing/src/manager.rs` | [25-Codex沙箱与权限对比.md](25-Codex沙箱与权限对比.md) |

### 5.2 25 章结构图（导航视角）

<div bgcolor="#ffffff">

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"background": "#ffffff","primaryColor": "#f5f5f5","primaryTextColor": "#000000","primaryBorderColor": "#333333","lineColor": "#444444","secondaryColor": "#f8f8f8","tertiaryColor": "#ffffff","clusterBkg": "#fafafa","clusterBorder": "#888888","edgeLabelBackground": "#ffffff","fontFamily": "Helvetica"}, "themeCSS": "svg { background: #ffffff !important; } .label, .nodeLabel, .edgeLabel, text { fill: #000000 !important; color: #000000 !important; }"}}%%
flowchart TB
  ROOT[OpenAI Codex 源码深度研究]
  ROOT --> P1[Part I 原理与使用]
  ROOT --> P2[Part II 源码分析]
  ROOT --> P3[Part III 对比与演进]

  P1 --> C1[ch01-ch08]
  C1 --> C1A[ch01 全景哲学]
  C1 --> C1B[ch02 多入口]
  C1 --> C1C[ch03 配置]
  C1 --> C1D[ch04/ch05 使用]
  C1 --> C1E[ch06-ch08 Agent/Prompt/Provider]

  P2 --> C2[ch09-ch21]
  C2 --> C2A[工具与执行: ch09-ch11]
  C2 --> C2B[安全与策略: ch12-ch15]
  C2 --> C2C[生态扩展: ch16-ch18]
  C2 --> C2D[持久化与协议: ch19-ch21]

  P3 --> C3[ch22-ch25]
  C3 --> C3A[ch22 TUI + V8]
  C3 --> C3B[ch23 Cloud Tasks]
  C3 --> C3C[ch24 同类架构对比]
  C3 --> C3D[ch25 权限模型对比]
```

</div>

---

## 6. 与同类对比（火爆原因分析）

### 6.1 横向表格：Codex / Claude Code / Opencode / Aider / Goose

> 说明：本表尽量只写“文档可证事实”；无法确认处明确写“未公开”或“未见同等级公开抽象”。

| 维度 | Codex | Claude Code | Opencode | Aider | Goose |
|---|---|---|---|---|---|
| 入口分发方式 | npm 包装 + native Rust 二进制 + SDK（`codex-cli/bin/codex.js:L24-L36`, `sdk/*`） | CLI + VS Code + Desktop + Web + JetBrains（[overview](https://docs.anthropic.com/en/docs/claude-code/overview)） | CLI + desktop + IDE extension（[OpenCode docs](https://open-code.ai/en/docs)） | 终端为主（[Aider docs](https://aider.chat/docs/)） | CLI + desktop（[goose docs](https://goose-docs.ai)） |
| 协议层是否独立 | 有独立 `app-server` / `app-server-protocol` / `transport` crate（`codex-rs/app-server-transport/src/transport/mod.rs:L67-L72`） | 公开文档强调多 surface 共用引擎，但未公开独立协议 crate 形态（overview） | 文档强调产品入口与 agent 配置，未见与 Codex 等价的独立协议层公开分层（OpenCode docs） | 公开文档以 CLI 交互和配置为中心（Aider docs） | 以 MCP 扩展为核心协议能力，未见单独 app-server 分层文档（goose docs） |
| 沙箱矩阵 | 三 OS 分治 + 统一 manager（`sandboxing::Manager` + seatbelt/bwrap/Windows） | 文档突出权限与工具编排，OS 级矩阵细节未集中公开（overview） | 文档强调 permissions/agents，OS 级隔离矩阵未主打（OpenCode docs） | 文档主打终端协作与 git 流程，未突出 OS 级矩阵（Aider docs） | Local-first + 扩展，安全边界更多由本机与扩展配置决定（goose docs） |
| 插件系统 | `core-plugins` + `startup_sync` + `codex-mcp`（`codex-rs/core-plugins/src/manager.rs:L194-L224`） | MCP + skills + hooks（overview） | agents + MCP + project `AGENTS.md`（OpenCode docs, agents docs） | 命令/配置生态强，插件市场型分发不是核心叙事（Aider docs） | extensions（MCP servers）是一等能力（goose docs） |
| 持久化方案 | rollout JSONL + state SQLite（`codex-rs/rollout/src/recorder.rs:L758-L814`, `codex-rs/state/src/runtime/threads.rs:L494-L584`） | 文档强调跨 surface 会话连续与调度（overview） | 文档有会话与分享链路说明（OpenCode docs） | 以 git 历史 + 对话流程为主（Aider docs） | Sessions + Memory 为核心概念（goose docs） |
| 商业策略 | 本地 CLI + Cloud Tasks + OpenAI API/产品协同（`codex-rs/cloud-tasks/src/lib.rs:L157-L182`；[introducing codex](https://openai.com/index/introducing-codex/)） | 订阅/账户体系驱动，多 surface 服务统一（overview） | 开源 + 多 provider + BYO key（OpenCode docs） | 开源工具 + 多模型连接（Aider docs） | 开源 + 多 provider + extension 生态（goose docs） |
| 开源治理 | 源码开放但贡献 invitation-only + CLA（`docs/contributing.md:L3`, `L80-L92`） | 文档开放、产品运营主导（overview） | 开源社区模式（OpenCode docs） | 开源社区模式（Aider docs） | 开源（页面写明 community / Apache 2.0）（goose docs） |

### 6.2 事实后的判断

1. **Codex 的差异点不是“有 CLI”，而是“把协议层显式产品化”。**  
   同类大多强调入口和体验，Codex 额外强调 `app-server + protocol + transport` 的内聚边界，这更像平台 SDK 化路径。

2. **Codex 的安全叙事更工程化。**  
   通过三 OS 不同后端 + 统一调度器，它把安全问题从“抽象权限开关”推进到“系统调用和网络控制层”。

3. **Codex 的治理模式是优势和限制并存。**  
   主干演进一致性强，但外部贡献灵活性相对低。对研究者友好，对社区共建型贡献者不一定友好。

4. **2025-2026 的“火爆”更像综合效应，而不是单指标领先。**  
   分发体验、协议分层、沙箱可信、插件/MCP 生态、云端任务联动一起作用，单点解释力都不够。

---

## 7. 沙箱矩阵（独立小节）

### 7.1 统一调度与多后端执行

Codex 的沙箱不走“所有平台同一实现”，而是“统一调度 + 后端差异化”：

- `SandboxManager` 负责抽象层分发（`codex-rs/sandboxing/src/manager.rs:L23-L28`）。
- macOS：Seatbelt profile，依赖 `sandbox-exec`（`codex-rs/sandboxing/src/seatbelt.rs:L25-L29`）。
- Linux：`bwrap` 管文件系统边界，Landlock/seccomp 管进程/网络能力（`codex-rs/linux-sandbox/src/bwrap.rs:L58-L67`, `codex-rs/linux-sandbox/src/landlock.rs:L119-L169`）。
- Windows：token/ACL/WFP 组合（`codex-rs/windows-sandbox-rs/src/lib.rs:L255-L279`）。
- 执行前后还叠加 `execpolicy` 与 `network policy` 决策（`codex-rs/execpolicy/src/parser.rs:L57-L61`, `codex-rs/network-proxy/src/network_policy.rs:L43-L45`）。

### 7.2 关系图：`sandboxing::Manager` + `execpolicy` + `network-policy`

<div bgcolor="#ffffff">

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"background": "#ffffff","primaryColor": "#f5f5f5","primaryTextColor": "#000000","primaryBorderColor": "#333333","lineColor": "#444444","secondaryColor": "#f8f8f8","tertiaryColor": "#ffffff","edgeLabelBackground": "#ffffff","fontFamily": "Helvetica"}, "themeCSS": "svg { background: #ffffff !important; } .label, .nodeLabel, .edgeLabel, text { fill: #000000 !important; color: #000000 !important; }"}}%%
flowchart TD
  REQ[Tool/Command Request]
  EXECP[execpolicy(Starlark)]
  NP[network policy<br/>Deny/Ask]
  MGR[sandboxing::Manager]
  MAC[macOS Seatbelt]
  LNX[Linux bwrap + Landlock]
  WIN[Windows Token + ACL + WFP]
  RUN[Command Running]
  BLOCK[Blocked / Need Approval]

  REQ --> EXECP
  EXECP -->|allow| NP
  EXECP -->|deny| BLOCK
  NP -->|allow| MGR
  NP -->|ask or deny| BLOCK
  MGR --> MAC
  MGR --> LNX
  MGR --> WIN
  MAC --> RUN
  LNX --> RUN
  WIN --> RUN
```

</div>

### 7.3 判断：为什么这套结构在工程上更“稳”

- 统一调度器降低了“入口多样化”造成的安全行为漂移；
- 各平台实现可独立迭代，不需要为了一致性牺牲系统级能力；
- 策略层（exec/network）和执行层（sandbox backend）分离，便于企业场景做合规审计。

代价也很明确：测试矩阵和故障排查复杂度显著上升，尤其在 Windows 路径（这也是社区 issue 高发区域之一，见 `全网调研-社区认知地图.md:L30-L34`）。

---

## 8. App-Server 演进：从 stdio 到 IDE 集成

### 8.1 从“单通道 server”到“多传输协议中枢”

从源码和文档信号看，Codex 的演进路径是清晰的：

1. **早期形态：`mcp-server`（stdio JSON-RPC）**  
   `run_main` 直接从 stdin 读 JSON-RPC、处理后写 stdout（`codex-rs/mcp-server/src/lib.rs:L125-L142`, `L174-L195`）。

2. **中期形态：`app-server`（协议抽象 + 多传输）**  
   transport 从一条 stdio 拓展到 `Stdio/UnixSocket/WebSocket`（`codex-rs/app-server-transport/src/transport/mod.rs:L67-L72`），并通过 protocol crate 明确方法与版本边界（`codex-rs/app-server-protocol/src/protocol/common.rs:L437-L465`）。

3. **驻留形态：`app-server-daemon`（生命周期管理）**  
   支持 `start/restart/stop` 与 remote control（`codex-rs/app-server-daemon/src/lib.rs:L276-L289`, `L294-L357`, `L200-L231`），将一次性 CLI 会话升级为可复用后台服务。

4. **多端接入：CLI/TUI/SDK/IDE 共享中枢**  
   `app-server-client` 明确自己是 in-process facade，服务 TUI/exec 等 surface（`codex-rs/app-server-client/src/lib.rs:L1-L16`）。

### 8.2 v1 与 v2：兼容共存，但新增面向 v2

- `app-server-protocol` 仍保留 v1 allowlist（`codex-rs/app-server-protocol/src/export.rs:L41-L50`），说明兼容负担仍在。
- 同时 AGENTS 明确新 API 面向 v2，避免继续扩 v1（`AGENTS.md:L186-L190`）。
- v2 模块化拆分明显更完整（`codex-rs/app-server-protocol/src/protocol/v2/mod.rs:L1-L54`）。

这三条合起来是典型“迁移中态”：  
**不是立刻废弃 v1，而是通过治理规则和新功能投放方向，逐步把生态重心推向 v2**。

### 8.3 演进图（stdio → app-server → daemon → 远程集成）

<div bgcolor="#ffffff">

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"background": "#ffffff","primaryColor": "#f5f5f5","primaryTextColor": "#000000","primaryBorderColor": "#333333","lineColor": "#444444","secondaryColor": "#f8f8f8","tertiaryColor": "#ffffff","edgeLabelBackground": "#ffffff","fontFamily": "Helvetica"}, "themeCSS": "svg { background: #ffffff !important; } .label, .nodeLabel, .edgeLabel, text { fill: #000000 !important; color: #000000 !important; }"}}%%
flowchart LR
  A[mcp-server<br/>stdio JSON-RPC] --> B[app-server<br/>protocol + transport]
  B --> C[app-server-daemon<br/>lifecycle + remote control]
  C --> D[Remote IDE / SDK / multi-surface]
  B --> E[v1 compatibility]
  B --> F[v2 active evolution]
  E --> G[allowlist + legacy methods]
  F --> H[new API surface]
```

</div>

---

## 9. 火爆原因（深度分析）

这一节回答“为什么是 Codex”，只保留 2025-2026 可证据化结论。

### 9.1 原因一：上手摩擦低，入口覆盖广

**事实**：npm 一键安装后即落地 native 二进制（`codex-cli/package.json:L2-L8`, `codex-cli/bin/codex.js:L24-L36`, `L184-L187`）；同时支持 Rust CLI、TS/Python SDK（`sdk/typescript/README.md:L5-L6`, `sdk/python/README.md:L3-L4`）。  
**影响**：开发者无需先理解架构即可启动，扩散成本低（`全网调研-社区认知地图.md:L117-L129`）。

### 9.2 原因二：把 harness 做成“可复用协议中枢”

**事实**：`app-server` + `app-server-protocol` + `transport` 三层分离（`codex-rs/app-server/src/lib.rs:L420-L433`, `codex-rs/app-server-transport/src/transport/mod.rs:L67-L72`, `codex-rs/app-server-protocol/src/protocol/common.rs:L437-L465`）。  
**影响**：价值不止于终端交互，而是可被 IDE/SDK/远程形态复用（`全网调研-社区认知地图.md:L156-L160`）。

### 9.3 原因三：安全能力从策略文本下沉到系统实现

**事实**：三 OS 后端 + 统一 manager（`codex-rs/sandboxing/src/manager.rs:L23-L28`, `codex-rs/sandboxing/src/seatbelt.rs:L25-L29`, `codex-rs/linux-sandbox/src/bwrap.rs:L58-L67`, `codex-rs/windows-sandbox-rs/src/lib.rs:L255-L279`）。  
**影响**：权限边界更可解释，企业评估门槛更低。

### 9.4 原因四：MCP 与 Plugin 形成网络效应

**事实**：`codex-mcp` 管连接生命周期，`core-plugins` 管安装与同步（`codex-rs/codex-mcp/src/connection_manager.rs:L71-L111`, `codex-rs/core-plugins/src/manager.rs:L194-L224`, `codex-rs/core-plugins/src/startup_sync.rs:L66-L99`）。  
**影响**：能力增长不只依赖主仓迭代。社区对 MCP 的“table stakes”判断很早出现（`全网调研-社区认知地图.md:L49-L53`, `L131-L134`）。

### 9.5 原因五：长会话可恢复，工程可追溯

**事实**：轨迹写 JSONL，状态写 SQLite；关键事件按“必须送达”处理（`codex-rs/rollout/src/recorder.rs:L758-L814`, `codex-rs/state/src/runtime/threads.rs:L494-L584`, `codex-rs/app-server-client/src/lib.rs:L151-L186`）。  
**影响**：更接近“可运行系统”而非一次性聊天工具。

### 9.6 原因六：开源可见度与官方节奏并存

**事实**：仓库可见度高，同时治理是 invitation-only + CLA（`docs/contributing.md:L3`, `L80-L92`）。  
**影响**：社区传播强，但主线演进不易分叉拉散。

### 9.7 原因七：本地工作流与云端任务形成双模闭环

**事实**：本地 CLI/TUI 与 `cloud-tasks` 共存（`codex-rs/cli/src/main.rs:L507-L560`, `codex-rs/cloud-tasks/src/lib.rs:L157-L182`）；官方也强调本地与云端协同（`全网调研-社区认知地图.md:L72-L75`）。  
**影响**：团队可按任务类型选择执行面，降低单模式失效风险。

### 9.8 因果图：火爆不是单点优势

<div bgcolor="#ffffff">

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"background": "#ffffff","primaryColor": "#f5f5f5","primaryTextColor": "#000000","primaryBorderColor": "#333333","lineColor": "#444444","secondaryColor": "#f8f8f8","tertiaryColor": "#ffffff","edgeLabelBackground": "#ffffff","fontFamily": "Helvetica"}, "themeCSS": "svg { background: #ffffff !important; } .label, .nodeLabel, .edgeLabel, text { fill: #000000 !important; color: #000000 !important; }"}}%%
flowchart LR
  A[低摩擦分发<br/>npm + native + SDK]
  B[协议中枢化<br/>app-server]
  C[沙箱矩阵<br/>三 OS]
  D[生态接口<br/>MCP + Plugin]
  E[持久化与恢复<br/>JSONL + SQLite]
  F[双模产品线<br/>local + cloud]
  G[社区可见度高]
  H[采用扩散加速]
  I[头部 harness 地位]

  A --> H
  B --> H
  C --> H
  D --> H
  E --> H
  F --> H
  G --> H
  H --> I
```

</div>

---

## 10. 设计权衡（不藏拙）

### 10.1 哪些设计是 OpenAI 更有资本做的

1. **一手 API 协同**：workspace 内直接出现 `responses-api-proxy`、`realtime-webrtc`、`cloud-tasks`（`codex-rs/Cargo.toml:L26-L28`, `L70`, `L74`）。
2. **多 surface 一致性投入**：CLI、TUI、SDK、daemon、remote control 一起演化（`codex-rs/cli/src/main.rs:L437-L560`, `codex-rs/app-server-daemon/src/lib.rs:L276-L357`）。
3. **协议迁移可控性**：v1/v2 并存下，新增能力被明确约束到 v2（`AGENTS.md:L186-L190`, `codex-rs/app-server-protocol/src/export.rs:L41-L50`）。

### 10.2 哪些设计有明显代价

1. **Rust 维护成本**：113 crate 协作复杂度高（`codex-rs/Cargo.toml:L2-L116`）。
2. **跨平台安全测试成本**：三套后端导致测试矩阵扩大，Windows 边界问题密集（`全网调研-社区认知地图.md:L30-L34`, `L146-L149`）。
3. **治理门槛成本**：invitation-only + CLA 提升主线控制，同时降低外部贡献即时性（`docs/contributing.md:L3`, `L80-L92`）。

### 10.3 哪些方向仍在演进

1. **协议层重心迁移**：v2 持续扩展，v1 处于兼容期（`codex-rs/app-server-protocol/src/protocol/v2/mod.rs:L1-L54`, `AGENTS.md:L186-L190`）。
2. **Code Mode 运行时成熟度**：`v8::IsolateHandle` + 独立 runtime/session 已在主线出现（`codex-rs/code-mode/src/service.rs:L57-L59`, `L357`）。
3. **插件链路稳定性**：市场发现与刷新一致性仍在打磨（`全网调研-社区认知地图.md:L51-L55`）。

---

## 11. 阅读路径建议

### 11.1 初学者

- 先读：第 4 / 7 / 9 章；再回看第 1 / 2 章建立坐标。

### 11.2 工程师

- 重点：第 6 / 10 / 12-15 / 18-21 章。
- 方法：章节与源码路径并行阅读（`scripts/chapters.yaml:L119-L265`）。

### 11.3 平台研究者

- 重点：第 1 / 17 / 22 / 24 / 25 章。
- 方法：先读本总纲第 6 / 8 / 10 节，再进入对比与演进章节。

---

## 12. 全文 Mermaid 图清单（≥ 6 张）

本文共 **6 张** Mermaid 图，均使用 `neutral/base` + 白底容器：

1. 总体架构图（第 3 节）  
2. 一次命令时序图（第 4 节）  
3. 25 章映射图（第 5 节）  
4. 沙箱矩阵图（第 7 节）  
5. App-Server 演进图（第 8 节）  
6. 火爆原因因果图（第 9 节）

一句话结论：Codex 把“编码 agent”从模型调用问题，工程化为跨入口、跨协议、跨沙箱、可持久化、可扩展的运行系统。

