# 06 补充｜Agent 核心循环涉及的 Rust 源码阅读知识

这篇是《06｜Agent 核心循环：从一次提交到多次采样》的配套阅读笔记，目标不是系统讲 Rust，而是帮助你读懂第 12 节源码阅读路线中会遇到的 Rust 写法。

建议打开源码时把这篇放在旁边：遇到看不懂的语法，先在这里找对应概念，再回到业务调用链。

## 1. `mod.rs` 与模块路径

Rust 的 `mod` 是 module 的意思。文件名 `mod.rs` 通常表示“这个目录模块的入口文件”。

例如：

```text
codex-rs/core/src/session/mod.rs
codex-rs/core/src/session/session.rs
codex-rs/core/src/session/turn.rs
```

对应模块路径大致是：

```rust
crate::session          // session/mod.rs
crate::session::session // session/session.rs
crate::session::turn    // session/turn.rs
```

所以看到下面这种路径不要困惑：

```rust
use crate::session::session::Session;
```

它不是重复写错，而是：

```text
外层 session 模块 :: 内层 session 子模块 :: Session 类型
```

阅读建议：先看 `mod.rs` 里声明了哪些子模块，再跳到具体文件。

## 2. `pub`、`pub(crate)` 与封装边界

Codex core 里经常看到：

```rust
pub struct CodexThread { ... }
pub(crate) mod session;
pub(crate) fn something(...) { ... }
```

含义：

| 写法 | 可见范围 |
| --- | --- |
| `pub` | 对外部 crate 可见。 |
| `pub(crate)` | 只在当前 crate 内可见。 |
| 不写 `pub` | 只在当前模块及子模块可见。 |

例如 `CodexThread` 是公开类型，但很多字段是私有的，外部只能通过 `submit`、`next_event`、`shutdown_and_wait` 等方法操作。

这就是 Rust 代码里的 API 边界：类型可以公开，但状态不一定公开。

## 3. `struct`、`enum` 与状态建模

读 Agent 核心循环时会遇到很多结构体和枚举。

结构体表示一组状态：

```rust
pub struct Codex {
    pub(crate) tx_sub: Sender<Submission>,
    pub(crate) rx_event: Receiver<Event>,
    pub(crate) session: Arc<Session>,
}
```

枚举表示有限分支：

```rust
pub enum EventMsg {
    TurnStarted(...),
    TurnComplete(...),
    Error(...),
    ...
}
```

读 `EventMsg`、`Op`、`TaskKind` 这类 enum 时，不要只看名字，要把它们当成状态机的分支。比如：

```text
Op::UserInput       外部提交了一次用户输入
EventMsg::TurnStarted   core 告诉外部 turn 开始了
EventMsg::TurnComplete  core 告诉外部 turn 结束了
```

## 4. `impl`：给类型实现方法

Rust 方法通常写在 `impl` 块里：

```rust
impl CodexThread {
    pub async fn submit(&self, op: Op) -> CodexResult<String> {
        self.codex.submit(op).await
    }
}
```

这里的 `&self` 表示“借用当前对象”，不会拿走所有权。调用后还能继续用同一个 `CodexThread`。

源码阅读时可以先扫 `impl TypeName`，它往往比字段更能说明这个类型的职责。

## 5. 所有权、借用与 `Arc`

Agent runtime 是异步、多任务系统，很多对象需要被多个任务共享，所以 `Arc` 很常见：

```rust
Arc<Session>
Arc<TurnContext>
Arc<ThreadManagerState>
```

心智模型：

| 类型 | 用途 |
| --- | --- |
| `T` | 当前作用域拥有这个值。 |
| `&T` | 只读借用，不拿走所有权。 |
| `&mut T` | 可变借用，独占修改。 |
| `Arc<T>` | 多个异步任务共享同一个值。 |

常见写法：

```rust
Arc::clone(&session)
```

这不是深拷贝 `Session`，只是增加引用计数，让另一个任务也能持有同一个 `Session`。

阅读建议：看到 `Arc::clone`，就想“这里要把同一个对象交给另一个异步任务或长期持有者”。

## 6. `async fn`、`.await` 与 Future

Codex 的核心路径基本都是异步的：

```rust
pub async fn submit(&self, op: Op) -> CodexResult<String> {
    self.codex.submit(op).await
}
```

含义：

```text
async fn  返回一个 Future
.await    等待这个 Future 完成
```

为什么需要 async？

- 等模型流式响应；
- 等事件 channel；
- 等 tokio task；
- 等文件/数据库/rollout flush；
- 等 MCP/tool 调用。

