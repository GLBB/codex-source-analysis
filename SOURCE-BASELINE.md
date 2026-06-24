# 源码基线与更新规则

## 基线

本轮重写直接基于：

```text
repository: openai/codex
remote:     upstream = https://github.com/openai/codex.git
branch:     upstream/main
commit:     283bc4cf011047314b4804c0f1ccd06e4f6a95c5
date:       2026-06-24
```

当前分支在拉取上游后，将 17 个本地学习资料提交完整 rebase 到该基线。

## 2026-06-24 需要特别校准的变化

| 机制 | 当前事实 | 关键入口 |
| --- | --- | --- |
| Context window | `ContextWindowTokenStatus` 同时记录完整活动上下文、auto-compact scope、完整窗口余量和 prefix baseline。 | `codex-rs/core/src/session/context_window.rs` |
| Session budget | rollout budget 耗尽现在向外暴露为 `SessionBudgetExceeded`，避免把整段 session 预算误解成单个 rollout 文件限制。 | `codex-rs/core/src/session/rollout_budget.rs` |
| MCP tools | 模型/provider 支持 namespace search tool 时，MCP 工具默认 deferred，通过 `tool_search` 按需加载；旧版“超过 100 个才 deferred”已经过期。 | `codex-rs/core/src/mcp_tool_exposure.rs`、`tools/spec_plan.rs` |
| Multi-Agent | v2 协作工具可进入配置的 namespace，默认配置不应再假设所有工具都以顶层函数名出现。 | `codex-rs/core/src/tools/spec_plan.rs`、`tools/handlers/multi_agents_v2/` |
| App Server | 持久化 item 分页方法是实验性的 `thread/items/list`，可选 `turnId`；旧方法名不再是当前契约。 | `codex-rs/app-server-protocol/src/protocol/common.rs`、`app-server/README.md` |
| Code Mode | host 边界新增显式版本与 capability 握手：`connection/hello`、`connection/ready`、`connection/rejected`。 | `codex-rs/code-mode-protocol/src/host/` |
| Path / Plugin | selected plugin roots 与 executor skills 继续迁移为 URI-native / environment-native，不能默认把远程路径转成本机 `PathBuf`。 | `codex-rs/utils/path-uri/`、`core-plugins/`、`core-skills/` |
| Environment | turn 使用当前 step/world-state environment，工具、图片和审批必须绑定正确执行环境。 | `codex-rs/core/src/environment_selection.rs`、`session/turn.rs` |

## 证据等级

1. 当前源码、测试、schema fixture。
2. 同一路径的 Git 历史与重构提交。
3. OpenAI 官方 README、协议文档和产品文档。
4. issue、社区文章和横向评测。
5. 设计动因推断。

第 5 类必须使用“可能”“推测”“从约束看”等措辞，不得伪装成作者意图。

## 更新流程

```bash
git fetch upstream main
git rebase upstream/main

git log --oneline <old-baseline>..upstream/main -- codex-rs
git diff --stat <old-baseline>..upstream/main -- codex-rs

rg -n "关键符号" codex-rs
npm run validate:mermaid
```

更新时优先检查：

- `core/src/session/turn.rs` 和 context/compaction；
- `core/src/tools/`、`codex-mcp/` 和 plugins/skills；
- `app-server-protocol` 与生成 schema；
- rollout reconstruction、thread store 和 state migrations；
- sandbox/permission/exec policy；
- TUI snapshot 和跨平台测试。

不要只改提交号。若符号、状态机或协议名已经变化，应直接重写相关段落。
