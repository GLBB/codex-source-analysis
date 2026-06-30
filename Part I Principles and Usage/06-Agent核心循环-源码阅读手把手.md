# 06 实战｜Agent 核心循环源码阅读手把手

这篇是《06｜Agent 核心循环：从一次提交到多次采样》的实战阅读版。原文讲原理，这篇讲“怎么打开源码、先看哪里、忽略哪里、卡住时怎么继续”。

适合的阅读方式：

1. 左边打开本文；
2. 右边打开源码；
3. 每一步只完成本节目标，不展开所有细节；
4. 每读完一步，用“我现在能解释什么”检查自己。

如果 Rust 语法影响阅读，配合《[06 补充｜Agent 核心循环涉及的 Rust 源码阅读知识](06-Agent核心循环-Rust源码阅读知识.md)》一起看。

## 0. 阅读前先定边界

这次只读一条主线：

```text
用户提交一次输入
  -> 外部把 Op::UserInput 交给 CodexThread
  -> Codex 把 Submission 放入 channel
  -> 后台 submission_loop 取出 Submission
  -> Op::UserInput 被分派成 RegularTask
  -> Session task 框架登记 active turn 并启动后台 task
  -> RegularTask 进入 session::turn::run_turn
  -> run_turn 可能多次请求模型和执行工具
  -> core 发出 EventMsg
  -> 外部通过 next_event 读到事件
```

先不要读：

- app-server 的完整 RPC 协议；
- TUI UI 展示逻辑；
- MCP 每个工具的实现；
- sandbox 的平台细节；
- compact 的完整摘要算法；
- multi-agent 的完整子代理协议。

这些都是后续专题。第一遍只要把主循环走通。

## 1. 第一站：从 sample 看外部调用方式

打开：

```text
codex-rs/thread-manager-sample/src/main.rs
```

只看三个函数：

```text
run_main
new_config
run_turn
```

关注点：

1. `run_main` 如何创建 `ThreadManager`；
2. `thread_manager.start_thread(config)` 如何得到 `CodexThread`；
3. sample 自己的 `run_turn` 如何调用 `thread.submit(...)`；
4. sample 自己的 `run_turn` 如何循环 `thread.next_event().await`。

先把这段代码翻译成一句话：

```text
sample 是一个外部消费者：它提交 Op，然后消费 Event。
```

不要纠结：

- `Config` 里每个字段是什么；
- `AuthManager`、`EnvironmentManager` 内部怎么实现；
- 为什么很多参数传 `None`；
- 事件映射到 app-server notification 的全部细节。

这一站只建立外部视角。

检查自己：

```text
我能否说清楚 sample 里 submit 和 next_event 分别对应什么方向？
```

答案应该是：

```text
submit：外部 -> core
next_event：core -> 外部
```

## 2. 第二站：`CodexThread` 只是线程句柄

打开：

```text
codex-rs/core/src/codex_thread.rs
```

这一站重点看四个入口：

- `CodexThread` 结构体；
- `submit`：外部把操作送进 core；
- `next_event`：外部从 core 取事件；
- `shutdown_and_wait`：关闭底层 session loop 并等待结束。

你会看到 `CodexThread` 里有一个核心字段：

```rust
codex: Codex
```

以及很多薄包装方法：

```rust
pub async fn submit(&self, op: Op) -> CodexResult<String> {
    self.codex.submit(op).await
}

pub async fn next_event(&self) -> CodexResult<Event> {
    self.codex.next_event().await
}
```

关注点：

1. `CodexThread` 不是 Agent loop 本身；
2. 它是 thread 级别的门面；
3. 很多方法只是转发到内部 `Codex` 或 `Session`；
4. 外部调用方不直接碰 `Session`，而是通过 `CodexThread` 操作。

不要纠结：

- 所有 config snapshot 字段；
- MCP resource 方法；
- metadata 更新；
- background terminal 管理。

这一站只记住：

```text
CodexThread = 外部持有的 thread 句柄
```

常见疑问：

**问：为什么不直接暴露 Session？**  
因为 `Session` 太强大，包含大量内部可变状态。`CodexThread` 是更窄、更安全的 API 边界。

## 3. 第三站：`Codex` 是两条单向队列的门面

打开：

```text
codex-rs/core/src/session/mod.rs
```

