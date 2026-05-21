# 第 16 章：焦点与 Tab 导航

焦点（Focus）决定了哪个元素接收键盘输入。

> 源文件参考：`src/window.rs`（FocusHandle/FocusMap）、`src/tab_stop.rs`、`src/key_dispatch.rs`

## 焦点系统架构

```rust
// src/window.rs:228-233
pub(crate) type FocusMap = RwLock<SlotMap<FocusId, FocusRef>>;

pub(crate) struct FocusRef {
    pub(crate) ref_count: AtomicUsize,
    pub(crate) tab_index: isize,
    pub(crate) tab_stop: bool,
}
```

`FocusRef` 存储于 `FocusMap` 中，字段含义：
- **`ref_count`**：引用计数（`FocusHandle` 的 clone 递增，drop 递减）。计数归零后焦点 entry 可回收。
- **`tab_index`**：Tab 键导航的排序索引。
- **`tab_stop`**：是否参与 Tab 导航。

### FocusHandle / WeakFocusHandle

```rust
// src/window.rs:266-273
pub struct FocusHandle {
    pub(crate) id: FocusId,
    handles: Arc<FocusMap>,
    pub tab_index: isize,
    pub tab_stop: bool,
}

// src/window.rs:405-408
#[derive(Clone)]
pub struct WeakFocusHandle {
    id: FocusId,
    handles: Weak<FocusMap>,
}
```

`FocusHandle` 包含 `tab_index` 和 `tab_stop` 字段的副本，所以你可以直接读写它们而不需要每次都访问共享的 `FocusRef`。`.tab_index(n)` 和 `.tab_stop(true)` 是 builder 方法，同时更新 handle 自身和共享的 `FocusRef`。

### 创建和聚焦

```rust
let focus_handle = cx.focus_handle();  // 创建 FocusHandle

// 聚焦
window.focus(&focus_handle);           // Window::focus(&FocusHandle)
// 注意：不是 cx.focus(handle)，而是 window.focus(&handle)
```

`FocusHandle` 实现了 `Clone`——clone 时递增 `FocusRef.ref_count`。Drop 时递减引用计数。

## Tab 导航

```rust
// src/tab_stop.rs
type TabIndex = isize;  // 不是 enum，是简单类型别名

// src/window.rs:1413-1431
impl Window {
    pub fn focus_next(&mut self) { ... }
    pub fn focus_prev(&mut self) { ... }
}
```

**`TabIndex` 就是 `isize` 的类型别名**——没有任何 variant。它不是 enum。在 builder API 中：

```rust
div()
    .tab_index(0)       // 设置 tab 顺序
    .tab_stop(true)     // 参与 tab 导航
```

底层通过 `FocusHandle::tab_index(mut self, index: isize) -> Self` 实现。

Tab 导航由 `TabStopMap`（`src/tab_stop.rs:11-16`）管理，它在内部维护一个 `SumTree<TabStopNode>` 以支持高效的顺序遍历。

## 焦点与键盘事件分发

```
按键事件
  ├── 如果存在焦点元素：
  │     └── dispatch_path = dispatch_tree.dispatch_path(
  │           dispatch_tree.focusable_node_id(focused.id)
  │         )
  │         └── 沿 dispatch_path 分发（Capture → Bubble）
  │
  └── 如果不存在焦点元素：
       └── dispatch_path = [root_node_id]
```

## 焦点与样式

```rust
impl Render for MyView {
    fn render(&mut self, window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        let is_focused = self.focus_handle.id.is_focused(window);

        div()
            .track_focus(&self.focus_handle)
            .when(is_focused, |d| {
                d.border_color(blue())
            })
    }
}
```

`FocusId::is_focused(&self, window: &Window) -> bool` 查询 `window.focus == Some(*self)`。是廉价查询。

## Focusable trait

```rust
// src/window.rs:440-443
pub trait Focusable: 'static {
    fn focus_handle(&self, cx: &App) -> FocusHandle;
}
```

`'static` bound 是必须的——FocusHandle 不持有对 entity 的引用，所以 entity 类型必须可以独立存在。

## 焦点丢失与恢复

```
场景一：聚焦元素被移除
  Window.focus 被自动设置为 None

场景二：另一个窗口被激活
  系统通知 Window 失活
  焦点仍保持（但窗口不响应键盘）

场景三：模态对话框打开
  焦点被"借走"给对话框
  你需要手动管理恢复
```

## 最佳实践

1. **每个可交互元素绑定一个 FocusHandle**。
2. **`tab_index` 给关键导航元素**：不是每个 div 都需要，只为编辑区域、按钮、列表项设置。
3. **WeakFocusHandle 用于观察者**：避免循环引用。

---

**如果你只记住一件事：** `FocusRef` 存储 `(ref_count, tab_index, tab_stop)` 三个字段。`TabIndex = isize`（类型别名）。`FocusHandle` 同时持有这些字段的副本和 `Arc<FocusMap>`。Tab 导航由 `TabStopMap` 的 `SumTree` 管理。
