# 附录 B｜关键数据结构索引

> 源码基线：`upstream/main@283bc4cf011047314b4804c0f1ccd06e4f6a95c5`（2026-06-24）。路径均相对于 Codex 仓库根目录。

## B.1 Session / Turn / Context

| 名称 | 路径 | 用途 |
| --- | --- | --- |
| `ThreadManager` | `codex-rs/core/src/thread_manager.rs` | 创建、恢复、分叉和管理线程 |
| `CodexThread` | `codex-rs/core/src/codex_thread.rs` | 线程级外部控制句柄 |
| `Session` | `codex-rs/core/src/session/session.rs` | 长生命周期会话状态 |
| `TurnContext` | `codex-rs/core/src/session/turn_context.rs` | 单次采样步骤共享设置 |
| `ContextManager` | `codex-rs/core/src/context_manager/` | 模型可见历史与 generation |
| `ContextualUserFragment` | `codex-rs/context-fragments/` | 类型化上下文片段 |

## B.2 Tools

| 名称 | 路径 | 用途 |
| --- | --- | --- |
| `ToolSpec` | `codex-rs/tools/src/tool_spec.rs` | 模型可见工具契约 |
| `ToolExposure` | `codex-rs/tools/src/tool_executor.rs` | Direct/Deferred/Hidden |
| `ToolRouter` | `codex-rs/core/src/tools/router.rs` | Responses item → ToolCall |
| `ToolRegistry` | `codex-rs/core/src/tools/registry.rs` | Executor 注册与治理派发 |
| `ToolCallRuntime` | `codex-rs/core/src/tools/parallel.rs` | 并发、取消、结果回写 |

## B.3 Model / Provider

| 名称 | 路径 | 用途 |
| --- | --- | --- |
| `ModelProvider` | `codex-rs/model-provider/src/provider.rs` | 认证、能力、模型与 URL |
| `ModelProviderInfo` | `codex-rs/model-provider-info/` | Provider 静态配置 |
| `ModelClient` | `codex-rs/core/src/client.rs` | Session 级模型客户端 |
| `ModelClientSession` | `codex-rs/core/src/client.rs` | Turn 级 WS/HTTP 状态 |
| `WireApi` | `codex-rs/model-provider-info/` | 当前 Responses wire API |

## B.4 Security / Execution

| 名称 | 路径 | 用途 |
| --- | --- | --- |
| `PermissionProfile` | `codex-rs/protocol/src/permissions.rs` | 文件系统与网络权限 |
| `SandboxManager` | `codex-rs/sandboxing/src/manager.rs` | 平台沙箱选择与转换 |
| `Policy` / `Decision` | `codex-rs/execpolicy/src/` | Starlark 执行规则 |
| `UnifiedExecProcessManager` | `codex-rs/core/src/unified_exec/` | PTY/后台进程生命周期 |
| `AppliedPatchDelta` | `codex-rs/apply-patch/src/lib.rs` | 已提交 patch 变化与精确度 |
| `PathUri` | `codex-rs/utils/path-uri/` | 跨环境路径身份 |

## B.5 App Server

| 名称 | 路径 | 用途 |
| --- | --- | --- |
| `ClientRequest` | `codex-rs/app-server-protocol/src/protocol/common.rs` | client→server RPC |
| `ServerRequest` | 同上 | server→client 同步请求 |
| `ServerNotification` | 同上 | server→client 事件 |
| `ClientRequestSerializationScope` | 同上 | 请求冲突域 |
| `AppServerTransport` | `codex-rs/app-server-transport/src/transport/mod.rs` | stdio/Unix/WS/off |
| `AppServerClient` | `codex-rs/app-server-client/src/lib.rs` | InProcess/Remote facade |

## B.6 Persistence

| 名称 | 路径 | 用途 |
| --- | --- | --- |
| `ThreadStore` | `codex-rs/thread-store/src/store.rs` | 存储无关线程接口 |
| `LiveThread` | `codex-rs/thread-store/src/live_thread.rs` | 活动线程写入 |
| `LocalThreadStore` | `codex-rs/thread-store/src/local/` | 本地 rollout 实现 |
| `StateRuntime` | `codex-rs/state/src/runtime.rs` | state/log/goal/memory SQLite |
| `Stage1Output` | `codex-rs/state/src/model/` | 单 rollout 记忆提取结果 |

## B.7 Extensions

| 名称 | 路径 | 用途 |
| --- | --- | --- |
| `PluginsManager` | `codex-rs/core-plugins/src/manager.rs` | 市场、安装、同步和加载 |
| `PluginLoadOutcome` | `codex-rs/plugin/src/load_outcome.rs` | Plugin 能力聚合结果 |
| `McpConnectionManager` | `codex-rs/codex-mcp/src/connection_manager.rs` | MCP 多 Server 聚合 |
| `HookEventsToml` | `codex-rs/config/src/hook_config.rs` | 10 类 Hook 配置 |
| `HookSource` | `codex-rs/protocol/src/protocol.rs` | Hook 来源 |

## B.8 Collaboration

| 名称 | 路径 | 用途 |
| --- | --- | --- |
| `GuardianApprovalRequest` | `codex-rs/core/src/guardian/approval_request.rs` | 安全审查输入 |
| Review task | `codex-rs/core/src/tasks/review.rs` | 代码审查执行 |
| Goal runtime | `codex-rs/ext/goal/src/runtime.rs` | 长目标 continuation |
| Goal store | `codex-rs/state/src/runtime/` | Goal 状态与 accounting |

具体字段以源码为准；本索引只提供稳定阅读入口。
