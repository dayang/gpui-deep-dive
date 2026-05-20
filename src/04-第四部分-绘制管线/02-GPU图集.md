# 第 12 章：GPU 图集——字形与精灵的纹理管理

文本和图像渲染是 GUI 框架的性能关键路径。GPUI 使用纹理图集（texture atlas）技术来高效管理成千上万个需要 GPU 渲染的小图形。

> 源文件参考：`src/platform.rs` 的 `PlatformAtlas` trait、`src/platform/mac/metal_atlas.rs`、`src/platform/windows/directx_atlas.rs`、`src/platform/blade/blade_atlas.rs`

## 为什么需要 Atlas？

假设你要在屏幕上渲染一段 100 个字符的文本。每个字符都是一个独立的字形（glyph）：

**不用 Atlas**（naive 做法）：
```
100 个字符 × 每个字符一个 GPU texture = 100 次纹理切换
100 次 draw calls
= 性能灾难（帧率可能降到个位数）
```

**用 Atlas**：
```
所有 100 个字形 → 打包到一张大纹理中
1 次纹理绑定 + 1 次 draw call（如果都在同一批）
= 现代 GPU 轻松跑 60fps
```

## 图集的工作原理

```
纹理图集（例如 2048×2048 像素）
┌──────────────────────────────┐
│  A │  b  │  快 │  the │ QQQ  │
│    │     │     │      │      │
│─────┼─────┼─────┼──────┼──────│
│  径 │ 文  │ .   │ 123  │      │
│     │     │     │      │      │
│─────┼─────┼─────┼──────┼──────│
│  !  │ (空)│ 中  │ 时   │  X   │
│ ────┼─────┼─────┼──────┼──────│
│  em │ oji │ 國  │ (空) │ (空) │
└──────────────────────────────┘

每个小矩形是一个 glyph（字形栅格化后的位图）。
MonochromeSprite 引用 atlas 中的一个矩形子区域。
```

## PlatformAtlas trait

```rust
pub trait PlatformAtlas: Send + Sync {
    fn get_or_insert_with(
        &self,
        key: &dyn Hash,
        render: &dyn Fn() -> RenderImage,
    ) -> Result<AtlasItem>;

    fn remove(&self, key: &dyn Hash);

    fn clear(&self);
}

pub struct AtlasItem {
    pub texture_id: usize,       // 在哪个纹理中
    pub bounds: Bounds<DevicePixels>,  // 纹理中的位置
}
```

`get_or_insert_with` 是核心方法：
- 如果 key 已在 atlas 中，直接返回位置。
- 如果不在，调用 `render` 闭包生成位图，打包进 atlas，返回位置。

## Atlas 的填充与驱逐

当 atlas 满时（新 glyph 放不下了）：

1. **分配更大的纹理**：例如 2048 → 4096
2. **驱逐旧条目**：从未使用的条目开始清理
3. **重新打包**：使用 `etagere` 库（高效的矩形打包算法）

`etagere` 使用的是类似于游戏引擎中常用的 shelf-packing 算法，比简单的贪心算法更好地利用纹理空间。

## 字形渲染流程

从字体文件到屏幕上的像素：

```
1. 文本系统请求字形
   字体文件 (TTF/OTF)
     │
     ├─ font-kit / Core Text / DirectWrite
     │    └─ 提取字形轮廓
     │       └─ 缩放为指定大小
     │          └─ 光栅化为位图（灰度图）
     │
2. 位图打包进 Atlas
   etagere::AtlasAllocator
     └─ 找到合适的矩形区域
        └─ PlatformAtlas::get_or_insert_with(glyph_key, || render_glyph())
           └─ 返回 AtlasItem { texture_id, bounds }

3. 渲染时
   paint_text()
     └─ MonochromeSprite {
            atlas_item,        ← 在 atlas 中的位置
            color: Hsla,       ← 文字颜色
            // ... 其他属性
        }
     └─ Scene::insert_primitive(...)

4. GPU 渲染
   shader 使用 atlas 纹理 + UV 坐标 + 文字颜色 → 屏幕
```

## 三种平台 Atlas 实现

| | macOS (Metal) | Windows (DX11) | Blade (跨平台) |
|---|---|---|---|
| **文件** | `metal_atlas.rs` | `directx_atlas.rs` | `blade_atlas.rs` |
| **GPU API** | Metal (`MTLTexture`) | Direct3D 11 (`ID3D11Texture2D`) | blade-graphics |
| **纹理格式** | `MTLPixelFormatA8Unorm`（单通道） | `DXGI_FORMAT_A8_UNORM` | 取决于后端 |
| **打包算法** | `etagere` | `etagere` | `etagere` |

所有实现都使用 `etagere` 作为打包库——GPUI 没有为每个平台重新发明打包算法。

## 精灵渲染中的 Atlas 使用

文字是最主要的 atlas 用户，但也用于：

- **图片渲染**（`img()` 元素）——解码后的图片数据进入 atlas
- **SVG 渲染**——`resvg` 光栅化的位图进入 atlas
- **GIF 帧**——每一帧单独打包
- **Canvas 绘制**——自定义路径填充后的光栅化结果

## 纹理管理与性能

### batch 合并

在 `Scene::finish()` 中，monochrome sprites 被按 `atlas_texture_id` 分组。这意味着：

```
Batch 1: texture_id=0, sprites[0..50]   → 1 次 draw call
Batch 2: texture_id=1, sprites[50..72]  → 1 次 draw call
Batch 3: texture_id=0, sprites[72..100] → 又 1 次 draw call（非连续）
                                            ↑ 这是一个优化点
```

理想情况下，同一纹理的所有精灵连续排列，以最少 draw calls。GPUI 的排序策略努力做到这一点，但少量不连续是正常的。

### 图集驱逐策略

- **LRU（最近最少使用）**：长时间未渲染的 glyph 优先驱逐。
- **按帧标记**：每帧标记使用的 entry，帧末清理未标记的。
- **引用计数辅助**：`EntityMap` 释放视图时，关联的精灵引用减少。

---

**如果你只记住一件事：** Atlas 是"把很多小纹理打包成一张大纹理"的技术。它让 GPU 能用一次 bind + 一次 draw call 渲染大量不同的字形和图标，是文字密集型 GUI 的性能基石。
