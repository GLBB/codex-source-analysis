# 附录 E：逐章重写覆盖矩阵

> 基线：`upstream/main@283bc4cf011047314b4804c0f1ccd06e4f6a95c5`（2026-06-24）。

本表用于证明“原有每一章都已基于最新代码直接覆盖更新”，而不是仅在旁边新增一套文档。章节只有在完成源码锚点复核、正文改写和验证后，才能从“待重写”改为“已完成”。

## 验收标准

每章至少满足：

1. 当前行为、历史行为和设计推断分开表述。
2. 核心路径、类型与函数在基线提交中真实存在。
3. 删除或改写已经失效的文件名、协议名、默认值和规模数据。
4. 解释关键状态、算法、异常路径与跨模块边界。
5. 提供可执行的 `rg`、`git log` 或测试命令。
6. Mermaid、相对链接和代码路径通过全量校验。

## Part I

| 章 | 主题 | 当前源码主线 | 状态 |
| --- | --- | --- | --- |
| 01 | 项目全景与设计哲学 | workspace、protocol、runtime、tools、安全、状态与扩展边界 | 已完成 |
| 02 | 多入口与启动分发 | npm launcher、multicall CLI、TUI、exec、app-server、SDK | 已完成 |
| 03 | 配置系统与企业要求 | config layer stack、requirements、trust、AGENTS、managed policy | 已完成 |
| 04 | 初级使用方法 | install、login、TUI、exec、默认权限与恢复 | 已完成 |
| 05 | 高级使用方法 | resume/fork、MCP、plugins、remote environment、cloud | 已完成 |
| 06 | Agent 核心循环 | submission、task、turn、sampling、pending input、hooks | 已完成 |
| 07 | Prompt 与 Skill 注入 | context fragments、skills、plugins、connectors、tool exposure | 已完成 |
| 08 | Provider 与 API 模式 | provider、Responses、WebSocket、Realtime、retry、metadata | 已完成 |

## Part II

| 章 | 主题 | 当前源码主线 | 状态 |
| --- | --- | --- | --- |
| 09 | 工具系统总览 | spec plan、registry、router、runtime、direct/deferred exposure | 已完成 |
| 10 | 命令执行与 unified exec | exec expiration、process manager、PTY、output cap、cancellation | 已完成 |
| 11 | apply_patch | grammar、PathUri、ExecutorFileSystem、preview、approval、commit | 已完成 |
| 12 | macOS/Linux 沙箱 | Seatbelt、bwrap、landlock、filesystem/network policy | 已完成 |
| 13 | Windows 沙箱与 WFP | token、ACL、WFP、setup、平台限制 | 已完成 |
| 14 | execpolicy | Starlark rules、decision、amendment、legacy bridge | 已完成 |
| 15 | 网络代理与策略 | proxy、network approval、policy、late denial | 已完成 |
| 16 | Hook 生命周期 | discovery、source、pre/post tool、stop/session hooks | 已完成 |
| 17 | Plugin 市场 | manifest、marketplace、install、selected plugin、attribution | 已完成 |
| 18 | MCP 双向集成 | connection manager、OAuth、elicitation、direct/deferred tools | 已完成 |
| 19 | 会话与轨迹 | rollout、reconstruction、thread-store、state、rollback/fork | 已完成 |
| 20 | 记忆系统 | memory read/write/runtime、范围、注入与重置 | 已完成 |
| 21 | App Server | initialize、v2、serialization scope、projector、schema | 已完成 |

## Part III

| 章 | 主题 | 当前源码主线 | 状态 |
| --- | --- | --- | --- |
| 22 | TUI 与 Code Mode | streaming/history cell、render width、V8 host protocol | 已完成 |
| 23 | Cloud Tasks 与迁移 | cloud workflow、external migration、agent graph | 已完成 |
| 24 | 同类架构对比 | 先按当前 Codex 机制建立对比维度，再更新外部证据 | 已完成 |
| 25 | 沙箱与权限对比 | permission、approval、sandbox、network、guardian 分层比较 | 已完成 |

## 缺漏主题补充结果

| 缺漏主题 | 正式落点 | 状态 |
| --- | --- | --- |
| Context 构建、token budget、compaction 与 reference context | `Appendix/F-Context预算压缩与ReferenceContext.md` | 已完成 |
| Multi-Agent、Review、Guardian 与 Goal | `Appendix/G-MultiAgentReviewGuardian与Goal.md` | 已完成 |
| 工程化测试、schema/snapshot、telemetry | `Appendix/H-工程测试Schema与可观测性.md` | 已完成 |
| PathUri、Environment Registry 与远程执行 | `Appendix/I-PathUri环境注册与远程执行.md` | 已完成 |

`Current Source Analysis/` 保留为研究素材；正式阅读主线是原 25 章与附录 F–I。
