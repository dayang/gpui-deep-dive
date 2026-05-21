# 第 3 章：App 的所有权模型

如果你来自 React 或 Vue 的世界，GPUI 的所有权模型可能是需要重新适应的部分。在 JS 生态中，组件状态自然地活在你的组件对象中。在 GPUI 中，"数据在哪"这个问题没有那么简单。

> 源文件参考：`src/app.rs`、`src/app/context.rs`

## App 不是全局变量

先澄清一个常见的直觉错误。当你写：

```rust
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // ...
    }
}
```

你可能会以为 `cx` 是"上下文"，通过它访问一些全局服务。但实际上，**`Context<'a, T>` 的主要部分就是 `&'a mut App`**。它是 App 的独占可变借用加上一个实体 ID。

## 借用树

GPUI 的所有权结构是一棵借用树，根是 `App`：

```
App (通过 RefCell 实现内部可变性)
  │
  ├── Context<'a, Counter> ──┬── &'a mut App
  │                          └── entity_state: WeakEntity<Counter>
  │
  ├── Context<'a, TodoList> ──┬── &'a mut App
  │                           └── entity_state: WeakEntity<TodoList>
  │
  └── ...
```

### RefCell 的关键作用

`App` 外面包了一层 `AppCell`——本质上是一个 `RefCell<App>`。这意味着：

```rust
// AppCell 内部
pub struct AppCell(RefCell<App>);

// 当你调用 App::update() 时
pub fn update<R>(&self, f: impl FnOnce(&mut App) -> R) -> R {
    let mut app = self.0.borrow_mut(); // RefCell::borrow_mut
    f(&mut app)
}
```

在运行时借用检查：同一时刻只能有一个 `&mut App`（但不能有多个 `&App` 同时存在）。如果你在 observer 中尝试再次借入 App——panic。

这就是你在 `cx.spawn()` 中需要 `.update()` 而不能直接访问 `cx` 的原因：**异步块可能运行在另一个线程，而 `&mut App` 不是 `Send`**。

## 为什么 Entity 的数据存在 App 里？

这是 GPUI 最反直觉的设计之一。你以为：

```rust
let entity: Entity<Counter> = cx.new(|_| Counter { count: 0 });
// Counter 在 entity 里？不——
// Counter 在 App.entity_map[entity_id] 里。
```

这种中心化存储有几个好处：

1. **生命周期由框架控制**：你不需要考虑 Counter 的生命周期如何与 UI 协调。App 活着，数据就在。App 销毁，一切清理。
2. **借用检查更灵活**：因为数据在 `EntityMap` 的 slotmap 中（通过 `Box<dyn Any>` 抹除类型），框架可以在运行时"借出"一个实体的 `&mut T` 而不会在编译期就把整个 `App` 锁死。
3. **通知自动化**：当 `GpuiBorrow<T>`（借出 guard）drop 时，框架自动检查是否有变化并触发 notify。

## 借出的机制：Lease 和 GpuiBorrow

当你调用 `entity.update(cx, |counter, cx| ...)` 时，实际的流程是：

```rust
// src/app/entity_map.rs:108-123 — EntityMap::lease()
pub fn lease<T>(&mut self, pointer: &Entity<T>) -> Lease<T> {
    self.assert_valid_context(pointer);
    let mut accessed_entities = self.accessed_entities.borrow_mut();
    accessed_entities.insert(pointer.entity_id);

    // 从 entities map 中物理移出 Box<dyn Any>
    let entity = Some(
        self.entities
            .remove(pointer.entity_id)
            .unwrap_or_else(|| double_lease_panic::<T>("update")),
    );
    Lease {
        entity,
        id: pointer.entity_id,
        entity_type: PhantomData,
    }
}

// src/app/entity_map.rs:125-128 — 归还
pub fn end_lease<T>(&mut self, mut lease: Lease<T>) {
    self.entities.insert(lease.id, lease.entity.take().unwrap());
}
```

`Lease<T>` 实现了 `Deref<Target=T>` 和 `DerefMut`，所以你可以像操作 `&mut T` 一样操作它。操作完毕后 `end_lease()` 将数据放回。

而 `GpuiBorrow<'a, T>`（`src/app.rs:2373-2376`）包装了 `Lease<T>` 并额外持有 `&'a mut App`：

```rust
pub struct GpuiBorrow<'a, T> {
    inner: Option<Lease<T>>,
    app: &'a mut App,
}
```

`GpuiBorrow` 的 `Drop` 会调用 `end_lease()` 归还数据，然后检查实体是否被标记为 dirty——如果是，自动触发 `notify()`。

这个 remove-操作-放回的"租约"模式确保：在你持有 `&mut T` 时，`T` 不在 map 中，所以不会有其他代码同时访问它。Rust 编译期的借用检查无法表达这种动态 slot map 借用，但运行时保证安全。

## Context 的泛型参数 T

`Context<'a, T>` 中的 `T` 不只是"装饰"——它标识了你当前在哪个实体的上下文中操作：

```rust
impl<'a, T: 'static> Context<'a, T> {
    // notify 自动通知"这个"实体
    pub fn notify(&mut self) { ... }

    // emit 自动以"这个"实体为源
    pub fn emit<E: Event>(&mut self, event: E) { ... }

    // observe 订阅"这个"实体的变化
    pub fn observe<W, E>(&mut self, on_notify: W)
    where W: FnMut(&mut E, &mut Context<E>) + 'static,
          E: 'static
    { ... }
}
```

`T` 是"我"，让框架知道 `notify` 应该通知谁，`emit` 应该以谁为 source。

## 所有权对并发的影响

因为 `App` 是 `!Sync`（不可共享），所有对 `App` 的访问必须在主线程。这是设计选择——GUI 框架的主线程模型避免了无处不在的锁。

但后台工作仍然需要。这就是 `BackgroundExecutor` 和 `AsyncApp` 的由来：后台任务在自己的线程运行，完成时通过 `AsyncApp::update()` 回到主线程与 App 交互。

```rust
cx.spawn(|mut cx| async move {
    let data = fetch_from_network().await;     // 后台线程
    cx.update(|window, cx| {                    // 回到主线程
        // 现在可以安全访问 App 和 Window
    }).ok();
})
.detach();
```

> 我们将在第 17-18 章深入讨论异步并发模型。

---

**如果你只记住一件事：** `Context<'a, T>` = `&'a mut App` + `WeakEntity<T>`。数据不在 Entity 句柄里，在 App 的 EntityMap 里。访问数据通过 `Lease<T>` 的租约模式。这是 GPUI 所有权模型的所有核心约束的来源。
