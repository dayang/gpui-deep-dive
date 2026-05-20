# 第 8 章：Element 的三阶段协议

Element 是 GPUI 中定义 UI 的核心 trait。它的设计可能是整个框架中最精妙的部分——用 Rust 的类型系统编码了一个三阶段的状态机，既保证了正确性，又提供了灵活性。

> 源文件参考：`src/element.rs`（完整阅读）、`src/window.rs` 的 `draw()` 方法

## 协议概述

```rust
pub trait Element {
    type RequestLayoutState: 'static;
    type PrepaintState: 'static;

    fn request_layout(
        &mut self,                     // &mut Self
        id: Option<&GlobalElementId>,
        window: &mut Window,
        cx: &mut App,
    ) -> (LayoutId, Self::RequestLayoutState);

    fn prepaint(
        &mut self,                     // &mut Self
        id: Option<&GlobalElementId>,
        bounds: Bounds<Pixels>,        // ← 此时已知尺寸
        request_layout: &mut Self::RequestLayoutState,
        window: &mut Window,
        cx: &mut App,
    ) -> Self::PrepaintState;

    fn paint(
        &mut self,                     // &mut Self
        id: Option<&GlobalElementId>,
        bounds: Bounds<Pixels>,
        request_layout: &mut Self::RequestLayoutState,
        prepaint: &mut Self::PrepaintState,
        window: &mut Window,
        cx: &mut App,
    );
}
```

两个关联类型 `RequestLayoutState` 和 `PrepaintState` 是状态机各阶段间的"传递变量"——它们让你在不同阶段之间传递临时数据，而不需要把一切都变成元素结构体的字段。

## 为什么要三阶段？

答案很简单：**布局需要自底向上，但绘制需要自顶向下**。

```
       A
      / \
     B   C
        / \
       D   E

request_layout 遍历顺序：D → E → C → B → A （子节点先，父节点后）
  原因：父节点的尺寸取决于子节点的尺寸
        （flexbox 需要知道子元素的大小才能分配空间）

prepaint 和 paint 遍历顺序：A → B → C → D → E （父节点先，子节点后）
  原因：父节点设置 content_mask 和 offset，
        子节点在它的范围内绘制
```

这两个方向的遍历无法用一个函数完成——必须拆成三个阶段。

## request_layout 阶段（自底向上）

每个元素向 Taffy 布局引擎注册自己：

```rust
fn request_layout(
    &mut self,
    id: Option<&GlobalElementId>,
    window: &mut Window,
    cx: &mut App,
) -> (LayoutId, Self::RequestLayoutState) {
    // 1. 为子元素递归调用 request_layout
    // 2. 注册当前元素到 Taffy
    let layout_id = window.request_layout(&self.style, child_layout_ids);
    // 3. 返回 layout_id + 可选的中间数据
    (layout_id, MyLayoutState { child_count: n })
}
```

在这个阶段，**元素的尺寸尚未确定**——Taffy 只收集布局约束。你的 `RequestLayoutState` 可以存储任何在布局计算后 `prepaint` 阶段需要的信息。

## compute_layout（过渡阶段）

`request_layout` 全部完成后，`Window` 调用 Taffy 求解所有布局约束：

```rust
window.compute_layout(viewport_size);
// 现在每个 LayoutId 都有对应的 Bounds<Pixels>
```

这是 GPUI 内部的自动步骤，元素不需要参与。

## prepaint 阶段（自顶向下）

此时元素的 `bounds` 已知。元素定义它的命中测试区域（hitbox），并设置子元素的裁剪和偏移：

```rust
fn prepaint(
    &mut self,
    id: Option<&GlobalElementId>,
    bounds: Bounds<Pixels>,            // ← 计算后的尺寸
    request_layout: &mut Self::RequestLayoutState,
    window: &mut Window,
    cx: &mut App,
) -> Self::PrepaintState {
    // 1. 注册 hitbox
    window.insert_hitbox(bounds, self.interactivity);
    // 2. 推送 content_mask（裁剪区域）
    window.push_content_mask(bounds);
    // 3. 推送 element offset
    window.push_element_offset(bounds.origin);
    // 4. 递归调用子元素的 prepaint
    // ...
    MyPrepaintState { sprite_id }
}
```

