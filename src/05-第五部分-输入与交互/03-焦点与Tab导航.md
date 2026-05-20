# 第 16 章：焦点与 Tab 导航

焦点（Focus）决定了哪个元素接收键盘输入。在一个有几十个可选元素的窗口中，同一时刻只有一个被聚焦。

> 源文件参考：`src/tab_stop.rs`、`src/window.rs` 的焦点管理部分

## 焦点系统架构

```
Window
  └── focus: Option<FocusId>
        │
        └── FocusMap (全局): SlotMap<FocusId, FocusState>
              ├── FocusId(0) → { parent: _, order: 0, handle: Arc<...> }
              ├── FocusId(1) → { parent: _, order: 1, handle: Arc<...> }
              └── ...

FocusHandle {
    id: FocusId,
    map: Arc<RwLock<FocusMap>>,
}

WeakFocusHandle {
    id: FocusId,
    map: Weak<RwLock<FocusMap>>,
}
```

### 三种焦点相关类型

| 类型 | 作用 | 引用计数 |
|------|------|---------|
| `FocusHandle` | 赋予元素可聚焦能力 | `Arc`（不释放） |
| `WeakFocusHandle` | 观察焦点但不阻止 | `Weak` |
| `FocusId` | 焦点 map 中的 integer key | 不参与引用计数 |

### FocusHandle 的创建

```rust
let focus_handle = cx.focus_handle();
// focus_handle 现在拥有一个全局唯一的 FocusId
// 它的引用计数阻止 FocusMap 中的 entry 被清理
```

销毁时：`FocusHandle` drop → 引用计数减一 → 当 `Arc<FocusState>` 强引用归零时，如果它是当前焦点，焦点自动清除。

## 聚焦一个元素

```rust
// 方式一：显式调用
focus_handle.focus(cx);

// 方式二：点击时自动聚焦（如果元素设置了 focus_handle）
div()
    .track_focus(&focus_handle)
    .on_click(cx.listener(|this, _event, cx| {
        // 点击后 focus_handle 自动成为焦点
    }))
```

`cx.focus(handle)` 内部：
1. 验证 `handle` 对应的实体还活着
2. 设置 `Window.focus = Some(handle.id)`
3. 通知旧的焦点元素（`blur`）和新的焦点元素（`focus`）

## Focusable trait

对于需要完整键盘控制的 View：

```rust
pub trait Focusable {
    fn focus_handle(&self, cx: &App) -> FocusHandle;
}

// 常见实现
impl Focusable for MyView {
    fn focus_handle(&self, cx: &App) -> FocusHandle {
        self.focus_handle.clone()  // 存储 FocusHandle 作为字段
    }
}
```

`Focusable` 被 `ManagedView`（`Focusable + EventEmitter<DismissEvent> + Render`）组合使用。实现了 `Focusable` 的 View 能够：
- 接收 Tab 导航
- 触发 `Escape` 关闭（DismissEvent）
- 被 dispatch tree 自动识别为可聚焦节点

## Tab 导航

GPUI 实现了完整的 Tab 键导航：

```rust
// 前向导航（Tab）
window.focus_next(cx);

// 后向导航（Shift+Tab）
window.focus_prev(cx);
```

底层实现基于 `focusable` 元素的 tab_index 排序。

### tab_index 与 tab_stop

```rust
div()
    .tab_index(TabIndex::Focusable { order: 0 })  // 参与 Tab 导航
    .tab_stop(true)  // 快捷键：UI 预览中可设为 false 以跳过装饰性元素
```

Tab 顺序由 `order` 决定——order 相同的按 DOM 顺序。这让你可以精确控制键盘用户的导航路径。

## 焦点与键盘事件分发

```
按键事件
  ├── 如果存在焦点元素：
  │     └── dispatch_action 沿焦点元素的 dispatch 路径
  │         （Capture → Bubble）
  │
  └── 如果不存在焦点元素：
       └── dispatch_action 沿窗口根路径
           → 可能触发全局 on_app_action 处理器
```

## 焦点与样式

焦点状态通常需要通过视觉反馈：

```rust
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let is_focused = self.focus_handle.is_focused(cx);

        div()
            .track_focus(&self.focus_handle)
            .when(is_focused, |d| {
                d.border_color(blue())
            })
    }
}
```

`is_focused()` 查询全局焦点状态——如果 `Window.focus == Some(self.focus_handle.id)` 则返回 true。这是一个廉价查询，可以在 `render()` 中放心使用。

## 焦点丢失与恢复

```
场景一：聚焦元素被移除
  Window.focus 被自动设置为 None

场景二：另一个窗口被激活
  系统通知 Window 失活
  焦点仍然保持（但窗口不响应键盘事件）

场景三：模态对话框打开
  焦点被"借走"给对话框
  对话框关闭时自动恢复
```

GPUI 不自动保存/恢复焦点历史栈。如果你需要实现"关闭对话框后恢复之前的焦点"，你需要自己管理：

```rust
let previous_focus = window.focused(cx);
// 打开对话框……
// 关闭后：
if let Some(handle) = previous_focus {
    handle.focus(cx);
}
```

## 最佳实践

1. **每个可交互元素绑定一个 FocusHandle**：即使你不在 `render()` 中使用，也为将来的键盘操作留出可能。
2. **tab_index 给关键导航元素**：不是每个 div 都需要 tab_index，只为编辑区域、按钮、列表项等设置。
3. **WeakFocusHandle 用于观察者**：避免循环引用。
4. **先构建 FocusHandle 再构建 Entity**：在 `cx.new()` 中使用 Reservation 模式时，先创建 FocusHandle。

---

**如果你只记住一件事：** FocusHandle 是焦点的句柄（引用计数的 ID）。`is_focused(cx)` 查询谁有焦点，`focus(cx)` 设置焦点。Tab 导航按 tab_index.order 排序所有 Focusable 元素。
