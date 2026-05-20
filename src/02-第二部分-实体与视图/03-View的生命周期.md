# 第 6 章：View 的生命周期

"View"（视图）是 GPUI 中特殊的一类 Entity——实现了 `Render` trait 的实体。本章讨论 View 从创建到销毁的完整生命周期，以及它与普通 Entity 的关键区别。

> 源文件参考：`src/view.rs`、`src/element.rs` 的 `Entity<V: Render>` impl

## 什么时候 Entity 成为 View？

```rust
// 普通实体——不实现 Render
struct Counter {
    count: i32,
}
// Counter 只是数据，不能直接放入元素树

// View——实现了 Render
struct MyView {
    counter: Entity<Counter>,
}
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child("Hello")
    }
}
// MyView 可以作为窗口的根视图放入元素树
```

技术上讲：`Entity<V>` 只有在 `V: Render` 时才实现 `Element` trait。这就是为什么你可以把 `Entity<MyView>` 放入 `div().child()` ——但 `Entity<Counter>` 不行。

## View 的创建

```rust
// 方式一：直接创建
let view = cx.new(|cx| MyView { counter: cx.new(|_| Counter { count: 0 }) });

// 方式二：作为 window 根视图
cx.open_window(WindowOptions::default(), |window, cx| {
    cx.new(|cx| MyView { counter: cx.new(|_| Counter { count: 0 }) })
});
```

`cx.new()` 做了三件事：
1. 分配 `EntityId`（在 slotmap 中插入）
2. 执行你的闭包创建 `T`
3. 推送 `EntityCreated` 效果到队列

## 每帧的渲染周期

View 的渲染通过 `Entity<V: Render>` 实现的 `Element` trait 来驱动（`src/view.rs`）：

```rust
impl<V: Render> Element for Entity<V> {
    // 三阶段委托给 render() 返回的元素树
}
```

每帧流程：

```
1. Window::draw()
     └── 调用根 View 的 render() → 返回元素树
           │
           你的 MyView::render() 运行时：
           - 读取 self 中的状态字段
           - 构建布局（div、元素嵌套）
           - 注册事件处理 (on_click, on_drag, etc.)
           - 返回根元素
           │
     └── 对整棵元素树执行 request_layout / compute / prepaint / paint
```

关键点：**render() 每次调用都在构建一棵新树**。上一帧的树已经被销毁了（Element Arena 在帧末清理）。只有明确保存到 element state 的信息会跨帧存活。

## cached_style：跳过渲染的优化

如果 View 的内容没有变化，重复渲染是浪费。GPUI 提供了 `.cached()` 方法：

```rust
// 在创建 View 时：
let view = cx.new(|_| MyView { ... }).cached(style_refinement);

// 或者
Entity::new(cx, |_| MyView { ... }).cached(default_style())
```

缓存的生效条件（`src/view.rs`）：

```
跳过 render() 的条件（全部满足才跳过）：
  1. cached_style 存在
  2. bounds 与上一帧相同
  3. content_mask 与上一帧相同
  4. text_style 与上一帧相同
  5. View 未被标记为 dirty
  6. Window 未被标记为 dirty
```

当条件满足时，上一帧的 prepaint 和 paint 结果直接复用，`render()` 完全不被调用。这是 GPUI 渲染管线的核心优化。

## View 的销毁

View 在以下情况被销毁：

1. **窗口关闭**：窗口 drop 时，所有关联的视图在清理过程中被移除。
2. **所有句柄释放**：当所有 `Entity<V>` 句柄（不包括 `WeakEntity<V>`）都被 drop 时，entity 从 `EntityMap` 中移除。
3. **显式移除**：`cx.remove_entity(entity_id)`。

常见的意外销毁场景：

```rust
// Bug：view 创建后无人持有强引用
fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    let temp_view = cx.new(|_| TempView {});
    // temp_view 在函数结束时 drop——Entity<V> 的强引用归零
    // TempView 可能在此帧后就被清理
    div() // ...
}
```

## AnyView：类型擦除的视图

`AnyView` 是类型擦除版本的视图句柄（`src/view.rs`）：

```rust
pub struct AnyView {
    entity: AnyEntity,
    render_fn: fn(&AnyView, &mut Window, &mut App) -> AnyElement,
    cached_style: Option<Rc<StyleRefinement>>,
}
```

它的 `render_fn` 是一个**函数指针**（vtable 式的动态分发），在构造时被单态化为具体类型：

```rust
impl AnyView {
    fn new<V: Render>(view: Entity<V>) -> Self {
        AnyView {
            entity: view.into(),
            // 这里生成了一个 V 已知的闭包——类型擦除发生在此处
            render_fn: |any_view, window, app| {
                // 从 AnyView 恢复出 Entity<V>，调用 V::render()
                // ...
            },
            cached_style: None,
        }
    }
}
```

`AnyView` 用于 `Window` 内部——窗口持有 `AnyView` 作为根视图，这样不需要知道根视图的具体类型。

## View 与普通 Entity 的对比

| | Entity\<T: Render\> (View) | Entity\<T\> (普通 Entity) |
|---|---|---|
| **实现 Element** | 是（可放入元素树） | 否（只能作为数据） |
| **render()** | 每帧调用 | 不适用 |
| **典型角色** | UI 组件 | 数据模型、Service |
| **是否需要 Render trait** | 是 | 否 |
| **是否需要 Focusable** | 通常需要（接收键盘输入） | 不需要 |
| **是否需要 EventEmitter** | 通常需要（发出动作） | 可能需要 |

---

**如果你只记住一件事：** View = Entity + Render。View 持有 `Entity<V>` 句柄嵌入元素树，每帧通过 `render()` 重建元素树。只有 element state 中的数据和 Entity 的状态跨帧存活。
