# 第 4 章：EntityMap 内部机制

第 3 章告诉我们数据存在 `App.entity_map` 中。本章深入这个数据结构，理解 `Entity<T>` 到底是什么、引用计数如何工作、以及 GPUI 如何安全地借出数据。

> 源文件参考：`src/app/entity_map.rs`（完整阅读）

## 数据结构

```rust
// src/app/entity_map.rs:57-68
pub(crate) struct EntityMap {
    entities: SecondaryMap<EntityId, Box<dyn Any>>,
    pub accessed_entities: RefCell<FxHashSet<EntityId>>,
    ref_counts: Arc<RwLock<EntityRefCounts>>,
}

struct EntityRefCounts {
    counts: SlotMap<EntityId, AtomicUsize>,
    dropped_entity_ids: Vec<EntityId>,
    #[cfg(any(test, feature = "leak-detection"))]
    leak_detector: LeakDetector,
}
```

三层结构：

1. **`entities: SecondaryMap<EntityId, Box<dyn Any>>`**：实际的数据存储。`SecondaryMap` 是与 `SlotMap` 共享 key 空间的关联映射。每个实体以 `Box<dyn Any>` 形式存在。
2. **`ref_counts: Arc<RwLock<EntityRefCounts>>`**：引用计数层。这里的 `EntityRefCounts` 内部是一个 `SlotMap<EntityId, AtomicUsize>`——每个 `EntityId` 对应一个 `AtomicUsize` 计数器。它还维护了 `dropped_entity_ids` 列表，记录了引用计数归零但尚未清理的实体。
3. **`accessed_entities: RefCell<FxHashSet<EntityId>>`**：当前帧中被访问过的实体集合，用于脏检测（dirty checking）。

引用计数和实际数据分离的设计有两个好处：
- `clone()` / `drop()` 句柄时只需要原子操作更新 `counts` 中的计数，不需要碰实体数据。
- `EntityRefCounts` 被 `Arc<RwLock<>>` 包裹，意味者多个线程可以同时读引用计数（例如并行 clone 句柄）。

## `Entity<T>` 的解剖

```rust
// src/app/entity_map.rs:376-382
#[derive(Deref, DerefMut)]
pub struct Entity<T> {
    #[deref]
    #[deref_mut]
    pub(crate) any_entity: AnyEntity,
    pub(crate) entity_type: PhantomData<fn(T) -> T>,
}
```

`Entity<T>` 本身只有两个字段：一个 `AnyEntity`（类型擦除的句柄）+ 一个编译期幻影类型。真正的数据都在 `AnyEntity` 中：

```rust
// src/app/entity_map.rs:221-227
pub struct AnyEntity {
    pub(crate) entity_id: EntityId,
    pub(crate) entity_type: TypeId,
    entity_map: Weak<RwLock<EntityRefCounts>>,
    #[cfg(any(test, feature = "leak-detection"))]
    handle_id: HandleId,
}
```

- **`entity_id: EntityId`**：slotmap key，定位 `EntityMap.entities` 中的数据。
- **`entity_type: TypeId`**：运行时类型标记，用于 `downcast`。
- **`entity_map: Weak<RwLock<EntityRefCounts>>`**：弱引用指向 `EntityRefCounts`。在 `clone()` 时通过 `upgrade()` 获取计数 map 并递增；在 `drop()` 时递减计数。

`PhantomData<fn(T) -> T>` 不占运行时空间，但保证 `Entity<Counter>` 和 `Entity<TodoList>` 是不同类型——加上 `fn(T) -> T` 的写法同时使类型对协变（covariant），这对类型系统很重要。

### Clone 和 Drop 的引用计数机制

**Clone**（`src/app/entity_map.rs:463-470`）：
```rust
impl<T> Clone for Entity<T> {
    fn clone(&self) -> Self {
        Self {
            any_entity: self.any_entity.clone(),  // 委托给 AnyEntity::clone
            entity_type: self.entity_type,
        }
    }
}
```