先看 `Codex` 结构体，再看它的三个入口方法：

```text
Codex
Codex::submit
Codex::submit_with_id
Codex::next_event
```

```rust
pub struct Codex {
    tx_sub: Sender<Submission>,
    rx_event: Receiver<Event>,
    session: Arc<Session>,
    ...
}
```

这个结构是理解 core 的关键。源码注释说它是一个 queue pair：外部往里送 `Submission`，外部再从里面收 `Event`。这两条路方向相反，但都集中在 `Codex` 这个门面上。

先把字段按“外部能做什么”来读：

```text
Codex
  tx_sub    外部用它把 Submission 发送给后台 session loop
  rx_event  外部用它从后台 session loop 接收 Event
  session   指向内部长期状态；Codex 持有它，但不在 submit 里直接跑 turn
```

注意这里的名字容易误导：`tx_sub` 里的 `tx` 是 sender。`Codex` 持有的是发送端；真正接收 `Submission` 的 `rx_sub` 不在结构体里，它被交给了后台的 `submission_loop`。反过来，`rx_event` 是 event 的接收端；发送端 `tx_event` 放在 `Session` 里，内部代码通过 `Session::send_event(...)` 把事件发出来。

所以这不是“一条队列”，而是两条单向通道：

```text
提交方向：
外部调用方
  -> CodexThread::submit(...)
  -> Codex::submit(...)
  -> Codex::submit_with_id(...)
  -> tx_sub.send(Submission)
  -> 后台 submission_loop 从 rx_sub.recv() 取出 Submission

事件方向：
内部 Session
  -> Session::send_event(...) / send_event_raw(...)
  -> tx_event.send(Event)
  -> Codex::next_event(...)
  -> rx_event.recv()
  -> 外部调用方
```

再把它和前两站连起来：

```text
sample 持有 CodexThread
  -> CodexThread 转发给 Codex
  -> Codex 只负责把输入放进队列、把事件从队列拿出来
  -> Session / submission_loop / task 才负责真正执行 turn
```

这一站最容易读错的是“submit 返回了什么”。`submit` 返回的是 submission id，不是模型最终回答。这个 id 用来标识这次提交，后面的事件也会带着对应的 id 或 turn id；真正的输出要继续通过 `next_event()` 一条条读。

可以用一个非常小的时序图记住：

```text
时间 1：外部 submit(Op::UserInput)      -> 得到 submission id
时间 2：后台 loop 收到 Submission       -> 分发成 task
时间 3：task / turn 持续发 EventMsg      -> 外部 next_event() 逐条收到
时间 4：外部收到 TurnComplete/Aborted   -> 这一 turn 才算结束
```

不要纠结：

- `CodexSpawnArgs` 全部字段；
- model catalog 刷新；
- exec policy 加载；
- startup prewarm。

这一站你要建立的心智模型是：

```text
Codex 不是“调用 Agent 并等待回答”的对象。
Codex 是外部调用方和后台 session loop 之间的异步收发门面。
```

检查自己：

```text
为什么 submit 之后不会立刻拿到最终回答？
```

答案：

```text
submit 只是生成 id，并把 Submission 放进 tx_sub。
真正执行发生在后台 submission_loop 里。
执行过程和结果会被包装成 Event，通过 rx_event 被 next_event 读出来。
```

## 4. 第四站：`Codex::spawn` 启动后台 session loop

仍然在：

```text
codex-rs/core/src/session/mod.rs
```

看：

```text
Codex::spawn
Codex::spawn_internal
```

重点找这几件事：

```text
async_channel::bounded(...)
async_channel::unbounded(...)
Session::new(...)
tokio::spawn(submission_loop(...))
```

你要拼出这条链：

```text
Codex::spawn
  -> 创建 tx_sub / rx_sub
  -> 创建 tx_event / rx_event
  -> 把 tx_event 交给 Session
  -> Session::new(...)
  -> 把 rx_sub 交给 tokio::spawn(submission_loop(...))
  -> 返回 Codex { tx_sub, rx_event, session, ... }
```

这一步非常关键：它解释了为什么后面是异步状态机。

把第 3 站和第 4 站合起来看，四个通道端点各有归属：