阅读建议：不要把 `.await` 当普通函数调用。它是“这里可能暂停当前任务，把执行权交还给 runtime，等结果回来再继续”。

## 7. `tokio::spawn`：启动后台任务

在 `Codex::spawn` 里会看到类似：

```rust
tokio::spawn(async move {
    submission_loop(session_for_loop, config, rx_sub).await;
});
```

含义：

```text
启动一个异步后台任务
这个任务负责持续从 rx_sub 读取 Submission
直到收到 Shutdown 或 channel 关闭
```

`async move` 表示把闭包里用到的变量移动进异步任务。因为任务可能比当前函数活得更久，所以它必须拥有需要的值。

阅读建议：看到 `tokio::spawn`，就标记一个新的并发边界。后面的代码不会同步等它一行行跑完，而是通过 channel、watch、CancellationToken 等机制协作。

## 8. channel：`tx_sub` 和 `rx_event`

`Codex` 可以理解成一对队列：

```rust
tx_sub: Sender<Submission>
rx_event: Receiver<Event>
```

提交链：

```text
CodexThread::submit
  -> Codex::submit
  -> tx_sub.send(Submission)
```

事件链：

```text
Session::send_event
  -> tx_event.send(Event)
  -> CodexThread::next_event
  -> rx_event.recv()
```

也就是说，外部不是直接调用 `run_turn`，而是把 `Op` 放进 submission channel；内部执行后再把 `Event` 放进 event channel。

阅读建议：看到 `Sender` / `Receiver`，先问“这条消息从哪里发出，在哪里接收？”

## 9. `Result`、`?` 与错误传播

Rust 里错误经常用 `Result<T, E>` 表示：

```rust
CodexResult<String>
anyhow::Result<()>
ThreadStoreResult<StoredThread>
```

`?` 表示如果是错误就提前返回：

```rust
self.tx_sub.send(sub).await.map_err(|_| CodexErr::InternalAgentDied)?;
```

可以理解成：

```text
成功：取出里面的值继续执行
失败：把错误返回给调用者
```

`map_err` 用来转换错误类型：

```rust
.map_err(|err| ThreadStoreError::Internal {
    message: err.to_string(),
})?
```

阅读建议：看到 `?` 就问“这个错误会冒泡到哪里？最终会变成 `EventMsg::Error`，还是只是函数返回失败？”

## 10. `Option` 与三态配置

`Option<T>` 表示可能有值，也可能没有：

```rust
Option<String>
Option<PathBuf>
Option<ModelClientSession>
```

含义：

```text
Some(value)  有值
None         没有值
```

配置覆盖里还会看到嵌套：

```rust
Option<Option<ReasoningEffort>>
```

这通常表示三态：

```text
None              不修改
Some(None)        显式清空
Some(Some(value)) 设置为 value
```

阅读建议：遇到 `Option<Option<T>>`，不要急着觉得复杂，它通常是在表达 patch/override。

## 11. `if let`、`let ... else` 与模式匹配

Rust 经常用模式匹配拆值。

`if let`：

```rust
if let Some(trace) = sub.trace.as_ref() {
    ...
}
```

意思是：只有当 `trace` 是 `Some` 时才执行。

`let ... else`：

```rust
let Some((injection_items, connectors)) = build_skills_and_plugins(...).await else {
    return Ok(None);
};
```

意思是：如果不是 `Some`，就走 `else` 提前返回；如果是 `Some`，就把里面的值拆出来继续。

`match`：

```rust
match event.msg {
    EventMsg::TurnComplete(_) => return Ok(()),
    EventMsg::Error(event) => bail!(event.message),
    _ => {}
}
```

阅读建议：Agent runtime 的核心状态变化常常藏在 `match Op`、`match EventMsg`、`match task_result` 里。

## 12. trait 与动态分发

读 `tasks` 时会遇到 trait：

```rust
trait SessionTask {
    fn kind(&self) -> TaskKind;
    async fn run(...) -> SessionTaskResult;
}
```

实际代码可能用不同形式表达异步 trait，但理解上可以先当成：

```text
SessionTask 是任务接口
RegularTask / ReviewTask / CompactTask 是不同实现
```

当你看到：

```rust
impl SessionTask for RegularTask
```

意思是 `RegularTask` 实现了 `SessionTask` 这套行为。

阅读建议：先看 trait 方法列表，再看具体 `impl`。这能帮你分清“通用任务生命周期”和“普通用户 turn 的特殊逻辑”。

