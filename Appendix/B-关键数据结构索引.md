# 附录 B — 关键数据结构索引

> 本附录索引 Codex 源码中横跨多章使用的核心 `struct` / `enum` / `trait` / `JSON-RPC` 消息类型，方便阅读时跳转。所有路径均相对于 `/Users/hexiaonan/workspace/formless/refer/codex`。

## B.1 Session / Turn

| 名称 | 路径 | 用途 |
|------|------|------|
| `Session` | `codex-rs/core/src/session/session.rs` | 单次会话生命周期容器 |
| `Turn` | `codex-rs/core/src/session/turn.rs` | 单轮 prompt → tool → response 状态机 |
| `ThreadManager` | `codex-rs/core/src/thread_manager.rs` | 线程创建 / fork / resume |
| `TurnContext` | `codex-rs/core/src/session/mod.rs` | turn 内共享的引用与状态 |

> 本附录在生成 25 章正文后自动补全（参见 Phase 6）；当前仅给出索引骨架。

## B.2 Tools

| 名称 | 路径 | 用途 |
|------|------|------|
| `ToolSpec` | `codex-rs/tools/src/tool_spec.rs` | 工具 JSON Schema 定义 |
| `ToolDefinition` | `codex-rs/tools/src/tool_definition.rs` | 运行时工具注册项 |
| `ToolHandler` (trait) | `codex-rs/core/src/tools/mod.rs` | 工具实现统一接口 |

## B.3 Protocol（App-Server）

| 名称 | 路径 | 用途 |
|------|------|------|
| `Item` | `codex-rs/protocol/src/items.rs` | 会话项（消息、工具调用、事件） |
| `Permission` | `codex-rs/protocol/src/permissions.rs` | 权限/审批模型 |
| `ModelV2` | `codex-rs/protocol/src/models.rs` | 模型描述 |
| `JsonRpcRequest` / `Notification` | `codex-rs/app-server-protocol/src/protocol/common.rs` | v1/v2 通用 JSON-RPC 信封 |

## B.4 Sandbox / ExecPolicy

| 名称 | 路径 | 用途 |
|------|------|------|
| `SandboxManager` | `codex-rs/sandboxing/src/manager.rs` | 跨平台沙箱总入口 |
| `SeatbeltPolicy` | `codex-rs/sandboxing/src/seatbelt.rs` | macOS Seatbelt 规则 |
| `BwrapArgs` | `codex-rs/linux-sandbox/src/bwrap.rs` | Linux bwrap 调用参数 |
| `Policy` (Starlark) | `codex-rs/execpolicy/src/policy.rs` | 执行策略规则 |

## B.5 Persistence

| 名称 | 路径 | 用途 |
|------|------|------|
| `RolloutRecorder` | `codex-rs/rollout/src/recorder.rs` | 会话 JSONL 持久化 |
| `StateDb` | `codex-rs/state/src/log_db.rs` | SQLite 状态库 |
| `MemoryRecord` | `codex-rs/memories/write/src/storage.rs` | 记忆条目结构 |

## B.6 Plugins / Skills / Hooks

| 名称 | 路径 | 用途 |
|------|------|------|
| `PluginManifest` | `codex-rs/core-plugins/src/manager.rs` | 插件元数据 |
| `SkillRecord` | `codex-rs/core-skills/src/loader.rs` | Skill 注册项 |
| `HookEvent` | `codex-rs/hooks/src/schema.rs` | 钩子事件枚举 |

> 详细字段、字节大小、序列化格式见对应章节正文。