| 端点 | 谁持有 | 用途 |
| --- | --- | --- |
| `tx_sub` | `Codex` | 外部提交 `Submission`。 |
| `rx_sub` | `submission_loop` | 后台接收并分发 `Submission`。 |
| `tx_event` | `Session` | 内部发送 `Event`。 |
| `rx_event` | `Codex` | 外部读取 `Event`。 |

常见疑问：

**问：`submission_loop` 是谁调用的？**  
不是普通函数同步调用，而是 `tokio::spawn` 启动的后台任务调用。

**问：外部怎么和这个后台任务通信？**  
通过 `tx_sub.send(Submission)`。

**问：后台任务怎么把消息给外部？**  
通过 `tx_event.send(Event)`，外部用 `next_event()` 从 `rx_event` 读。

## 5. 第五站：`ThreadManager` 负责创建 thread，不负责跑 turn

打开：

```text
codex-rs/core/src/thread_manager.rs
```

目标是拼出：

```text
ThreadManager::start_thread
  -> start_thread_with_tools
  -> start_thread_with_options
  -> start_thread_with_options_and_fork_source
  -> ThreadManagerState::spawn_thread_with_source
  -> Codex::spawn
  -> finalize_thread_spawn
  -> CodexThread::new
  -> threads.entry(thread_id).insert(thread)
```

中间会经过 state/finalize 层。第一遍只看“创建 `Codex`”和“登记 `CodexThread`”这两个动作，不必展开每个参数。

关注点：

1. `ThreadManager` 管 thread 生命周期；
2. 它创建 `Codex`；
3. 它包装成 `CodexThread`；
4. 它把 thread 放进内存 map；
5. 它不是每个 turn 的执行循环。

不要纠结：

- resume/fork 的所有分支；
- subagent spawn；
- originator 细节；
- multi-agent version 推导。

这一站的结论：

```text
ThreadManager 是工厂和注册表，run_turn 不在这里。
```

## 6. 第六站：从 Submission 找到分发点

打开：

```text
codex-rs/core/src/session/handlers.rs
```

重点看：

```text
submission_loop
  -> rx_sub.recv()
  -> submission_dispatch_span
  -> match sub.op
  -> Op::UserInput
  -> user_input_or_turn
  -> user_input_or_turn_inner
  -> sess.spawn_task(..., RegularTask::new())
```

你要确认：

1. `Submission` 是从 `rx_sub` 取出来的；
2. 每个 submission 会有一个 tracing span；
3. `Op::UserInput` 会被分派到用户输入处理路径；
4. `user_input_or_turn_inner` 会建立新的 turn context；
5. 如果当前没有可 steering 的 active turn，用户输入会进入 `RegularTask`。

不要一次性读完整 `handlers.rs`，它会处理很多 `Op`：

- shutdown；
- interrupt；
- approvals；
- compact；
- review；
- user input；
- realtime；
- thread settings。

第一遍只追 `Op::UserInput`。

常见疑问：

**问：为什么有这么多 Op？**  
因为 `submit` 不只提交用户消息，也提交控制操作。比如 interrupt、approval response、compact 都是运行时控制面。

**问：`Op::UserInput` 为什么不是直接调用 `run_turn`？**  
因为 handlers 层还要先处理 thread settings、turn context、pending input 和 active turn。真正的普通 turn 会通过 `sess.spawn_task(..., RegularTask::new())` 进入 task 生命周期。

## 6.5. 第六点五站：`spawn_task` 进入 task 框架

打开：

```text
codex-rs/core/src/tasks/mod.rs
```

第六站最后看到的是：

```text
sess.spawn_task(..., RegularTask::new())
```

先不要直接跳到 `RegularTask::run`。中间还有一层 task 框架，它解释“这个 task 是怎么被启动、登记、收尾的”。

重点看：

```text
Session::spawn_task
  -> abort_all_tasks(TurnAbortReason::Replaced)
  -> clear_connector_selection()
  -> start_task(...)

Session::start_task
  -> 创建 CancellationToken / Notify
  -> 记录 turn started 时间和 token 起点
  -> 把已有 pending input 归到当前 turn state
  -> emit_turn_start_lifecycle(...)
  -> 创建 session_task.turn span
  -> tokio::spawn(async move { task.run(...).await; ... })
  -> task.run 返回后 flush_rollout()
  -> Session::on_task_finished(...)
```

这一站回答一个很容易漏掉的问题：