`AnyEntity::clone()` 内部通过 `self.entity_map.upgrade()` 获取 `EntityRefCounts`，找到 `entity_id` 对应的 `AtomicUsize`，执行 `fetch_add(1, SeqCst)`。插入断言防止引用已归零的实体。

**Drop**（`src/app/entity_map.rs:307-332`）：
```rust
impl Drop for AnyEntity {
    fn drop(&mut self) {
        if let Some(entity_map) = self.entity_map.upgrade() {
            let entity_map = entity_map.upgradable_read();
            let count = entity_map.counts.get(self.entity_id)
                .expect("detected over-release of a handle.");
            let prev_count = count.fetch_sub(1, SeqCst);
            assert_ne!(prev_count, 0, "Detected over-release of a entity.");
            if prev_count == 1 {
                // 我们是最后一个引用，可以标记为待清理
                let mut entity_map = RwLockUpgradableReadGuard::upgrade(entity_map);
                entity_map.dropped_entity_ids.push(self.entity_id);
            }
        }
    }
}
```

Drop 时先递减 `AtomicUsize`。如果减到 0（说明 `prev_count == 1`），将 `entity_id` 推入 `dropped_entity_ids`，等待 `EntityMap::take_dropped()` 在下一轮 `flush_effects` 中真正清理。

## `Lease<T>`：RAII 租约

```rust
// src/app/entity_map.rs:189-215
pub(crate) struct Lease<T> {
    entity: Option<Box<dyn Any>>,
    pub id: EntityId,
    entity_type: PhantomData<T>,
}
```

`EntityMap::lease()` 从 `entities` map 中**物理移出** `Box<dyn Any>`（源码第 113-116 行），存入 `Lease`。`Lease` 通过 `Deref`/`DerefMut` 暴露 `&T` / `&mut T`。操作完毕后调用 `EntityMap::end_lease()` 将数据放回。

`Lease` 的 `Drop` 实现有个保护：如果 `end_lease` 没被调用而 `Lease` 被 drop（正常路径不该发生），它会 panic。

## GpuiBorrow：对外暴露的租约包装

```rust
// src/app.rs:2373-2376
pub struct GpuiBorrow<'a, T> {
    inner: Option<Lease<T>>,
    app: &'a mut App,
}
```

`GpuiBorrow` 包装了 `Lease<T>` 并持有 `&mut App`。它的 `Drop` 实现会：
1. 调用 `end_lease` 将数据放回 `EntityMap`。
2. 如果实体被标记为 dirty，自动调用 `notify()`。

这种"移除-租借-归还"模式是 GPUI 实现动态借用检查的秘诀——**数据在你使用时不在 map 中，所以不可能有双重借用**。

## 三种句柄对比

```rust
// 强引用——阻止实体释放
let handle: Entity<Counter> = cx.new(|_| Counter { count: 0 });

// 弱引用——不阻止释放，用前需 upgrade
let weak: WeakEntity<Counter> = handle.downgrade();

// 类型擦除——可以存储不同类型的实体
let any: AnyEntity = handle.into_any();
```

| | Entity\<T\> | WeakEntity\<T\> | AnyEntity |
|---|---|---|---|
| **类型** | 保留 T | 保留 T | 类型擦除 |
| **阻止释放** | 是 | 否 | 是 |
| **访问数据** | `.read(cx)` / `.update(cx, f)` | 需 `.upgrade()` → `Option<Entity<T>>` | 需 `.downcast::<T>()` |
| **内部引用** | 通过 `AnyEntity` 间接持有 `Weak<EntityRefCounts>` | 持有 `Weak<EntityRefCounts>` | 持有 `Weak<EntityRefCounts>` + `TypeId` |
| **典型用途** | View handle, 字段存储 | 观察者回调（避免循环引用） | 集合/列表存储 |

## `WeakEntity<T>` 的 upgrade 流程

