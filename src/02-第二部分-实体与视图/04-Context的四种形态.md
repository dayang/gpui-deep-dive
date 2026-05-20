# 第 7 章：Context 的四种形态

如果你翻看 GPUI 的源码，会反复看到四个相似的名称：`Context`、`AsyncApp`、`AsyncWindowContext`、`TestAppContext`。它们都提供了访问 App 和实体的方法，但适用场景完全不同。选错类型是最常见的编译错误来源之一。

> 源文件参考：`src/app/context.rs`、`src/app/async_context.rs`、`src/app/test_context.rs`

## 全貌

```
                拥有数据      生命周期      Send    使用场景
Context<'a, T>  否（借用）    'a (~1ms)    ❌     render(), 事件处理
AsyncApp        是(Weak)      'static      ✅     spawn(async {})
AsyncWindowCtx  是(Weak)      'static      ✅     spawn(async {}) + 需要窗口信息
TestAppContext  是(自有)      'static      ✅     #[gpui::test]
```

## Context\<'a, T\>：短暂的借用

```rust
fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    //  cx 生命周期 = 此次 render() 调用
    // 不能 escape 到闭包外
    // 不能跨 .await
}
```

`Context<'a, T>` 的生命周期 `'a` 就是你对 `App` 的借用期。实际上它是一个**透明的新类型包装**：

```rust
pub struct Context<'a, T> {
    app: &'a mut App,
    entity: WeakEntity<T>,
}
```

它的所有方法（`notify()`、`emit()`、`new()` 等）都是对底层 `app` 的转发。

### 局限性

```rust
// 编译错误：cx 不能跨 .await
cx.spawn(|mut cx| async move {
    tokio::time::sleep(Duration::from_secs(1)).await;
    cx.notify(); // ❌ cx 不是 Send
});
```

解决方案：GPUI 自己的 `cx.spawn` 在背后做了类型转换，传入闭包的不是 `Context` 而是 `AsyncApp`。

```rust
cx.spawn(|mut cx: AsyncApp| async move {
    // ✅ cx 现在是 AsyncApp (Send + 'static)
    smol::Timer::after(Duration::from_secs(1)).await;
    cx.update(|cx: &mut Context<SenderView>| {
        cx.notify();
    }).ok();
}).detach();
```

## AsyncApp：跨 await 的上下文

```rust
pub struct AsyncApp {
    app: Weak<AppCell>,      // 弱引用 App
    background_executor: BackgroundExecutor,
    foreground_executor: ForegroundExecutor,
}
```

`AsyncApp` 是 `Clone` + `Send` + `'static`，设计用于在 async block 中跨 `.await` 持有。它通过 `Weak<AppCell>` 访问 App：

```rust
impl AsyncApp {
    pub fn update<R>(
        &mut self,
        f: impl FnOnce(&mut App) -> R + Send
    ) -> Result<R> {
        let app = self.app.upgrade()?;    // 获取 Arc<AppCell>
        let result = app.update(f)?;      // 借入 App
        Ok(result)
    }
}
```

`update()` 返回 `Result` 而不是 `T`——因为 `AsyncApp` 活得比 `App` 久是可能的（App 关闭后 async 任务还在运行）。

### update() vs read_with()

```rust
cx.update(|cx: &mut Context<MyView>| {
    // 可以修改 App、EntityMap、任何实体
    cx.notify();
})?;

let value = cx.read_with(|cx: &Context<MyView>, this: &MyView| {
    // 只读访问
    this.some_field
})?;
```

`update` 提供 `&mut` 访问，`read_with` 只提供 `&` 访问。如果有多个并发 async 任务需要读取同一个实体，`read_with` 可以并行执行（内部共享锁）。

## AsyncWindowContext：需要窗口句柄时

```rust
pub struct AsyncWindowContext {
    app: AsyncApp,
    window: AnyWindowHandle,
}
```

`AsyncWindowContext` 是 `AsyncApp` + 一个窗口句柄。当你需要在 async 任务中操作特定窗口时使用。

```rust
cx.spawn_in(window_handle, |mut cx: AsyncWindowContext| async move {
    smol::Timer::after(Duration::from_secs(1)).await;
    cx.update(|window, cx: &mut Context<MyView>| {
        // window 可用——调整窗口大小、位置等
        let bounds = window.bounds();
    }).ok();
}).detach();
```

## TestAppContext：测试专用

```rust
pub struct TestAppContext {
    app: AppCell,   // 自有所有权，不是 Weak
    // 拥有模拟时钟、模拟输入发射器
}
```

`TestAppContext` 直接拥有 `AppCell`，不需要 `Weak`。它对已关闭的 App 访问会 panic（而不是返回 `Result::Err`）——积极暴露 bug。

```rust
#[gpui::test]
async fn test_counter(cx: &mut TestAppContext) {
    let view = cx.new(|_| Counter { count: 0 });
    // 模拟点击、键盘输入、时间流逝
    cx.dispatch_keystroke("enter");
    cx.run_until_parked(); // 等待所有异步任务完成
    let count = cx.read_entity(&view, |view, _| view.count);
    assert_eq!(count, 1);
}
```

## 转换关系

```
Context<'a, T>
  │ .to_async() ──→ AsyncApp
  │ .to_async() + window ──→ AsyncWindowContext
  │
  └── cx.spawn(|mut cx: AsyncApp| ...)
                      │
                      └── cx.update(|cx: &mut Context<MyView>| ...) → 回到 Context
```

`Context::to_async()` 创建了从当前借用上下文到 `'static` 上下文的桥接。`AsyncApp::update()` 又提供了反向的桥接——从 `AsyncApp` 重新借回 `Context`。

## 常见编译错误成因

**错误：`cx` does not live long enough**

你试图把 `Context<'a, T>` 传给需要 `'static` 的闭包。改用 `cx.to_async()` 或 `cx.spawn()`。

**错误：`cx` is not `Send`**

你试图在 `async move {}` 中保持 `Context<'a, T>`。改用 `cx.spawn()` ——它会自动转为 `AsyncApp`。

**错误：`AsyncApp::update` returns `Result`**

你忘了 handle `Ok/Err`。最简单的方式是在你不关心 App 是否还活着的场景（通常如此）写 `cx.update(|cx| ...).ok();`。

---

**如果你只记住一件事：** `Context<'a, T>` 是借用（短命、不同步），`AsyncApp` 是弱引用（长命、可同步）。二者的转换通过 `to_async()` 和 `update()` 完成。
