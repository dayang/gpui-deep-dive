# 第 4 章：EntityMap 内部机制

第 3 章告诉我们数据存在 `App.entity_map` 中。本章深入这个数据结构，理解 `Entity<T>` 到底是什么、引用计数如何工作、以及 GPUI 如何安全地借出数据。

> 源文件参考：`src/app/entity_map.rs`（完整阅读）

## 数据结构

```rust
pub struct EntityMap {
    entities: SecondaryMap<EntityId, Option<Box<dyn Any>>>,
    ref_counts: Arc<RwLock<SlotMap<EntityId, EntityRefCounts>>>,
    // ...
}

pub struct EntityRefCounts {
    entity_ref_count: usize,    // Entity<T> 的存活数
    weak_ref_count: usize,      // WeakEntity<T> 的存活数
    any_ref_count: usize,       // AnyEntity 的存活数
}
```

三层结构：

1. **`entities: SecondaryMap`**：实际的 `T` 值存在这里。当你调用 `cx.new()` 时，`T` 被装箱并插入。
2. **`ref_counts`**：独立的 `RwLock<SlotMap>`，追踪三种句柄的引用计数。
3. **`reservations`**：预留槽位（`Reservation<T>`），用于先占位后填入。

引用计数和实际数据分离的设计是为了**减少锁竞争**——`clone()` / `drop()` 句柄时只需要写引用计数（Hash + 原子操作），不需要碰实体数据。

## `Entity<T>` 的解剖

```rust
pub struct Entity<T> {
    pub(crate) entity_id: EntityId,
    pub(crate) ref_counts: Arc<Weak<RwLock<SlotMap<EntityId, EntityRefCounts>>>>,
    _phantom: PhantomData<T>,  // 仅用于类型安全，运行时不占空间
}
```

`Entity<T>` 很小——两个指针大小（`EntityId` 是 32 位 key，`Arc` 是一个指针）。clone 它是廉价的。

`PhantomData<T>` 不占运行时空间，但确保你无法把 `Entity<Counter>` 赋值给 `Entity<TodoList>`。

### 为什么引用计数需要 `Arc<Weak<RwLock<...>>>`？

```
Entity<Counter> ──Arc<Weak<...>>──→ RwLock<SlotMap<...>>
                                        ↑
Entity<Counter> （另一个 clone）─────────┘
                                        ↑
WeakEntity<Counter> ──Weak<...>─────────┘
```

- **`Arc`**（在 `Entity` 这边）：`Entity<T>` 的每个 clone 都持有一个 `Arc`，drop 时 Arc 减引用计数。
- **`Weak`**（在 `ref_counts` 这边）：`WeakEntity<T>` 只持有一个 `Weak`，不阻止释放。
- **`RwLock`**：允许多个 `clone()` 同时读。

当最后一个 `Entity<T>` 被 drop 后，`entity_ref_count` 归零，框架清理 `EntityMap` 中的对应条目。但如果有 `WeakEntity<T>` 存活，它 `upgrade()` 时会得到 `None`。

## GpuiBorrow：RAII 租约

```rust
pub(crate) struct GpuiBorrow<'a, T: 'static> {
    entity_id: EntityId,
    any_value: Box<dyn Any>,
    entity_map: &'a mut EntityMap,
    became_dirty: bool,
}
```

当你调用 `update_entity()` 时：

1. `Box<dyn Any>` 从 `EntityMap` **物理移出**（不再是 `Option::Some`，变成 `Option::None`）。
2. `downcast_mut::<T>()` 获取 `&mut T`。
3. 你的闭包执行。
4. Drop 时，`Box<dyn Any>` **放回** EntityMap。
5. 如果设置了 `became_dirty`，自动调用 `notify()`。

这种"移除-租借-归还"模式是 GPUI 实现动态借用检查的秘诀——**数据在你使用时不在 map 中，所以不可能有双重借用**。

## 三种句柄对比

```rust
// 强引用——阻止实体释放
let handle: Entity<Counter> = cx.new(|_| Counter { count: 0 });

// 弱引用——不阻止释放，用前需 upgrade
let weak: WeakEntity<Counter> = handle.downgrade();

// 类型擦除——可以存储不同类型的实体
let any: AnyEntity = handle.into();
```

| | Entity\<T\> | WeakEntity\<T\> | AnyEntity |
|---|---|---|---|
| **类型** | 保留 T | 保留 T | 类型擦除 |
| **阻止释放** | 是 | 否 | 是 |
| **访问数据** | 直接 update/read | 需 upgrade → Option\<Entity\<T\>\> | 需 downcast |
| **典型用途** | View handle, 字段存储 | 观察者回调（避免循环引用） | 集合/列表存储 |

## Reservation：解决循环引用

两个实体互相引用是常见需求：

```rust
struct Parent {
    child: Option<Entity<Child>>, // 需要引用 Child
}
struct Child {
    parent: Option<WeakEntity<Parent>>, // 弱引用 Parent，避免循环
}
```

但如果 `Parent` 的构造需要 `Child` 的句柄，`Child` 的构造又需要 `Parent` 的句柄呢？Reservation 解决了这个先有鸡还是先有蛋的问题：

```rust
// 1. 预留两个槽位
let parent_reservation = cx.reserve::<Parent>();
let child_reservation = cx.reserve::<Child>();

// 2. 此时 Entity 还不存在，但 EntityId 已分配
let parent_entity = parent_reservation.entity(); // Entity<Parent>
let child_entity = child_reservation.entity();   // Entity<Child>

// 3. 用预留的 Entity 插入实际数据
cx.insert(parent_reservation, Parent {
    child: Some(child_entity.clone()),
});
cx.insert(child_reservation, Child {
    parent: Some(parent_entity.downgrade()),
});
```

## 实体何时被清理？

一个实体从 EntityMap 中移除需要同时满足：

1. 所有 `Entity<T>` 句柄被 drop（`entity_ref_count == 0`）
2. 所有 `AnyEntity` 句柄被 drop（`any_ref_count == 0`）
3. `WeakEntity<T>` 不影响（`weak_ref_count` 可以非零）

清理在 `flush_effects` 过程中发生，不是在句柄 drop 时立即发生。

---

**如果你只记住一件事：** `Entity<T>` 是一个轻量句柄（ID + 引用计数），数据在 `EntityMap` 的 slotmap 中。借出数据时采用 "移除-操作-归还" 模式，Rust 编译期看不到这个正确性，但 GPUI 在运行时保证了安全。
