# 第 11 章：Scene——从元素树到绘制命令

`Scene` 是 GPUI 的中间表示层——它是 `paint()` 阶段构建的输出，被平台渲染器消费。它的设计原则是**扁平、分批、可缓存**。

> 源文件参考：`src/scene.rs`

## Scene 不是场景图

首先澄清一个命名上的混淆。传统图形学中的"场景图"（scene graph）是一个**树状结构**，节点有父子关系，变换矩阵从根向下累积。

GPUI 的 `Scene` 完全不同——它是一个**扁平的绘制命令列表**（display list）：

```rust
pub struct Scene {
    paint_operations: Vec<PaintOperation>,
    primitive_bounds: BoundsTree<ScaledPixels>,
    layer_stack: Vec<DrawOrder>,

    shadows: Vec<Shadow>,
    quads: Vec<Quad>,
    paths: Vec<Path<Pixels>>,
    underlines: Vec<Underline>,
    monochrome_sprites: Vec<MonochromeSprite>,
    polychrome_sprites: Vec<PolychromeSprite>,
    surfaces: Vec<PaintSurface>,
}
```

图元被分类存储在不同的 `Vec` 中，然后按 `DrawOrder` 排序后批处理提交给 GPU。

## 七种图元

| 图元 | 用途 | 典型来源 |
|------|------|---------|
| `Quad` | 矩形 + 背景色 + 边框 + 圆角 | `div().bg(blue).border().rounded()` |
| `Shadow` | 盒阴影（支持模糊半径和偏移） | `div().shadow()` |
| `Underline` | 下划线（支持波浪线） | `Label` 中文本装饰 |
| `MonochromeSprite` | 单色精灵（字形/图标） | 文字渲染、SVG 图标 |
| `PolychromeSprite` | 彩色精灵 | `img()` 元素 |
| `Path<Pixels>` | 矢量路径 | `canvas()` 自定义绘制 |
| `PaintSurface` | 原生系统表面 | 视频播放、屏幕捕捉（macOS） |

### Quad——最常用的图元

```rust
pub struct Quad {
    pub bounds: Bounds<ScaledPixels>,
    pub background: Option<Hsla>,
    pub border_color: Option<Hsla>,
    pub border_widths: Edges<ScaledPixels>,
    pub border_style: BorderStyle,
    pub corner_radii: Corners<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub order: DrawOrder,
}
```

你在 GPUI 中看到的大部分界面——按钮、面板、卡片、输入框——都由 Quad(s) 渲染。一个带阴影的按钮至少产生两个图元：一个 `Shadow` + 一个 `Quad`。

## 绘制命令与图层

```rust
pub enum PaintOperation {
    Primitive(Primitive),
    StartLayer(Bounds<ScaledPixels>),
    EndLayer,
}
```

`StartLayer` / `EndLayer` 表示嵌套的"层"——每层有一个裁剪边界。这是 `overflow: hidden` 或 `rounded` 裁剪在 GPU 层面实现的方式。

### DrawOrder

```rust
pub struct DrawOrder {
    pub layer_depth: u32,     // 层嵌套深度
    pub z_index: u16,         // CSS z-index
    pub primitive_kind: PrimitiveKind,  // 排序分组
}
```

`DrawOrder` 决定了图元的绘制顺序：

1. 首先按 `layer_depth` 排序（内层在后）
2. 然后按 `z_index` 排序
3. 最后按 `primitive_kind` 分组（允许 GPU 批处理）

## 构建流程

```
paint() 阶段：
  每个元素调用 window.paint_quad(...)
    → Scene::insert_primitive(Primitive::Quad(quad))
      ├─ 与 content_mask 做交集验证（完全在裁剪区域外 → 跳过）
      ├─ 转换为 ScaledPixels（应用 scale_factor）
      ├─ 计算 DrawOrder（基于当前 layer_stack 和 z_index）
      └─ 推入对应 Vec

  window.paint_text(...)
    → Scene::insert_primitive(Primitive::MonochromeSprite(sprite))
      └─ ... 类似流程

  以此类推
```

## finish() 和批处理

```rust
impl Scene {
    pub fn finish(&mut self) {
        // 1. 将所有图元与它们的 DrawOrder 合并
        // 2. 按 DrawOrder 排序
        // 3. 对 sprites 按 atlas texture_id 分组（同一纹理的批次一起提交）
        self.sort_and_batch();
    }

    pub fn batches(&self) -> BatchIterator {
        // 返回 PrimitiveBatch 迭代器
        // 每个 batch 包含相同类型的图元，可以一次 GPU draw call 完成
    }
}
```

一个 `PrimitiveBatch` 包含：

```rust
pub struct PrimitiveBatch {
    pub kind: PrimitiveKind,
    pub draws: Range<usize>,      // 在对应 Vec 中的索引范围
    pub texture_id: Option<u64>,  // sprites 专属：atlas texture id
}
```

平台渲染器消费 batch 时，可以将一个 batch 内的所有图元在一次 draw call 中提交——无需在 draw call 之间切换纹理或 shader。

## 裁剪优化

Scene 在插入图元时做了一层裁剪检测：

```rust
fn insert_primitive(&mut self, primitive: Primitive, bounds: Bounds<ScaledPixels>) {
    // 与当前 content_mask 栈的 top 求交集
    let mask = self.content_mask_stack.last();
    if let Some(mask) = mask {
        let intersection = mask.intersect(&bounds);
        if intersection.is_empty() {
            return; // 完全在可见区域外——跳过
        }
    }
    // ... 推入图元
}
```

这确保不可见元素不仅不 paint，甚至不进入 Scene——减少 GPU 工作。

## 精灵的纹理批处理

文字是 GPUI 中最复杂的渲染工作。每个字符的每个字号都是一个 `MonochromeSprite`，来自 GPU 纹理图集（atlas）。

`Scene::finish()` 中，`MonochromeSprite` 按 `atlas_id` 分组：
- 同一 atlas 的所有字符可以在一次 draw call 中渲染
- Atlas 切换是昂贵的——需要 GPU 纹理绑定变更
- 所以 GPUI 努力将同一张 atlas 的所有精灵聚集在一起

---

**如果你只记住一件事：** Scene 是扁平的绘制命令列表，不是场景图。它把 paint() 产生的各种图元按 DrawOrder 排序并批处理，以减少 GPU draw calls。
