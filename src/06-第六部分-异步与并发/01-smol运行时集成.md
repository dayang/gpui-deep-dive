# 第 17 章：smol 运行时集成

GPUI 的异步运行时基于 `smol`，不是 `tokio`。

> 源文件参考：`src/executor.rs`、`src/platform.rs` 的 `PlatformDispatcher` trait

## GPUI 的两级执行器

```rust
// src/executor.rs:31-48
pub struct BackgroundExecutor {
    dispatcher: Arc<dyn PlatformDispatcher>,
}

pub struct ForegroundExecutor {
    dispatcher: Arc<dyn PlatformDispatcher>,
    not_send: PhantomData<Rc<()>>,  // 标记 !Send
}
```

- **`BackgroundExecutor`**：线程池，运行后台任务。`Send + Sync`。
- **`ForegroundExecutor`**：主线程专用。`not_send: PhantomData<Rc<()>>` 确保编译器拒绝跨线程移动。
- 两者共享同一个 `dispatcher: Arc<dyn PlatformDispatcher>`。

实际线程池实现在各自的平台中——executor 结构体不直接包含 `smol::Executor`。smol 运行时嵌入在 `PlatformDispatcher` 的实现中。

## Task\<T\>

```rust
// src/executor.rs:58-67
pub struct Task<T>(TaskState<T>);

enum TaskState<T> {
    Ready(Option<T>),
    Spawned(async_task::Task<T>),
}
```

`Task<T>` 的内部是 `TaskState<T>` 枚举——`Spawned` 状态持有一个 `async_task::Task<T>` handle。

### Task 的操作

- **`.await`**：`Task<T>` 实现了 `Future` trait（`src/executor.rs:100-109`），可以直接 `.await`。**没有名为 `await()` 的命名方法**。
- **`detach()`**：丢弃 handle，让 future 在后台独立运行（fire-and-forget）。
- **取消**：通过 `drop(task)` 实现——Drop 会取消内部的 `async_task::Task`。**没有名为 `cancel()` 的命名方法**。

## PlatformDispatcher trait

```rust
// src/platform.rs:562-575
pub(crate) trait PlatformDispatcher: Send + Sync {
    fn is_main_thread(&self) -> bool;
    fn dispatch(&self, runnable: Runnable, label: Option<TaskLabel>);
    fn dispatch_on_main_thread(&self, runnable: Runnable);
    fn dispatch_after(&self, duration: Duration, runnable: Runnable);
    fn now(&self) -> Instant { Instant::now() }
}
```

关键点：
- **参数是 `Runnable`**（`async_task::Runnable`），不是 `Box<dyn FnOnce()>`。Runnable 是 smol 运行时的标准调度原语。
- **`dispatch` 有可选的 `label: Option<TaskLabel>`**（用于性能追踪）。
- **`dispatch_after`** 用于定时器（`smol::Timer` 基于此实现）。
- **`now()`** 提供平台特定时钟（默认 `Instant::now()`）。

## smol::Timer

GPUI 重新导出 `smol::Timer`：

```rust
// 延迟
smol::Timer::after(Duration::from_millis(300)).await;

// 间隔
let mut interval = smol::Timer::interval(Duration::from_secs(1));
loop {
    interval.next().await;
}
```

## 与平台事件循环的融合

```
主线程（GUI 线程）：
  ┌──────────────────────────────────┐
  │  平台事件循环                      │
  │  ├─ CFRunLoop / calloop / MSG    │
  │  ├─ OS 事件（鼠标、键盘）           │
  │  │   └─ App::update(callback)     │
  │  ├─ ForegroundExecutor 任务        │
  │  │   └─ smol future 轮询          │
  │  └─ display_link / vsync          │
  │      └─ Window::draw()            │
  └──────────────────────────────────┘

后台线程池：
  ┌──────────────────────────────┐
  │  BackgroundExecutor (N 线程)   │
  │  ├─ smol runtime              │
  │  └─ 用户 spawn 的 futures     │
  └──────────────────────────────┘
```

主线程 smol executor 寄生在平台事件循环中——不是 tokio 的全接管模式。

## 平台实现

- **macOS**：GCD `dispatch_async_f`（底层通过 `src/platform/mac/dispatch.h` 的 bindgen 绑定）
- **Linux/FreeBSD**：`calloop` 事件循环 + `eventfd` 唤醒
- **Windows**：`PostMessage` + 私有消息

所有平台实现都通过 `Runnable` 与 smol 的调度器对接——这是 smol 生态的标准模式。

## 常见的并发陷阱

1. **在 `update()` 回调中再次调用 `update()`**：会导致 `&mut App` 重复借用 → panic。
2. **主线程阻塞**：CPU 密集工作放在 `BackgroundExecutor` 上，不要在主线程执行。
3. **忘记 `.ok()`**：`AsyncApp::update()` 返回 `Result`，App 可能已退出。

---

**如果你只记住一件事：** `BackgroundExecutor` 和 `ForegroundExecutor` 都持有 `Arc<dyn PlatformDispatcher>`。`PlatformDispatcher::dispatch(runnable: Runnable)` 是 smol 的标准调度原语（不是 `Box<dyn FnOnce()>`）。`Task<T>` 实现 `Future` trait，通过 `.await` 等待（不是 `.await()` 方法），通过 `drop()` 取消（不是 `.cancel()` 方法）。
