# 第 12 章：GPU 图集——字形与精灵的纹理管理

文本和图像渲染是 GUI 框架的性能关键路径。GPUI 使用纹理图集（texture atlas）技术来高效管理成千上万个需要 GPU 渲染的小图形。

> 源文件参考：`src/platform.rs` 的 `PlatformAtlas` trait、`src/platform/mac/metal_atlas.rs`、`src/platform/windows/directx_atlas.rs`、`src/platform/blade/blade_atlas.rs`

## 为什么需要 Atlas？

**不用 Atlas**：100 个字符 → 100 个独立 texture → 100 次纹理切换 → 100 次 draw call → 性能灾难。

**用 Atlas**：所有 100 个字形 → 打包到一张大纹理 → 1 次纹理绑定 + 1 次 draw call。

## PlatformAtlas trait

```rust
// src/platform.rs:762-769
pub(crate) trait PlatformAtlas: Send + Sync {
    fn get_or_insert_with<'a>(
        &self,
        key: &AtlasKey,
        build: &mut dyn FnMut() -> Result<Option<(Size<DevicePixels>, Cow<'a, [u8]>)>>,
    ) -> Result<Option<AtlasTile>>;

    fn remove(&self, key: &AtlasKey);
}
```

关键点：
- **trait 可见性是 `pub(crate)`**——不是公共 API。
- **key 是 `&AtlasKey`**（一个具体类型，不是 `&dyn Hash`）。
- **build 闭包签名**：`&mut dyn FnMut() -> Result<Option<(Size<DevicePixels>, Cow<'a, [u8]>)>>`——可变闭包，返回可选的尺寸+字节数据。
- **返回值是 `Result<Option<AtlasTile>>`**——注意 `Option` 包装：渲染器可能选择不缓存（返回 `None`）。

没有 `clear()` 方法。

## AtlasKey

```rust
// src/platform.rs:714
pub(crate) enum AtlasKey {
    Glyph(RenderGlyphParams),
    Image(ImageId),
    // ...
}
```

`AtlasKey` 是一个具体枚举——不是 trait object 或泛型。这避免了虚函数开销，对于每帧可能调用数千次的图集操作很重要。

## AtlasTile（不是 AtlasItem）

```rust
// src/platform.rs:808-813
pub(crate) struct AtlasTile {
    pub(crate) texture_id: AtlasTextureId,
    pub(crate) tile_id: TileId,
    pub(crate) padding: u32,
    pub(crate) bounds: Bounds<DevicePixels>,
}

// src/platform.rs:817
pub(crate) struct AtlasTextureId {
    pub(crate) index: u32,
    pub(crate) kind: AtlasTextureKind,
}
```

- **名称是 `AtlasTile`**，不是 `AtlasItem`。
- **`texture_id` 是 `AtlasTextureId` 结构体**（包含 `index: u32` 和 `kind: AtlasTextureKind`），不是 `usize`。
- **`tile_id`**：打包算法分配的瓦片标识。
- **`padding`**：瓦片周围的像素边距（防止纹理采样时渗色）。
- **`bounds`**：瓦片在大纹理中的位置和尺寸（以 `DevicePixels` 为单位）。

## 字形渲染流程

```
1. 文本系统请求字形
   字体文件 (TTF/OTF)
     └─ font-kit / Core Text / DirectWrite
        └─ 提取字形轮廓 → 缩放 → 光栅化为灰度位图

2. 位图打包进 Atlas
   etagere::AtlasAllocator
     └─ PlatformAtlas::get_or_insert_with(&AtlasKey::Glyph(params), build_fn)
        └─ 返回 Result<Option<AtlasTile>>

3. 渲染时
   paint_text()
     └─ MonochromeSprite { tile: atlas_tile, color: Hsla, ... }
        └─ Scene::insert_primitive(...)

4. GPU 渲染
   shader 使用 atlas 纹理 + UV 坐标 + 文字颜色 → 屏幕像素
```

## 三种平台 Atlas 实现

| | macOS (Metal) | Windows (DX11) | Blade (跨平台) |
|---|---|---|---|
| **文件** | `metal_atlas.rs` | `directx_atlas.rs` | `blade_atlas.rs` |
| **GPU API** | `MTLTexture` | `ID3D11Texture2D` | blade-graphics |
| **打包算法** | `etagere` | `etagere` | `etagere` |

所有实现都使用 `etagere` 作为打包库。

## 批处理与 Atlas 的关系

在 `Scene::finish()` 中，monochrome sprites 按 `(order, tile_id)` 排序。这意味着同一 atlas tile 的精灵倾向于连续排列。`PrimitiveBatch::MonochromeSprites { texture_id, sprites }` 携带 `texture_id`，渲染器可以在渲染前绑定正确的纹理。

## Atlas 驱逐策略

- **LRU（最近最少使用）**：长时间未渲染的条目优先驱逐。
- **按帧标记**：每帧标记使用的 entry，帧末清理未标记的。
- Atlas 满后分配更大的纹理（如 2048 → 4096），重新打包。

---

**如果你只记住一件事：** `PlatformAtlas` 是 `pub(crate)` trait。`get_or_insert_with` 的 key 是具体枚举 `AtlasKey`（不是 `&dyn Hash`）。返回值是 `AtlasTile`（不是 `AtlasItem`），包含 `texture_id: AtlasTextureId`、`tile_id`、`padding` 和 `bounds`。
