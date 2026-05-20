# 第 17 章：smol 运行时集成

GPUI 的异步运行时基于 `smol`，不是 `tokio`。这是一个深思熟虑的选择，与 GPUI 的设计理念密切相关。

> 源文件参考：`src/executor.rs`、`src/platform.rs` 中 PlatformDispatcher trait

## 为什么是 smol 而不是 tokio？

| | tokio | smol |
|---|---|---|
| **代码量** | ~100K+ 行 | ~10K 行 |
| **依赖数** | 大量 | 极少 |
| **生态** | 非常成熟 | 较小 |
| **核心抽象** | 自己的 reactor + 线程池 | 基于 `async-io` / `polling` |
| **适合场景** | 高并发服务器 | 嵌入式/桌面应用 |

GPUI 只需要小而精简的异步运行时——它不需要 tokio 的多线程 work-stealing、tracing 基础设施、或丰富的中间件生态。`smol` 的零开销设计和低依赖数量更适合作为 GUI 框架的内嵌运行时。

## GPUI 的两级执行器

```rust
// src/executor.rs
pub struct BackgroundExecutor { /* ... */ } // 线程池，运行后台任务
pub struct ForegroundExecutor { /* ... */ } // 运行在主线程
```

### ForegroundExecutor

运行在主线程上（GUI 线程）。`!Send` ——设计上不可跨线程。

```rust
pub struct ForegroundExecutor {
    // 通过 platform.dispatcher() 将任务提交到主线程事件循环
    dispatcher: Arc<dyn PlatformDispatcher>,
    // 主线程的任务队列
    ready_tasks: RefCell<Vec<...>>,
}
```

在 macOS 上，`PlatformDispatcher` 通过 GCD (`dispatch_async_f`) 将任务调度到主线程。在 Linux 上通过 `calloop`，Windows 上通过消息循环。

### BackgroundExecutor

一个线程池，用于 CPU 密集或 I/O 阻塞任务。所有后台任务都是 `Send + 'static`。

```rust
pub struct BackgroundExecutor {
    // 内部的 smol::Executor 实例
    // 每个 worker thread 一个
    // 默认使用 num_cpus 报告的核心数
}
```

## `Task<T>`：可取消的 Future

```rust
pub struct Task<T> {
    handle: async_task::Task<T>,  // smol 的 async-task
}

impl<T> Task<T> {
    pub async fn await(self) -> T { ... }

    pub fn detach(self) { ... }  // fire-and-forget

    pub fn cancel(self) {
        // Drop Task → handle 被销毁 → Future 的 Drop 被调用
    }
}
```

### cancel 机制

`async_task::Task` 利用 Rust 的标准取消机制——Drop：

```rust
let task = cx.spawn(|mut cx| async move {
    smol::Timer::after(Duration::from_secs(5)).await;
    cx.update(|cx| { /* 5 秒后执行 */ }).ok();
});
// ...
task.cancel();  // Drop → Future 被取消 → Timer 被释放 → 回调不执行
```

这与 tokio 的 `JoinHandle::abort()` 不同——GPUI 的任务取消是**同步**的（`Drop` 是同步的）。

## BackgroundExecutor 的使用

```rust
cx.background_executor()
    .spawn(async move {
        // 这个 async block 在后台线程池中运行
        let result = compute_expensive_thing().await;
        result
    })
    .detach();
```

或使用 `cx.spawn()` 的便利封装：

```rust
cx.spawn(|mut cx| async move {
    // 后台线程中执行
    let data = fetch_from_api().await;

    // 回到主线程更新 UI
    cx.update(|cx| {
        // 现在可以安全修改 Entity 和触发 notify
    }).ok();
})
.detach();
```

## smol::Timer

GPUI 重新导出了 `smol::Timer`：

```rust
// 延迟操作
smol::Timer::after(Duration::from_millis(300)).await;

// 定时器
let mut interval = smol::Timer::interval(Duration::from_secs(1));
loop {
    interval.next().await;
    // 每秒执行
}
```

`smol::Timer` 的实现在 macOS 上使用 `kqueue`，Linux 上使用 `timerfd`，Windows 上使用 `CreateWaitableTimer`。

## PlatformDispatcher 的桥接

```rust
pub trait PlatformDispatcher: Send + Sync {
    fn is_main_thread(&self) -> bool;
    fn dispatch(&self, f: Box<dyn FnOnce() + Send>);  // 异步执行
    fn dispatch_on_main_thread(&self, f: Box<dyn FnOnce() + Send>);  // 排队到主线程
    fn dispatch_after(&self, duration: Duration, f: Box<dyn FnOnce() + Send>);  // 延迟执行
}
```

平台实现：

- **macOS**: GCD `dispatch_async` / `dispatch_async_f`（通过 bindgen 生成的 FFI 绑定）
- **Linux/FreeBSD**: `calloop` 事件循环的 `EventLoop::wake()`
- **Windows**: `PostMessage` 发送私有消息到消息循环

## 与平台事件循环的融合

```
主线程（GUI 线程）:
  ┌────────────────────────────────┐
  │  平台事件循环                    │
  │  ├─ CFRunLoop / calloop / MSG  │
  │  │                              │
  │  ├─ 操作系统事件（鼠标、键盘）     │
  │  │   └─ App::update(callback)   │
  │  │                              │
  │  ├─ ForegroundExecutor 任务      │
  │  │   └─ smol future polling     │
  │  │                              │
  │  └─ display_link / vsync        │
  │      └─ Window::draw()          │
  └────────────────────────────────┘

后台线程池:
  ┌───────────────────────────┐
  │  BackgroundExecutor (N 线程) │
  │  ├─ smol runtime           │
  │  ├─ async-io (kqueue/epoll)│
  │  └─ 用户 spawn 的 futures  │
  └───────────────────────────┘
```

主线程的 smol executor 只是事件循环的"乘客"——平台事件循环是主循环，smol 在其中被轮询。这与 tokio 的全接管模式不同（tokio 的 `#[tokio::main]` 接管了 `main` 函数）。

## 常见的并发陷阱

### 陷阱一：在 update() 中死锁

```rust
// 危险：在 update 回调中再次调用 update
cx.update(|cx| {
    let other = other_entity.clone();
    cx.update_entity(&other, |entity, cx| {
        // 此时已经在 &mut App 的借用中
        // 如果 other_entity 和当前 entity 相同 → panic
    });
}).ok();
```

### 陷阱二：主线程阻塞

```rust
// 危险：在主线程上执行 CPU 密集工作
cx.spawn(|mut cx| async move {
    let result = expensive_calculation();  // ← 在主线程！（如果在 foreground spawn）
    // ...
})
```

使用 `cx.background_executor().spawn()` 将 CPU 工作移到后台。

### 陷阱三：忘记 handle 错误

```rust
cx.spawn(|mut cx| async move {
    smol::Timer::after(Duration::from_secs(1)).await;
    cx.update(|cx| {
        // 如果 App 已经退出，update 返回 Err
        some_entity.update(cx, |entity, cx| { ... });
    }).ok();  // ← 至少写 .ok()，否则编译警告
}).detach();
```

---

**如果你只记住一件事：** GPUI 用 smol（不是 tokio）作为异步运行时。主线程 smol executor 寄生在平台事件循环中。`ForegroundExecutor` 是 `!Send` 的（只在主线程），`BackgroundExecutor` 是线程池。