## 13. `CancellationToken` 与取消传播

Agent turn 可能被用户打断，所以代码里常见：

```rust
CancellationToken
cancellation_token.child_token()
is_cancelled()
```

心智模型：

```text
父 token 取消
  -> 子 token 也会观察到取消
  -> 正在执行的 task / tool / sampling 可以停止
```

阅读建议：看到 `child_token()`，说明这里开了一个可独立传递取消信号的子任务边界。

## 14. `Mutex`、锁与异步状态

`tokio::sync::Mutex` 用来保护异步共享状态：

```rust
let mut active = self.active_turn.lock().await;
```

和普通 `std::sync::Mutex` 不同，tokio 的锁可以 `.await`，适合异步 runtime。

常见模式：

```rust
{
    let mut state = self.state.lock().await;
    state.do_something();
}
```

大括号可以让锁尽早释放。

阅读建议：看到 `.lock().await`，就问“这里保护的是哪个状态？锁持有期间有没有 await 很久的操作？”这对理解死锁和并发边界很重要。

## 15. `watch`、`Notify` 与状态通知

除了普通 channel，源码中还会看到：

```rust
tokio::sync::watch
tokio::sync::Notify
```

大致区别：

| 类型 | 用途 |
| --- | --- |
| `watch` | 保存一个“最新状态”，订阅者关心状态变化。 |
| `Notify` | 发送一次“有人醒醒”的通知，不携带复杂数据。 |

例如 agent status 用 `watch` 很合适，因为 UI 关心的是当前状态；任务完成唤醒等待者用 `Notify` 很合适。

## 16. `tracing`：span、instrument 与可观测性

源码里常见：

```rust
#[instrument(level = "trace", skip_all)]
info_span!("turn", thread.id = %self.thread_id, turn.id = %turn_context.sub_id)
trace_span!("run_turn")
.instrument(span)
```

含义：

| 写法 | 作用 |
| --- | --- |
| `#[instrument]` | 给函数自动加 tracing span。 |
| `info_span!` / `trace_span!` | 手动创建 span。 |
| `.instrument(span)` | 让某个 Future 在这个 span 中执行。 |

这对读 Agent 循环特别有用，因为异步代码不是线性调用栈。span 是运行时调用链的“线索”。

阅读建议：当静态代码看不出谁调用谁时，搜 `info_span!`、`trace_span!`、`submission_dispatch_span`，再对照事件流。

## 17. 宏：`bail!`、`info_span!`、`rg` 搜到的 `foo!`

Rust 里带 `!` 的通常是宏：

```rust
bail!("turn aborted")
info_span!("turn", ...)
trace_span!("run_turn")
format!("turn-{id}")
```

宏不是普通函数，它在编译期展开。阅读时不需要先懂宏展开细节，只要知道它们常见用途：

| 宏 | 用途 |
| --- | --- |
| `format!` | 构造字符串。 |
| `bail!` | 直接返回错误。 |
| `info_span!` / `trace_span!` | 创建 tracing span。 |
| `warn!` / `error!` / `trace!` | 记录日志事件。 |

## 18. 闭包与 `map` / `and_then` / `is_some_and`

Rust 经常用链式方法处理集合和可选值：

```rust
option.map(|value| ...)
result.map_err(|err| ...)
active_turn.task.as_ref().is_some_and(|task| ...)
```

常见含义：

| 方法 | 含义 |
| --- | --- |
| `map` | 成功/有值时转换里面的值。 |
| `map_err` | 转换错误。 |
| `and_then` | 连续处理可能失败/为空的值。 |
| `is_some_and` | `Option` 有值且满足条件。 |

阅读建议：如果链式调用看不懂，先把它手动翻译成 `match` 或 `if let`。

## 19. `Pin`、`Box::pin` 与大型 Future

在 `ThreadManager`、`Session` 附近可能看到：

```rust
Box::pin(self.start_thread_with_tools(...)).await
```

先不用深入 `Pin`。这里可以粗略理解成：

```text
把一个较大的 async future 放到堆上，避免调用方的 async 状态机过大或递归展开过深
```

阅读 Agent 主流程时，只要知道它不改变业务链路：`Box::pin(foo()).await` 仍然是在等待 `foo()` 完成。

## 20. `Clone`、`Copy`、`Default`、`Debug` 等 derive

常见：

```rust
#[derive(Clone, Debug)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
#[derive(Default)]
```

含义：