```text
RegularTask::run 是谁调用的？
```

答案不是 `handlers.rs` 直接调用，而是 `Session::start_task` 把 `RegularTask` 包成一个后台 Tokio task，然后在后台 task 里调用 `task.run(...)`。

这里还有一个边界要分清：

```text
Session::spawn_task
  负责替换当前任务：先中断旧 task，再启动新 task。

Session::start_task
  负责通用 task 生命周期：登记 active turn、创建取消令牌、打开 span、启动后台任务、统一收尾。

RegularTask::run
  只负责普通用户 turn 的业务执行。
```

所以第六站和第七站之间真正的桥是：

```text
user_input_or_turn_inner
  -> sess.spawn_task(..., RegularTask::new())
  -> Session::spawn_task
  -> Session::start_task
  -> tokio::spawn(...)
  -> task.run(...)
  -> RegularTask::run
```

关注点：

1. `spawn_task` 会先替换旧的 active task；
2. `start_task` 是所有 `SessionTask` 共用的启动框架；
3. `RegularTask` 只是作为参数传进去的具体 task 实现；
4. `tokio::spawn` 之后，`submission_loop` 不会同步等待模型跑完；
5. task 结束后的 `TurnComplete / TurnAborted` 也在 task 框架里统一处理。

不要纠结：

- 每个 tracing 字段；
- guardian circuit breaker 的细节；
- token metrics 的完整计算；
- compact/review/user-shell task 的差异。

检查自己：

```text
我能否说清楚 handlers.rs 为什么只到 spawn_task，而 RegularTask::run 为什么还能被调用？
```

答案应该是：

```text
handlers.rs 只决定启动哪个 SessionTask；
tasks/mod.rs 负责把这个 SessionTask 放进后台任务并调用 run；
regular.rs 才定义普通用户 turn 的 run 具体做什么。
```

## 7. 第七站：`RegularTask` 是普通用户 turn 的 task 实现

打开：

```text
codex-rs/core/src/tasks/regular.rs
```

这一站要接住第六点五站最后的 `task.run(...)`。先不要直接跳进 `run_turn`，先看 `RegularTask` 在 task 框架里负责哪一段。

重点看：

```text
impl SessionTask for RegularTask
  -> kind() / span_name()
  -> run(...)
  -> emit EventMsg::TurnStarted
  -> consume_startup_prewarm_for_regular_turn(...)
  -> session::turn::run_turn(...)
  -> loop 检查 input_queue.has_pending_input(...)
```

这一步回答一个关键问题：

```text
EventMsg::TurnStarted 是在哪里发出的？
```

答案：普通 turn 在 `RegularTask::run` 里发出 `TurnStarted`，然后才进入真正的 `run_turn`。

但这里有一个边界要分清：

```text
Session::start_task
  负责创建 task、登记 active turn、打开 task span、tokio::spawn 后台任务

RegularTask::run
  负责普通用户 turn 的业务执行：发 TurnStarted、准备 prewarm、调用 run_turn

Session::on_task_finished
  负责统一收尾：flush rollout、记录 metrics、发 TurnComplete / TurnAborted
```

所以 `RegularTask` 不是“整个 task 生命周期管理器”。它是普通用户 turn 在 `SessionTask` 框架里的执行体。

关注点：

1. `RegularTask` 是 `SessionTask` trait 的一个实现；
2. `kind()` 返回 `TaskKind::Regular`，`span_name()` 返回 `session_task.turn`；
3. `RegularTask::run` 先发 `EventMsg::TurnStarted`；
4. startup prewarm 只是在进入 `run_turn` 前尝试复用模型 client session；
5. 普通用户 turn 的具体执行交给 `session::turn::run_turn`；
6. `run_turn` 返回后，`RegularTask` 会检查 `input_queue.has_pending_input(...)`；
7. 如果还有 pending input，`RegularTask` 用空 `next_input` 再次调用 `run_turn`，让下一轮从 queue 里 drain 追加输入；
8. 如果没有 pending input，`RegularTask::run` 返回 `last_agent_message`，后续由 `Session::on_task_finished` 发 `TurnComplete`。

这一步的心智模型可以这样画：

