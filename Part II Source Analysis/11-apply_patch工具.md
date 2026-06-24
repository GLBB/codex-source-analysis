# 11｜apply_patch：从受约束文本到文件系统变更

> 源码基线：`upstream/main@283bc4cf011047314b4804c0f1ccd06e4f6a95c5`（2026-06-24）。

`apply_patch` 不是 `git apply` 的别名，也不是任意 unified diff。它是一套为模型生成、预览、审批和跨环境执行设计的 freeform 工具协议。

## 1. 语法边界

工具规格使用 Lark grammar，基本结构为：

```text
*** Begin Patch
[*** Environment ID: <id>]
<one or more hunks>
*** End Patch
```

Hunk 只有三种：

- `*** Add File: path`
- `*** Delete File: path`
- `*** Update File: path`

Update 还可带 `*** Move to: path`，正文由 context、删除行和新增行组成。

明确的 grammar 让模型输出在进入执行前就能被验证，并能返回带行号的解析错误。

## 2. 完整链路

```mermaid
flowchart LR
    M[Model patch text] --> S[Freeform ToolSpec]
    S --> P[Streaming parser]
    P --> V[Final parser / verification]
    V --> A[Safety assessment]
    A --> R[ApplyPatchRuntime]
    R --> F[ExecutorFileSystem]
    F --> D[AppliedPatchDelta]
    D --> E[Events / TurnDiffTracker]
```

流式 parser 用于预览正在生成的 patch；最终执行仍会完整解析和验证。预览成功不表示变更已经提交。

## 3. 为什么不是普通 shell

在写入前，系统可以知道：

- 哪些路径会新增、删除、修改或移动；
- 目标属于哪个执行环境；
- 路径是否落在可写范围；
- 是否需要审批；
- 变更摘要应如何展示。

如果把 patch 作为任意 shell 文本执行，这些信息只能靠不可靠的命令字符串分析。

## 4. Environment ID

当线程连接多个执行环境时，grammar 可以允许 `*** Environment ID:`。它决定 patch 应交给哪个环境的文件系统。

这不是注释，而是路由字段。缺失、重复或未知 ID 都应在执行前失败，不能默认把远程 patch 落到本地。

## 5. `ExecutorFileSystem`

`codex-apply-patch` 不直接依赖本地 `std::fs`，而面向 `ExecutorFileSystem`：

- 本地文件系统；
- sandboxed 文件系统；
- remote exec-server 文件系统。

路径使用 `PathUri` 解析，使 Windows、本地 Unix 与远程环境的路径语义不会被一条裸字符串混用。

## 6. Hunk 匹配

Update hunk 需要在原文件中寻找上下文。匹配会逐步降低严格度，例如：

1. 精确匹配；
2. 忽略尾部空白；
3. 忽略首尾空白。

这种容错用于吸收模型生成中的轻微空白差异，但不会把找不到上下文的 patch 强行应用。歧义或缺失上下文会返回错误，促使模型重新读取文件并生成更精确的 patch。

## 7. 安全评估与审批

Core 侧先调用 `assess_patch_safety`，结合：

- approval policy；
- permission profile；
- file-system sandbox policy；
- cwd 与目标路径；
- Windows sandbox level。

结果可能是自动执行、请求审批或拒绝。审批发生在明确目标路径之后，因此提示可以展示实际变更范围。

## 8. 写入顺序与部分失败

多文件 patch 不是底层文件系统提供的原子事务。前几个文件可能已写入，后续写入才失败。因此结果使用 `AppliedPatchDelta` 记录：

- 已确认提交的 changes；
- `exact`：这份 delta 是否完整可信。

写操作可能先截断文件再返回磁盘错误，所以写失败会把 `exact` 置为 `false`。此时系统不能声称“所有变化都已精确掌握”。

```text
成功
  → exact delta

失败且已知无副作用
  → exact prefix delta

失败且可能已有未知副作用
  → inexact delta
```

`TurnDiffTracker` 只有在 delta 足够精确时才能可靠增量更新；否则需要重新读取或采取更保守的恢复方式。

## 9. 流式事件

启用对应 feature 时，`StreamingPatchParser` 会消费工具参数 delta，并以节流方式产生 patch 预览事件。其用途是：

- UI 提前展示计划修改的文件；
- 让长 patch 不显得完全无响应；
- 不把每个 token 都变成一次渲染。

最终 `PatchApply` 事件才代表执行结果。预览事件必须与提交事件在语义上分开。

## 10. Shell 中的 apply_patch 拦截

兼容路径可以识别：

- `apply_patch <body>`；
- shell heredoc 形式；
- 可选的 `cd ... && apply_patch ...`。

只有完整解析为合法调用后才走专用 patch runtime。仅仅字符串里出现 `apply_patch` 不足以拦截，否则普通脚本可能被误判。

## 11. 失败恢复

常见失败及处理：

| 失败 | 建议恢复 |
| --- | --- |
| grammar / hunk 错误 | 根据行号重写 patch |
| context 不匹配 | 重新读取目标文件 |
| 路径越界 | 请求合适权限或改目标 |
| 审批拒绝 | 停止，不绕过 |
| 环境不存在 | 重新选择 environment |
| inexact delta | 重新读取受影响文件确认状态 |

## 12. 源码阅读路线

```bash
sed -n '1,120p' codex-rs/core/src/tools/handlers/apply_patch.lark
rg -n "StreamingPatchParser|Environment ID" codex-rs/apply-patch/src
rg -n "AppliedPatchDelta|delta.exact|apply_hunks_to_files" codex-rs/apply-patch/src/lib.rs
rg -n "assess_patch_safety" codex-rs/core/src
rg -n "ExecutorFileSystem" codex-rs/apply-patch codex-rs/exec-server
rg -n "TurnDiffTracker|track_delta" codex-rs/core/src
```

`apply_patch` 的本质是：

> 用可解析协议限制模型输出，用权限和文件系统抽象约束执行，再用精确度可表达的 delta 诚实记录实际发生的变更。
