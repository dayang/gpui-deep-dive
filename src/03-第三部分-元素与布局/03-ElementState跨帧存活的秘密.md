# 第 10 章：Element State——跨帧存活的秘密

第 1 章我们提到了"三大寄存器"中的 Element State。本章深入讲解它的内部机制、常见陷阱和最佳实践。

> 源文件参考：`src/window.rs` 的 `element_state` 相关方法、`src/element.rs` 的 `Stateful<E>` 实现

## 存储机制

Element state 存储在 `Window` 的双缓冲 `Frame` 中（`src/window.rs:670, 900-904`）：

```rust
// Frame 中
pub(crate) element_states: FxHashMap<(GlobalElementId, TypeId), ElementStateBox>,

pub(crate) struct ElementStateBox {
    pub(crate) inner: Box<dyn Any>,
    #[cfg(debug_assertions)]
    pub(crate) type_name: &'static str,
}
```

注意三个关键点：
1. **Key 是 `(GlobalElementId, TypeId)` 的二元组**——同一个元素 ID 可以存储多种不同类型的 state，由 `TypeId` 区分。
2. **Value 是 `ElementStateBox`**——包装了 `Box<dyn Any>`，并在 debug 模式下记录类型名。
3. **状态在双缓冲的 `Frame` 之间迁移**——每帧结束时，上一帧中仍被访问的状态迁移到新帧（`window.rs:805-811`）。

`GlobalElementId` 是一个 `SmallVec<[ElementId; 32]>` ——路径向量。在 prepaint/paint 阶段被推入。

## 状态的存取

GPUI 不直接暴露 `element_states` HashMap。元素通过 `Window::with_element_state()` 访问（`src/window.rs:2628-2703`）：

```rust
// Window 的公开 API
pub fn with_element_state<S, R>(
    &mut self,
    global_id: GlobalElementId,
    f: impl FnOnce(Option<&mut S>, &mut Window) -> R,
) -> R
```

这个方法在内部：
1. 以 `(global_id, TypeId::of::<S>())` 为 key 查询 `next_frame.element_states`。
2. 如果没找到，回退查询 `rendered_frame.element_states`（上一帧的状态）。
3. 如果存在，取出 `ElementStateBox`，downcast 为 `S`，传入闭包。
4. 如果不存在，传入 `None`。
5. 闭包执行后，将状态（可能为新创建的）存回 `next_frame.element_states`。

每帧结束时（`Frame::finish()`），`accessed_element_states` 列表中的状态从 `prev_frame` 迁移到 `next_frame`。未被访问的状态自然丢弃。


## 状态丢失的场景

这是 GPUI 程序中最常见的隐形 bug。以下是保证状态丢失的场景：

### 场景一：条件渲染改变 ID 路径

```rust
fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    div()
        .when(self.show_sidebar, |d| {
            d.child(div().id("sidebar").child(
                // 这个滚动区域的状态在 sidebar 隐藏时丢失
                div().id("scroll").child(/* ... */)
            ))
        })
}
```

### 场景二：元素顺序改变

```rust
// 第一帧
div().child(div().id("A")).child(div().id("B"))
// 路径：[root, A, ...],  [root, B, ...]

// 第二帧（顺序变了）
div().child(div().id("B")).child(div().id("A"))
// 路径：[root, B, ...]  ← 和之前不同！即使每个元素都有 ID
// 原来 A 的状态匹配 [root, A, ...]，但现在路径变了 → 找不到 → 重置
```

### 场景三：父元素 ID 改变

```rust
// 第一帧
div().id("outer-v1").child(div().id("inner"))
// 路径：[outer-v1, inner]

// 第二帧
div().id("outer-v2").child(div().id("inner"))
// 路径：[outer-v2, inner] ← 不同！inner 的状态丢失
```

### 场景四：列表中的元素未用 key 标识

```rust
// 危险：没有稳定的 ID
for item in &self.items {
    div().child(div().child(format!("{}", item.name)))
    // 每帧可能位置不同，状态路径也变了
}

// 安全：用稳定的 key
for item in &self.items {
    div().id(item.id).child(div().child(format!("{}", item.name)))
    // GlobalElementId 基于 item.id，跨帧稳定
}
```

## 规避策略

### 策略一：用 visible/overflow 代替条件渲染

```rust
// 不好——状态在折叠时丢失
if expanded {
    div().id("content").child(/* ... */)
}

// 好——状态保持
div()
    .id("content")
    .visible(expanded)     // 或 .overflow_hidden() + .h(if expanded { auto } else { px(0) })
    .child(/* ... */)
```

### 策略二：稳定的 ID，不依赖位置

给需要保存状态的元素分配唯一且稳定的 `id`：

```rust
// 类型 + 稳定的标识符 = 唯一 ID
ElementId::Name("counter-value".into())
ElementId::Uuid(uuid)  // 需要生成并持久化
ElementId::Integer(index)  // 列表下标——仅在不重新排序时稳定
```

### 策略三：理解状态的生命周期，而非对抗它

Element state 设计为**短命的 UI 装饰状态**。如果你需要真正的数据持久化和跨组件共享——用 Entity。

## Element State vs Entity：选择指南

```
                              Element State        Entity
需要跨组件共享？              ❌ 很难              ✅ 天然
需要响应式通知？              ❌ 不支持            ✅ notify/emit
需要序列化/持久化？          ❌ 类型擦除，不易     ✅ 可以实现
数据量大（>几个字段）？      ⚠️ 可行但不便        ✅ 结构体字段
只是 UI 装饰（滚动、折叠）？ ✅ 完美匹配          ⚠️ 过度设计
```

实践中的一个好规则：**每个状态字段先问"别的组件需要知道这个值吗"。如果不需要——放 element state。如果需要——放 Entity。**

---

**如果你只记住一件事：** Element state 的 key 是 ID 路径（GlobalElementId），路径上的任何改变（父 ID 变、顺序变、条件渲染导致元素消失）都会导致状态丢失。这不是 bug，是设计——逼你把重要数据放在 Entity 里。