```text
Session::start_task(...)
  -> tokio::spawn(task.run(...))
  -> RegularTask::run
       -> TurnStarted
       -> run_turn(input)
       -> 如果 input_queue 还有 pending input，再 run_turn(Vec::new())
       -> 返回 last_agent_message
  -> Session::on_task_finished(...)
       -> TurnComplete / TurnAborted
```

常见疑问：

**问：为什么 `RegularTask` 里有一个 loop，`run_turn` 里也有 loop？**  
`run_turn` 里的 loop 处理同一次模型执行过程中的继续原因，比如工具调用、自动压缩、stop hook 和可 drain 的 pending input。`RegularTask` 外层 loop 是兜底：如果 `run_turn` 返回时队列里仍然有 pending input，就用同一个 task 生命周期再跑一次 `run_turn`。

**问：为什么 `TurnComplete` 不在 `RegularTask::run` 里发？**  
因为不同 task 共用统一收尾逻辑。`RegularTask::run` 只返回结果，`Session::on_task_finished` 负责 metrics、rollout flush 和最终 `TurnComplete / TurnAborted`。

不要纠结：

- startup prewarm 的细节；
- span 的所有字段；
- extension data 的生命周期。

## 8. 第八站：第一次读 `session/turn.rs`，只看主干

打开：

```text
codex-rs/core/src/session/turn.rs
```

第一次读 `run_turn`，只分块，不进细节。

把它切成三段：

```text
第一段：turn 前准备
  - ModelClientSession
  - pre-sampling compact
  - context updates
  - skills/plugins/connectors
  - hooks
  - record inputs
  - TurnDiffTracker

第二段：loop 主体
  - pending input
  - StepContext
  - build prompt
  - run_sampling_request
  - tool outputs
  - continue / compact / stop

第三段：结束
  - 返回 last_agent_message
  - 外层 task 负责 TurnComplete / TurnAborted
```

第一遍不要追：

- 每个 hook 怎么实现；
- 每个 tool 怎么执行；
- compact 内部怎么摘要；
- prompt 每个 fragment 怎么渲染。

你的目标只是能画出：

```text
run_turn
  -> 准备上下文
  -> 进入 loop
  -> 每轮 build prompt
  -> 每轮 sampling
  -> 根据工具/输入/压缩/stop hook 决定是否继续
```

## 9. 第九站：理解一次 sampling

仍然在 `turn.rs`。这一站只顺着一次采样读四个位置：

```text
build_prompt
run_sampling_request
try_run_sampling_request
ResponseEvent 处理分支
```

遇到 `handle_output_item_done` 时，跳到 `stream_events_utils.rs` 看它如何把模型返回的 output item 转成历史记录、事件或工具调用结果。

关注点：

1. `build_prompt` 从 history 和上下文生成模型请求；
2. `run_sampling_request` 发起一次模型请求；
3. 模型流式返回 `ResponseEvent`；
4. 非工具输出会变成 assistant message / reasoning / delta；
5. 工具调用会进入 tool runtime；
6. 工具输出会记录下来，供下一次 sampling 使用。

你要建立这条子链：

```text
history + context + tools
  -> build_prompt
  -> run_sampling_request
  -> model stream
  -> response items
  -> tool calls or assistant message
```

常见疑问：

**问：为什么一次用户输入会请求模型多次？**  
因为第一次 sampling 可能只得到 tool call。工具执行完后，工具结果要回送给模型，所以需要第二次 sampling。

**问：UI 看到的 delta 会原样进下一次模型请求吗？**  
不会。UI 事件和模型 history 是不同投影。下一次模型看到的是规范化记录后的 conversation items。

## 10. 第十站：找循环继续的原因

回到 `run_turn` 的 loop。读这一段时，把几个状态变量当成路标：

```text
pending_input
model_needs_follow_up
needs_follow_up
auto_compact_needed
stop_hook_active
can_drain_pending_input
```

把继续原因归类：

```text
1. 模型调用了工具
   -> 工具结果需要送回模型

2. 用户追加了 pending input
   -> 合适的采样边界吸收进 history

3. 上下文逼近限制
   -> 自动压缩后继续当前 turn

4. stop hook 要求续写
   -> 模型看似结束，但 hook 让它继续
```

读这一段时最重要的不是每个 bool 的名字，而是判断：

```text
当前 turn 到底能不能结束？
如果不能，是哪一种原因让它继续？
```