## paint 阶段（自顶向下）

构建 Scene 的绘制命令：

```rust
fn paint(
    &mut self,
    id: Option<&GlobalElementId>,
    bounds: Bounds<Pixels>,
    request_layout: &mut Self::RequestLayoutState,
    prepaint: &mut Self::PrepaintState,
    window: &mut Window,
    cx: &mut App,
) {
    // 向 Scene 推送图元
    window.paint_quad(PaintQuad {
        bounds,
        background: Some(self.background),
        border: self.border,
        corner_radii: self.corner_radii,
    });
    // 递归调用子元素的 paint
    // ...
}
```

## 类型状态机

`Drawable<E>`（`src/element.rs` 中的内部类型）用 Rust 的枚举编码了这些阶段：

```rust
enum DrawableState<L, P> {
    Start,
    RequestLayout(L),
    LayoutComputed {
        request: L,
        prepaint: P,
    },
    Prepainted {
        prepaint: P,
    },
    Painted,
}
```

这保证了你在编写自定义 Element 时不会意外跳过阶段——类型系统强制你必须先 `request_layout`，再 `prepaint`，再 `paint`。

## Window 的状态栈

在 `prepaint` 和 `paint` 阶段，Window 维护了几个关键的状态栈：

```
element_offset_stack: Vec<Point<Pixels>>     // 当前元素的偏移
content_mask_stack: Vec<ContentMask<Pixels>> // 裁剪区域（嵌套）
text_style_stack: Vec<TextStyleRefinement>   // 继承的文本样式
element_id_stack: Vec<ElementId>             // 当前路径（用于构建 GlobalElementId）
```

每进入一个子元素，相关状态被推入栈；退出子元素时弹出。这实现了 CSS 属性继承和裁剪区域嵌套。

## 自定义 Element 示例：红色圆点

```rust
struct RedDot {
    radius: Pixels,
}

impl Element for RedDot {
    type RequestLayoutState = ();
    type PrepaintState = ();

    fn request_layout(
        &mut self,
        _id: Option<&GlobalElementId>,
        window: &mut Window,
        _cx: &mut App,
    ) -> (LayoutId, Self::RequestLayoutState) {
        let size = Size::new(self.radius * 2.0, self.radius * 2.0);
        let layout_id = window.request_measured_layout(          // ← 固定尺寸布局
            Style::default(),
            size.into(),
        );
        (layout_id, ())
    }

    fn prepaint(
        &mut self,
        _id: Option<&GlobalElementId>,
        bounds: Bounds<Pixels>,
        _request_layout: &mut Self::RequestLayoutState,
        window: &mut Window,
        _cx: &mut App,
    ) -> Self::PrepaintState {
        window.insert_hitbox(bounds, HitTest::default());      // ← 可点击
        ()
    }

    fn paint(
        &mut self,
        _id: Option<&GlobalElementId>,
        bounds: Bounds<Pixels>,
        _request_layout: &mut Self::RequestLayoutState,
        _prepaint: &mut Self::PrepaintState,
        window: &mut Window,
        _cx: &mut App,
    ) {
        window.paint_quad(PaintQuad {
            bounds,
            background: Some(red().into()),
            corner_radii: (self.radius).into(),  // ← 圆形
            ..Default::default()
        });
    }
}
```

但你在日常开发中几乎从不需要写这个——对于 99% 的场景，`div()`、`canvas()` 等内置元素已足够。理解这个协议的价值在于：当你遇到布局 bug、z-index 问题、或命中测试失效时，你能定位到是哪个阶段出了问题。

---

**如果你只记住一件事：** Element 三阶段的本质是遍历顺序不同——`request_layout` 自底向上（子→父），`prepaint` 和 `paint` 自顶向下（父→子）。这是布局依赖方向和绘制继承方向的必然结果。
