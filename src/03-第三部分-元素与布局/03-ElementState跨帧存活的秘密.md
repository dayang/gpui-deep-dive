# 第 10 章：Element State——跨帧存活的秘密

第 1 章我们提到了"三大寄存器"中的 Element State。本章深入讲解它的内部机制、常见陷阱和最佳实践。

> 源文件参考：`src/window.rs` 的 `element_state` 相关方法、`src/element.rs` 的 `Stateful<E>` 实现

## 存储机制

```rust
// Window 中
pub(crate) struct Window {
    // ...
    element_state: FxHashMap<GlobalElementId, Box<dyn Any>>,
    // ...
}
```

`element_state` 就是一个 HashMap，key 是 `GlobalElementId`，value 是类型擦除的任意值。

`GlobalElementId` 是一个 `SmallVec<[ElementId; 32]>` ——路径向量。在 prepaint 阶段被推入：

```
Window::prepaint() 在遍历树时：
  进入 div#root → push "root" → 路径 = [root]
    进入 div#sidebar → push "sidebar" → 路径 = [root, sidebar]
      进入 button#save → push "save" → 路径 = [root, sidebar, save]
        → 以 [root, sidebar, save] 查找/存储 element state
      退出 button#save → pop
    退出 div#sidebar → pop
  退出 div#root → pop
```

## 状态的存取

元素的实现中典型访问模式：

```rust
// 读取已有状态（跨帧生命）
if let Some(state) = window.element_state::<MyState>(&element_id) {
    // 使用上一帧保存的状态
}

// 存储状态
window.set_element_state::<MyState>(&element_id, MyState { ... });
```

`Stateful<E>` 封装了这种模式，提供更便捷的 API：

```rust
div().id("scroll").child(
    Stateful::new(|element_id, window, _cx| {
        // 尝试恢复状态
        let state = window.element_state::<ScrollState>(&element_id)
            .cloned()
            .unwrap_or_default();
        // ... 使用 state
    })
    .on_element_state_changed(|element_id, state, window, _cx| {
        // 状态更新回调
    })
)
```

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