## 11. 第十一站：看事件从哪里发出来

现在把视角切到可观测性。

重点位置：

```text
tasks/regular.rs
  TurnStarted

tasks/mod.rs
  TurnComplete / TurnAborted

session/turn.rs
  AgentMessageContentDelta / ReasoningContentDelta / PlanDelta / Warning 等

stream_events_utils.rs
  ItemStarted / ItemCompleted / response item 处理

tools/
  ExecCommandBegin / ExecCommandOutputDelta / ExecCommandEnd / MCP events 等
```

你要理解：

```text
EventMsg 是 core 给外部世界看的运行时投影。
```

它不等于模型 history，也不等于 rollout 全部内容。

常见疑问：

**问：为什么我在 sample 看到很多 JSON 行？**  
因为 sample 把部分 `EventMsg` 映射成 app-server notification，然后作为 NDJSON 输出。

**问：为什么有些 EventMsg 没输出？**  
sample 只映射一部分事件；完整客户端会处理更多事件。

## 12. 第十二站：看 turn 怎么结束

打开：

```text
codex-rs/core/src/tasks/mod.rs
```

关注点：

1. `run_turn` 返回后，task 统一进入 `on_task_finished`；
2. 这里记录 token usage、tool call、duration 等 metrics；
3. 成功时发 `EventMsg::TurnComplete`；
4. 被中断时发 `EventMsg::TurnAborted`；
5. 最后清理 active turn 状态。

这一步连接了两个世界：

```text
内部 task 生命周期结束
  -> 对外 EventMsg::TurnComplete / TurnAborted
```

常见疑问：

**问：为什么 `run_turn` 里没有直接发 `TurnComplete`？**  
因为不同 task 类型共享统一的完成生命周期。`run_turn` 专注执行回合，task 层负责统一收尾和指标。

## 13. 第十三站：用 tracing 和 metrics 验证理解

最后再看 tracing 和 metrics。不要把它当成另一条主线，只把它当成验证理解的路标。

先看 `submission_dispatch_span`，它回答“这个 `Op` 是怎么被分发的”。再看 task 层围绕 turn 创建的 span，例如 `session_task.turn`、`session_task.run` 和 `run_turn`，它们把一次用户 turn 的线程、模型、token 字段串起来。最后看 task 收尾处记录的 token、工具调用次数和端到端耗时。

关注点：

| 可观测性 | 对应问题 |
| --- | --- |
| `submission_dispatch_span` | 这个 `Op` 是怎么被分发的？ |
| `session_task.turn` / `run_turn` | 这个 turn 属于哪个 thread、哪个 model？ |
| `TurnStarted/TurnComplete` | 用户可见生命周期在哪里开始/结束？ |
| token metrics | 这一轮用了多少 token？ |
| tool metrics | 这一轮调用了多少工具？ |

读异步系统时，tracing span 很重要。它相当于运行时调用栈的路标。

## 14. 一次完整心智回放

读完上面各站，尝试不看源码复述：

```text
1. 外部调用 CodexThread::submit(Op::UserInput)。
2. CodexThread 把调用转发给内部 Codex。
3. Codex::submit 生成 submission id。
4. Codex::submit_with_id 把 Submission 送入 tx_sub。
5. 后台 submission_loop 从 rx_sub 取出 Submission。
6. handlers 根据 Op 类型分发。
7. Op::UserInput 进入 user_input_or_turn_inner。
8. handlers 建立 turn context，整理用户输入和 additional context。
9. 普通用户输入通过 sess.spawn_task(..., RegularTask::new()) 启动 task。
10. Session::spawn_task 先替换旧 task，然后进入 start_task。
11. Session::start_task 登记 active turn，创建取消令牌和 task span。
12. start_task 用 tokio::spawn 启动后台 task，并调用 task.run。
13. RegularTask 先发 TurnStarted。
14. RegularTask 调用 session::turn::run_turn。
15. run_turn 准备上下文、hooks、skills、history。
16. run_turn 进入 loop。
17. 每轮 loop 构建 StepContext 和 Prompt。
18. run_sampling_request 请求模型。
19. 模型可能返回工具调用或 assistant message。
20. 工具调用执行后，结果写回 history。
21. 如果还需要继续，就再次 sampling。
22. 如果可以结束，run_turn 返回。
23. task 层 on_task_finished 记录指标并发 TurnComplete。
24. Session 通过 tx_event 发出 Event。
25. 外部 next_event 从 rx_event 收到 TurnComplete。
```