| derive | 作用 |
| --- | --- |
| `Clone` | 可以显式 `.clone()`。 |
| `Copy` | 小值可以隐式复制，不发生 move。 |
| `Debug` | 可以用 `{:?}` 打印。 |
| `Default` | 可以 `Default::default()`。 |
| `PartialEq` / `Eq` | 可以比较相等。 |

阅读建议：`Default` 在配置/更新结构体里很常见，常和 `..Default::default()` 搭配。

## 21. `..Default::default()` 与结构体补默认值

例如：

```rust
SessionSettingsUpdate {
    collaboration_mode: Some(collaboration_mode),
    personality,
    ..Default::default()
}
```

意思是：

```text
明确设置几个字段
其他字段使用默认值
```

这在配置更新、请求参数、测试 fixture 里很常见。

## 22. `as_ref()`、`as_deref()` 与避免移动

常见：

```rust
turn_context.as_ref()
option.as_ref()
path.as_deref()
```

粗略理解：

| 方法 | 用途 |
| --- | --- |
| `as_ref()` | 把拥有值转换成引用视角，避免移动。 |
| `as_deref()` | 对 `Option<PathBuf>` 这类值取内部引用并自动 deref。 |

阅读建议：看到 `as_ref()`，通常是在“我只想借用，不想拿走所有权”。

## 23. `Vec<T>`、`&[T]` 与 `std::slice::from_ref`

模型 history、事件 items、输入 items 经常是集合：

```rust
Vec<ResponseItem>
&[RolloutItem]
```

`Vec<T>` 是拥有所有权的动态数组。`&[T]` 是只读切片，不关心底层是不是 `Vec`。

有时只有一个 item，但函数需要 slice：

```rust
std::slice::from_ref(&response_item)
```

意思是创建一个长度为 1 的只读切片，避免临时分配一个 `Vec`。

## 24. `into_iter()`、`collect()` 与转换集合

例如：

```rust
self.instruction_sources()
    .await
    .into_iter()
    .map(Into::into)
    .collect()
```

含义：

```text
拿到一个集合
消耗它变成迭代器
逐个转换元素
收集成返回类型需要的新集合
```

返回类型通常帮助编译器推断 `collect()` 收集成什么。

## 25. 阅读路线中的 Rust 难点对照表

| 阅读步骤 | 容易卡住的 Rust 点 | 先理解到什么程度 |
| --- | --- | --- |
| 第 0 步 sample | `derive(Parser)`、`Result`、`Option`、`if let` | 知道 CLI 参数如何解析，事件如何按 `Option` 输出即可。 |
| 第 1 步骨架 | `mod.rs`、`pub(crate)`、`Arc`、`async fn` | 知道模块路径和异步包装层即可。 |
| 第 2 步 Submission -> Task | channel、`tokio::spawn`、trait、`match Op` | 知道 `Submission` 通过 channel 进入分发，再变成 task。 |
| 第 3 步 `run_turn` 主干 | `CancellationToken`、`Mutex`、`let ... else` | 知道 turn 前准备和 loop 的大块结构。 |
| 第 4 步一次采样 | `Stream`、`Future`、工具 runtime、`Result` | 知道 sampling 可能产生文本、工具调用和错误。 |
| 第 5 步继续循环 | `bool` 状态、`Option`、`match` | 知道哪些条件会让 loop 再跑一次。 |
| 第 6 步可观测性 | tracing 宏、span、metrics 方法 | 知道 span/metric/event 是三种不同投影。 |

## 26. 最小 Rust 心智模型

读 Codex Agent 核心循环时，先记住下面几句就够用：

```text
mod.rs 负责组织模块。
pub(crate) 表示只在当前 crate 内开放。
Arc 表示多个异步任务共享同一个对象。
async/.await 表示这里可能暂停并等待外部结果。
tokio::spawn 表示启动一个后台异步任务。
channel 表示通过队列传递 Submission 或 Event。
Result/? 表示错误会向上传播。
Option 表示值可能不存在。
match/if let 是状态机分支的主要入口。
trait/impl 表示用统一接口承载不同任务实现。
tracing span 是异步调用链的可观测性线索。
```

如果一段代码看不懂，先不要急着理解每个泛型。先问三个问题：

1. 这个值是谁拥有的？是 `T`、`&T`，还是 `Arc<T>`？
2. 这一步是在同步执行，还是 `.await` 等异步结果？
3. 失败时是返回 `Result`，发 `EventMsg::Error`，还是只记录 tracing/metric？

这三个问题通常足够把 Rust 语法重新拉回 Agent 业务主线。
