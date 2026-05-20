# 第 18 章：AsyncApp 的正确用法

上一章讲了运行时，本章聚焦于使用层面——如何正确地在异步代码中操作实体和窗口。

> 源文件参考：`src/app/async_context.rs`、`src/app/context.rs` 的 `spawn` 方法

## AsyncApp 的诞生

```rust
// Context::spawn 内部
pub fn spawn<F, R>(&mut self, f: F) -> Task<R>
where
    F: FnOnce(AsyncApp) -> R + 'static,
    R: Future<Output = ()> + 'static,
{
    let async_app = self.to_async();  // Context → AsyncApp
    self.app.spawn(async_app, |async_app| f(async_app))
}
```

`Context<'a, T>` 是短暂的 `&mut` 借用。要跨越 `.await`，需要 `'static` 的上下文。`to_async()` 创建了 `AsyncApp`：

```rust
impl<'a, T> Context<'a, T> {
    pub fn to_async(&self) -> AsyncApp {
        let app = self.app.as_weak();  // &mut App → Weak<AppCell>
        AsyncApp::new(app, self.background_executor.clone(), self.foreground_executor.clone())
    }
}
```

## AsyncApp 的三种操作

### update()：读写访问

```rust
cx.update(|window, cx: &mut Context<MyView>| {
    // &mut self + &mut App 的完全访问
    this.some_field = new_value;
    cx.notify();
}).ok();
```

`update()` 内部流程：
1. `self.app.upgrade()` → 获取 `Arc<AppCell>`
2. `AppCell::update()` → 借入 `RefCell<App>` + 执行闭包
3. 返回结果

### read_with()：只读访问

```rust
let value = cx.read_with(|cx: &Context<MyView>, this: &MyView| {
    this.some_field
}).ok();
```

`read_with()` 只借入 `&App`（共享引用），性能更高，允许多个并发读取。

### update_window()：窗口级操作

```rust
cx.update_window(window_handle, |window, cx: &mut Context<MyView>| {
    window.bounds();  // 读取窗口信息
    window.resize(Size { width: px(800.), height: px(600.) });
}).ok();
```

## spawn 模式

### 模式一：标准的 spawn + detach

```rust
cx.spawn(|mut cx: AsyncApp| async move {
    let data = load_data().await;
    cx.update(|cx: &mut Context<MyView>| {
        this.data = data;
        cx.notify();
    }).ok();
}).detach();
```

### 模式二：持有 Task 以支持取消

```rust
struct MyView {
    loading_task: Option<Task<()>>,
}

impl MyView {
    fn start_loading(&mut self, cx: &mut Context<Self>) {
        let task = cx.spawn(|mut cx| async move {
            let data = load_data().await;
            cx.update(|cx: &mut Context<MyView>| {
                this.data = data;
                this.loading_task = None;
                cx.notify();
            }).ok();
        });
        self.loading_task = Some(task);
    }

    fn cancel_loading(&mut self) {
        if let Some(task) = self.loading_task.take() {
            task.cancel();  // 取消正在进行的加载
        }
    }
}
```

### 模式三：spawn_in（特定窗口）

```rust
cx.spawn_in(window_handle, |mut cx: AsyncWindowContext| async move {
    smol::Timer::after(Duration::from_secs(1)).await;
    cx.update(|window, cx: &mut Context<MyView>| {
        window.resize(Size::default());
    }).ok();
}).detach();
```

`AsyncWindowContext` = `AsyncApp` + 一个 `AnyWindowHandle`，让你在回调中获得 `&mut Window`。

## 错误处理

### update() 返回 Result

```rust
match cx.update(|cx| { ... }) {
    Ok(()) => { /* App 还活着 */ }
    Err(_) => { /* App 已退出——什么也不做 */ }
}
```

在生产代码中，`.ok()` 是最常见的处理方式——App 退出后 async 任务继续执行是正常现象。

### 重试模式

```rust
cx.spawn(|mut cx| async move {
    loop {
        match try_network_call(&mut cx).await {
            Ok(result) => {
                cx.update(|cx| handle_result(result)).ok();
                break;
            }
            Err(e) => {
                log::warn!("Network error: {}, retrying...", e);
                smol::Timer::after(Duration::from_secs(5)).await;
            }
        }
    }
}).detach();
```

## 通道模式：避免嵌套 update

复杂的异步场景中，避免在 `update()` 中再调用 `update()`：

```rust
// 不好：嵌套 update
cx.spawn(|mut cx| async move {
    cx.update(|cx| {
        cx.spawn(|mut cx| async move {  // 在 update 中再 spawn
            // ...
        }).detach();
    }).ok();
}).detach();

// 好：用通道通信
let (tx, rx) = smol::channel::bounded(1);
cx.spawn(|mut cx| async move {
    // 生产者
    loop {
        let data = produce_data().await;
        tx.send(data).await.ok();
    }
}).detach();

cx.spawn(|mut cx| async move {
    // 消费者
    while let Ok(data) = rx.recv().await {
        cx.update(|cx| handle_data(data)).ok();
    }
}).detach();
```

## 测试中的异步

```rust
#[gpui::test]
async fn test_async_behavior(cx: &mut TestAppContext) {
    let view = cx.new(|_| MyView { loading: false });

    // 触发异步操作
    cx.update_entity(&view, |view, cx| {
        view.start_loading(cx);
    });

    // 等待所有异步任务完成
    cx.run_until_parked().await;

    // 验证结果
    cx.read_entity(&view, |view, _| {
        assert!(view.loading);
    });
}
```

`cx.run_until_parked()` 持续推进事件循环（包括异步任务的 poll），直到没有待处理的工作。

## 最佳实践清单

1. **detach 几乎总是正确的**——除非你需要取消功能。
2. **`.ok()` 接受 update 失败**——不要 panic，App 退出是正常的。
3. **BackgroundExecutor 用于 CPU 密集工作**——避免在主线程阻塞。
4. **不要嵌套 update 调用**——用通道或 spawn 新的顶层任务。
5. **Task 可以 cancel**——存储 `Task<T>` 以支持取消正在进行的异步操作。
6. **测试中用 run_until_parked**——确保异步操作完成后再断言。

---

**如果你只记住一件事：** `AsyncApp` 通过 `Weak<AppCell>` 实现 `'static` + `Send`。`update()` 跨主线程桥接，返回 `Result` 以应对 App 已退出。`.detach()` 是 fire-and-forget 的默认选择。