如果能顺畅讲出这 25 步，就说明主线已经打通。

## 15. 常见卡点与解法

### 卡点 1：`Session` 和 `Codex` 总是分不清

先这样记：

```text
Codex = 队列门面，负责 submit / next_event
Session = 内部状态和事件发送者，负责支撑真正的 turn 执行
```

`Codex` 更像门口的收发室。它收外部提交，也把内部事件递给外部；`Session` 才保存线程状态、发事件、支撑 task 和 turn。

### 卡点 2：为什么不直接调用 `run_turn`

因为 Codex 要支持：

- 中断；
- 审批响应；
- pending input；
- shutdown；
- compact；
- 多客户端事件消费；
- 异步工具和模型流。

所以外部通过 `Submission` channel 进入，而不是同步调用 `run_turn`。

### 卡点 3：`run_turn` 太长，看不动

不要从上到下逐行读。切成五块：

```text
1. 预处理
2. 记录输入和上下文
3. loop 前状态
4. loop 每轮 sampling
5. 继续/结束判断
```

每次只读一块。

### 卡点 4：事件、history、rollout 混在一起

分成三种输出：

```text
EventMsg   给 UI / app-server 看
history    给下一次模型请求看
rollout    给恢复和审计看
```

它们可能来自同一件事，但不是同一份数据。

### 卡点 5：工具调用到底在哪里继续模型

关键是：

```text
模型返回 tool call
  -> runtime 执行工具
  -> 工具结果记录进 history
  -> run_turn loop 再次 build_prompt
  -> 下一次 sampling 把工具结果送回模型
```

不是工具自己调用模型，而是 `run_turn` 的循环让模型再次采样。

### 卡点 6：看不懂 async 调用链

找这三个东西：

```text
channel：谁发消息，谁收消息
tokio::spawn：哪里开了后台任务
tracing span：运行时调用链怎么标记
```

异步 Rust 的调用链不总是函数嵌套，很多时候是“通过 channel 连接的状态机”。

## 16. 每一步的“停下标准”

| 步骤 | 可以停下的标准 |
| --- | --- |
| sample | 能解释 submit 和 next_event 的方向。 |
| CodexThread | 能解释它只是 thread 句柄。 |
| Codex | 能解释 tx_sub / rx_sub、tx_event / rx_event 两条单向通道。 |
| Codex::spawn | 能解释 submission_loop 是后台任务。 |
| ThreadManager | 能解释它负责创建和登记 thread。 |
| handlers | 能解释 Op::UserInput 怎么进入 regular task。 |
| task 框架 | 能解释 spawn_task/start_task 如何调用 RegularTask::run。 |
| RegularTask | 能解释 TurnStarted 和 run_turn 的关系。 |
| run_turn 主干 | 能画出准备、loop、sampling、继续/结束。 |
| sampling | 能解释工具调用为什么导致再次请求模型。 |
| events | 能说出 TurnComplete 从哪里发出。 |

不要追求第一遍读懂所有细节。Agent 核心循环真正重要的是主链路。

## 17. 推荐的实际阅读节奏

第一天：

```text
sample -> CodexThread -> Codex::submit/next_event
```

第二天：

```text
Codex::spawn -> submission_loop -> handlers -> task 框架 -> RegularTask
```

第三天：

```text
run_turn 主干 -> run_sampling_request -> 继续循环条件
```

第四天：

```text
EventMsg -> tracing span -> metrics -> rollout
```

这个节奏比一天硬啃完 `turn.rs` 更稳，也更像真实工程阅读方式。

## 18. 最后记住一句话

Codex Agent 核心循环不是“收到一句话，调用一次模型，返回一句话”。

它更像：

```text
一个由 Submission 驱动的异步状态机，
在一个 Turn 内多次构造 Prompt、请求模型、执行工具、记录结果，
直到工具、追加输入、压缩和 hook 都不再要求继续，
最后通过 EventMsg 把生命周期通知给外部。
```

读源码时，只要始终把自己放在这条主线上，就不会被 `Config`、MCP、sandbox、metrics、extension 这些支线带丢。
