# 附录 D：最新源码研究路线图

本附录记录直接以 `/home/goulei/code/codex` 当前 checkout 作为最新研究对象时的工作方法。后续更新以当前源码为事实来源，持续补充机制、算法、状态机和验证方法。

## 目标

后续完善优先服务三个目标：

1. **研究当前源码**：以当前 checkout 的文件、函数、测试和运行行为作为最新事实来源。
2. **补机制级算法**：对核心章节补充状态变量、触发条件、伪代码、异常路径和验证方法。
3. **增强可复核性**：每个重要判断尽量提供稳定源码锚点、当前行号锚点和验证命令。

## 源码优先原则

如果本研究已有章节与当前 `codex-rs` 最新代码不一致，处理顺序如下：

1. **以当前源码为准**：当前 checkout 的实现、测试、协议类型和命令输出是最高优先级证据。
2. **删除或改写不一致描述**：不要为了保留旧章节结构而维持过期描述；没有演进价值的旧描述可以直接删除。
3. **必要时保留历史说明**：如果旧描述有演进价值，可以标注为“历史实现/旧版本行为”，但不能继续当作当前事实。
4. **用命令证明差异**：优先给出 `rg`、`git log -- <path>`、测试或 schema 校验作为证据。

## 证据分层

后续补充内容按证据强度分层：

| 证据层 | 形式 | 用途 |
|---|---|---|
| 源码事实 | 当前 checkout 的文件、函数、struct、enum、测试 | 证明“现在代码怎么做”。 |
| Git 历史 | `git log`、重构提交、目录迁移 | 解释“为什么演进成这样”。 |
| 官方资料 | OpenAI 文档、README、协议说明 | 解释公开产品语义和接口契约。 |
| 社区材料 | issue、讨论、第三方评测 | 识别误解、争议和使用痛点。 |
| 推测判断 | 基于约束的设计动因解释 | 必须显式标注“可能/推测/不排除”。 |

## 与现有章节的补强映射

| 优先级 | 章节 | 补强方向 |
|---|---|---|
| P0 | 第 06 章 Agent 核心循环 | 校准 `run_turn` / `run_sampling_request` / `try_run_sampling_request` 当前行号；补 `can_drain_pending_input`、`stop_hook_active`、`active_item` 等状态变量。 |
| P0 | 第 10 章 命令执行与 unified_exec | 对齐当前 legacy exec 与 unified exec 的边界；补 stdout/stderr 双通道、output cap、timeout/cancel 终止策略。 |
| P0 | 第 11 章 apply_patch 工具 | 补 `AppliedPatchDelta`、`exact` 语义、PathUri 与 `ExecutorFileSystem` 关系。 |
| P0 | 第 19 章 会话与轨迹持久化 | 补 rollout reconstruction、replacement history、rollback、forward replay suffix。 |
| P1 | 第 07 章 Prompt 组装与 Skill 注入 | 补 tool exposure、skill/plugin injection、connector selection 的当前机制。 |
| P1 | 第 18 章 MCP 双向集成 | 补 direct/deferred MCP tool exposure、`tool_search` 索引、parallel/readOnlyHint。 |
| P1 | 第 21 章 App-Server JSON-RPC 协议层 | 补 initialize gate、experimental API、request serialization queue、outbound initialized。 |
| P1 | 第 22 章 TUI 与 Code Mode | 补 streaming cell、active cell、markdown consolidation、宽度重渲染。 |
| P2 | 第 12-15 章沙箱/权限专题 | 按平台补 OS enforcement 与 approval cache 的边界。 |
| P2 | 第 23-25 章对比专题 | 用当前 Codex 机制更新横向对比，避免只比较产品口号。 |

## 每章增量模板

每次补一个章节时，建议添加一个“最新源码研究补充”或“机制级补充”小节，模板如下：

```text
### 最新源码研究补充（YYYY-MM-DD）

当前源码锚点：
- 文件/函数/行号
- 关键 struct/enum
- 相关测试

状态变量：
- 变量名：语义、生命周期、边界条件

伪代码：
- 用当前源码流程重写一次主算法

异常路径：
- retry / cancellation / timeout / approval / compaction / rollback

验证命令：
- rg / git log / just test -p ...
```

## 当前第一批补丁

已开始的第一批补丁：

- 第 06 章新增“最新源码研究补充（2026-06-24）”，直接以当前源码解释 `run_turn`、`run_sampling_request`、`try_run_sampling_request`，并补充 turn 状态变量与伪代码。

## 推荐验证命令

```bash
# 查看当前章节结构
find learning/codex/codex-source-analysis -maxdepth 2 -type f -name '*.md' | sort

# 校验 Mermaid
cd learning/codex/codex-source-analysis
npm install
npm run validate:mermaid

# 校准源码锚点
rg -n "pub\\(crate\\) async fn run_turn|async fn run_sampling_request|async fn try_run_sampling_request" ../../codex-rs/core/src/session/turn.rs
```
