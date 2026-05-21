# 第 18 章：AsyncApp 的正确用法

第 17 章讲了运行时，本章聚焦于使用层面——如何正确地在异步代码中操作实体和窗口。

> 源文件参考：`src/app/async_context.rs`、`src/app/context.rs` 的 `spawn` 方法

## AsyncApp 的诞生

```rust
// src/app/context.rs:237-244
pub fn spawn<AsyncFn, R>(&self, f: AsyncFn) -> Task<R>
where
    T: 'static,
    AsyncFn: AsyncFnOnce(WeakEntity<T>, &mut AsyncApp) -> R + 'static,
    R: 'static,
```

几点与直觉不同：
1. `spawn` 接受 `&self`（不是 `&mut self`）。
2. 闭包参数是 `(WeakEntity<T>, &mut AsyncApp)` —— 第一个参数是**当前实体**的弱引用，让你在 async 块中访问自己。
3. 闭包 trait bound 是 `AsyncFnOnce` — 可以捕获 non-Send 状态的 once 闭包。
4. 返回的 `R` 不需要是 `Future<Output = ()>`，只需要 `'static`。

## AsyncApp 的结构

```rust
// src/app/async_context.rs:17-21
#[derive(Clone)]
pub struct AsyncApp {
    pub(crate) app: Weak<AppCell>,
    pub(crate) background_executor: BackgroundExecutor,
    pub(crate) foreground_executor: ForegroundExecutor,
}
```

`AsyncApp` 实现了 `AppContext` trait（`src/app/async_context.rs:23`），其 `Result<T> = anyhow::Result<T>`：

```rust
impl AppContext for AsyncApp {
    type Result<T> = Result<T>;

    fn new<T: 'static>(
        &mut self,
        build_entity: impl FnOnce(&mut Context<T>) -> T,
    ) -> Self::Result<Entity<T>> {
        let app = self.app.upgrade().context("app was released")?;
        let mut app = app.borrow_mut();
        Ok(app.new(build_entity))
    }
    // ... update_entity, read_entity, update_window 等
}
```

每次调用都先 `self.app.upgrade()`，App 退出后返回 `Err`。

## AsyncApp 的核心操作

### update()：读写访问

```rust
cx.update(|cx: &mut Context<MyView>| {
    // 在主线程上执行
    cx.notify();
}).ok();
```

内部通过 `app.upgrade()?.borrow_mut()` 获取 `&mut App`，构造 `Context`，执行你的闭包。

### update_window()：窗口级操作

```rust
cx.update_window(any_window_handle, |any_view, window, cx: &mut App| {
    // any_view: AnyView — 根视图的类型擦除句柄
    // window: &mut Window
    // cx: &mut App
}).ok();
```

回调签名是 `(AnyView, &mut Window, &mut App)`。

### spawn_in：特定窗口的异步上下文

```rust
cx.spawn_in(&window, |this, mut cx| async move {
    smol::Timer::after(Duration::from_secs(1)).await;
    cx.update(|cx: &mut Context<MyView>| {
        cx.notify();
    }).ok();
}).detach();
```

`Context::spawn_in` 签名（`src/app/context.rs:661-668`）：
- 第一个参数：`&Window`（不是 `AnyWindowHandle`）
- 闭包参数：`(WeakEntity<T>, &mut AsyncWindowContext)`

## spawn 模式

### 模式一：标准 spawn + detach

```rust
cx.spawn(|this, mut cx| async move {
    let data = load_data().await;
    cx.update(|cx: &mut Context<MyView>| {
        this.update(cx, |this, cx| {
            this.data = data;
            cx.notify();
        }).ok();
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
        let task = cx.spawn(|this, mut cx| async move {
            let data = load_data().await;
            cx.update(|cx: &mut Context<Self>| {
                cx.notify();
            }).ok();
        });
        self.loading_task = Some(task);
    }
}
```

取消通过 `drop(task)` 实现——`Task<T>` 的 Drop 会取消内部的 `async_task::Task`。

## 常见错误

**错误：在 update 回调中再次调用 update**

```rust
// 危险
cx.update(|cx: &mut Context<A>| {
    cx.update(|cx: &mut Context<A>| { ... })  // ❌ &mut App 已被借用
}).ok();
```

**错误：忘记 .ok()**

```rust
cx.update(|cx| { ... })  // 返回 Result<()>，必须处理
// 正确
cx.update(|cx| { ... }).ok();
```

**在测试中验证异步行为**

```rust
#[gpui::test]
async fn test_async(cx: &mut TestAppContext) {
    let view = cx.new(|_| MyView::new());
    cx.update_entity(&view, |view, cx| view.start_loading(cx));
    cx.run_until_parked().await;
    // 断言异步结果
}
```

---

**如果你只记住一件事：** `Context::spawn(f)` 的闭包签名是 `AsyncFnOnce(WeakEntity<T>, &mut AsyncApp) -> R`。`AsyncApp::update()` 每次通过 `Weak<AppCell>` 重新借入主线程。`.detach()` 是 fire-and-forget，`drop(task)` 是取消。
