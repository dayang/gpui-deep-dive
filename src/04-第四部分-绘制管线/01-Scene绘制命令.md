# 第 11 章：Scene——从元素树到绘制命令

`Scene` 是 GPUI 的中间表示层——`paint()` 阶段构建的输出，被平台渲染器消费。它是一个**扁平的绘制命令列表**（display list），不是场景图。

> 源文件参考：`src/scene.rs`

## Scene 数据结构

```rust
// src/scene.rs:23-35
pub(crate) struct Scene {
    pub(crate) paint_operations: Vec<PaintOperation>,
    primitive_bounds: BoundsTree<ScaledPixels>,
    layer_stack: Vec<DrawOrder>,
    pub(crate) shadows: Vec<Shadow>,
    pub(crate) quads: Vec<Quad>,
    pub(crate) paths: Vec<Path<ScaledPixels>>,
    pub(crate) underlines: Vec<Underline>,
    pub(crate) monochrome_sprites: Vec<MonochromeSprite>,
    pub(crate) polychrome_sprites: Vec<PolychromeSprite>,
    pub(crate) surfaces: Vec<PaintSurface>,
}
```

图元被分类存储在不同的 `Vec` 中，按 `DrawOrder` 排序后批处理提交给 GPU。

### DrawOrder

```rust
// src/scene.rs:21
pub(crate) type DrawOrder = u32;
```

`DrawOrder` 是 `u32` 类型别名——**不是结构体**。它通过 `BoundsTree<ScaledPixels>` 的 `insert()` 生成：每个图元在空间树中获得一个唯一的顺序值，确保正确的绘制顺序（越新插入的值越大，绘制越靠前）。

## 七种图元类型

```rust
// src/scene.rs:198-207
pub(crate) enum Primitive {
    Shadow(Shadow),
    Quad(Quad),
    Path(Path<ScaledPixels>),
    Underline(Underline),
    MonochromeSprite(MonochromeSprite),
    PolychromeSprite(PolychromeSprite),
    Surface(PaintSurface),
}
```

| 图元 | 用途 | 来源 |
|------|------|------|
| `Shadow` | 盒阴影（支持模糊半径和偏移） | `div().shadow()` |
| `Quad` | 矩形 + 背景 + 边框 + 圆角 | `div().bg(blue).border().rounded()` |
| `Path<ScaledPixels>` | 矢量路径 | `canvas()` 自定义绘制 |
| `Underline` | 下划线（支持波浪线） | `Label` 文本装饰 |
| `MonochromeSprite` | 单色精灵（字形/图标） | 文字渲染、SVG |
| `PolychromeSprite` | 彩色精灵 | `img()` 元素 |
| `PaintSurface` | 平台原生表面 | 视频、屏幕捕捉 |

### Quad —— 最常用的图元

```rust
// src/scene.rs:451-462
#[repr(C)]
pub(crate) struct Quad {
    pub order: DrawOrder,
    pub border_style: BorderStyle,
    pub bounds: Bounds<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub background: Background,          // ← 不是 Option<Hsla>，是 Background enum
    pub border_color: Hsla,              // ← 不是 Option<Hsla>
    pub corner_radii: Corners<ScaledPixels>,
    pub border_widths: Edges<ScaledPixels>,
}
```

关键点：
- **`background: Background`**：这是一个丰富的类型，支持 `Solid(Hsla)`、`LinearGradient(...)`、`PatternSlash(...)`，不只是一种颜色。
- **`border_color: Hsla`**：总是有值（不是 `Option`），透明边框用 `transparent_black()`。
- **`#[repr(C)]`**：确保与 shader 中的结构体内存布局一致。

## 绘制命令

```rust
// src/scene.rs:192-196
pub(crate) enum PaintOperation {
    Primitive(Primitive),
    StartLayer(Bounds<ScaledPixels>),
    EndLayer,
}
```

图层（StartLayer/EndLayer）实现嵌套裁剪（如 `overflow: hidden`）。

## insert_primitive——裁剪和排序

```rust
// src/scene.rs:67-115
pub fn insert_primitive(&mut self, primitive: impl Into<Primitive>) {
    let mut primitive = primitive.into();
    let clipped_bounds = primitive.bounds()
        .intersect(&primitive.content_mask().bounds);

    if clipped_bounds.is_empty() {
        return;  // 完全在可见区域外——跳过
    }

    let order = self.layer_stack.last().copied()
        .unwrap_or_else(|| self.primitive_bounds.insert(clipped_bounds));

    // 设置 order 并推入对应的 Vec
    match &mut primitive {
        Primitive::Quad(quad) => { quad.order = order; self.quads.push(quad.clone()); }
        // ... 其他类型类似
    }

    self.paint_operations.push(PaintOperation::Primitive(primitive));
}
```

关键行为：
1. **裁剪检测**：与 `content_mask` 的交集为空则跳过——不可见图元不进入 Scene。
2. **order 分配**：要么继承当前 layer_stack 的值，要么通过 `BoundsTree::insert` 分配新值。
3. **clone 到分类 Vec**：每个图元被复制到对应类型的 Vec 中以备后续批处理。

## finish() —— 排序

```rust
// src/scene.rs:127-137
pub fn finish(&mut self) {
    self.shadows.sort_by_key(|shadow| shadow.order);
    self.quads.sort_by_key(|quad| quad.order);
    self.paths.sort_by_key(|path| path.order);
    self.underlines.sort_by_key(|underline| underline.order);
    self.monochrome_sprites.sort_by_key(|sprite| (sprite.order, sprite.tile.tile_id));
    self.polychrome_sprites.sort_by_key(|sprite| (sprite.order, sprite.tile.tile_id));
    self.surfaces.sort_by_key(|surface| surface.order);
}
```

每个 Vec 独立按 `order` 排序。Sprites 还按 `tile_id` 二级排序（便于 GPU 批处理时按纹理分组）。

## PrimitiveBatch —— 渲染器的消费接口

```rust
// src/scene.rs:435-449
pub(crate) enum PrimitiveBatch<'a> {
    Shadows(&'a [Shadow]),
    Quads(&'a [Quad]),
    Paths(&'a [Path<ScaledPixels>]),
    Underlines(&'a [Underline]),
    MonochromeSprites { texture_id: AtlasTextureId, sprites: &'a [MonochromeSprite] },
    PolychromeSprites { texture_id: AtlasTextureId, sprites: &'a [PolychromeSprite] },
    Surfaces(&'a [PaintSurface]),
}
```

`PrimitiveBatch` 是**枚举**——不是结构体。每个变体携带对对应 Vec 的切片引用（不是 `Range<usize>`）。Sprites 变体还附带 `texture_id` 以便渲染器进行纹理绑定。

`batches()` 方法返回的迭代器产生按 `order` / `tile_id` 分组好的连续批次，每个 batch 对应一次 GPU draw call。

---

**如果你只记住一件事：** `DrawOrder = u32`（类型别名，不是结构体）。`PrimitiveBatch` 是 enum（不是 struct），变体携带切片引用。`Quad.background` 是 `Background` enum（不是 `Option<Hsla>`）。Scene 用 `finish()` 分别排序每个 Vec，用 `batches()` 产生渲染迭代器。