```rust
// src/app/entity_map.rs:681-686
impl<T: 'static> WeakEntity<T> {
    pub fn upgrade(&self) -> Option<Entity<T>> {
        Some(Entity {
            any_entity: self.any_entity.upgrade()?,
            entity_type: self.entity_type,
        })
    }
}
```

底层 `AnyWeakEntity::upgrade()` 做了原子操作：通过 `entity_ref_counts.upgrade()` 获取 `EntityRefCounts`，然后检查对应 `entity_id` 的 `AtomicUsize` 是否大于 0。如果用 `atomic_incr_if_not_zero` 成功递增（返回值 > 0），则创建新的 `AnyEntity`；如果计数已为 0 或 entity_id 已在 `dropped_entity_ids` 中，返回 `None`。

## Reservation：解决循环引用

两个实体互相引用是常见需求：

```rust
struct Parent {
    child: Option<Entity<Child>>,
}
struct Child {
    parent: Option<WeakEntity<Parent>>, // 弱引用，避免循环
}
```

但如果 `Parent` 的构造需要 `Child` 的句柄，`Child` 的构造又需要 `Parent` 的句柄呢？`EntityMap::reserve()` 解决了这个先有鸡还是先有蛋的问题：

```rust
// src/app/entity_map.rs:88-104
// 1. reserve 在 ref_counts 中先插入计数 1，返回 Slot<T>
let parent_slot = cx.reserve::<Parent>();  // → Slot<Parent>
let child_slot = cx.reserve::<Child>();    // → Slot<Child>

// Slot<T> 实现了 Deref<Entity<T>>，所以可以作为实体句柄使用
let parent_entity: Entity<Parent> = parent_slot.0;
let child_entity: Entity<Child> = child_slot.0;

// 2. insert 将实际数据写入 entities map
cx.insert(parent_slot, Parent {
    child: Some(child_entity.clone()),
});
cx.insert(child_slot, Child {
    parent: Some(parent_entity.downgrade()),
});
```

关键：`reserve` 在 `ref_counts.counts` 中插入 `AtomicUsize(1)`，同时创建 `Entity<T>` 句柄。此时 `entities` map 中尚无实际数据——句柄已经可用，但调用 `.read()` 会失败。只有当 `insert` 执行后数据才就位。

## 实体何时被清理？

清理分三步（`src/app/entity_map.rs:158-178`）：

```rust
pub fn take_dropped(&mut self) -> Vec<(EntityId, Box<dyn Any>)> {
    let mut ref_counts = self.ref_counts.write();
    let dropped_entity_ids = mem::take(&mut ref_counts.dropped_entity_ids);

    dropped_entity_ids
        .into_iter()
        .filter_map(|entity_id| {
            let count = ref_counts.counts.remove(entity_id).unwrap();
            debug_assert_eq!(count.load(SeqCst), 0,
                "dropped an entity that was referenced");
            // 从 entities 中移除数据（如果是通过 reserve 预留但未 insert，可能为 None）
            Some((entity_id, self.entities.remove(entity_id)?))
        })
        .collect()
}
```

1. 最后一个 `Entity<T>` drop → `AtomicUsize` 减到 0 → `entity_id` 推入 `dropped_entity_ids`。
2. `flush_effects` 循环中，`App::finish_update()` 调用 `entity_map.take_dropped()`。
3. `take_dropped` 验证计数确实为 0，从 `counts` slotmap 和 `entities` secondary map 中移除数据，返回给调用者（调用者可能触发 drop 副作用）。

清理不是在句柄 drop 时立即发生的——要等到下一轮 `flush_effects`。

---

**如果你只记住一件事：** `Entity<T>` 通过 `AnyEntity` 间接持有 `Weak<EntityRefCounts>`，数据在 `EntityMap.entities` 中。引用计数是单个 `AtomicUsize`（不是三个独立字段）。借出数据用 `Lease<T>` 的 "移除-操作-归还" 模式。清理分两步：计数归零 → 标记 dropped → `take_dropped()` 真正移除。
