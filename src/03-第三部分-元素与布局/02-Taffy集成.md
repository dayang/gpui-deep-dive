# 第 9 章：Taffy 集成——CSS 布局引擎

GPUI 不自己实现布局算法。它集成了 [Taffy](https://crates.io/crates/taffy)（v0.9.0），一个实现了 CSS Flexbox 和 CSS Grid 布局规范的 Rust 库。本章讨论 GPUI 如何将 Style 映射到 Taffy，以及在映射中发生了什么。

> 源文件参考：`src/taffy.rs`、`src/style.rs`、`src/styled.rs`

## Taffy 是什么

Taffy 是一个纯 Rust 的布局引擎，实现了：

- **CSS Flexbox**（`display: flex`）
- **CSS Grid**（`display: grid`）
- **Block layout**（`display: block`）
- 完整的 Box Model（content → padding → border → margin）

GPUI 选择 Taffy 而非自己实现的原因很简单：布局是极其复杂的，CSS 规范花了几十年才成熟。重新发明一个布局引擎没有任何竞争优势。

## GPUI 的适配层

`src/taffy.rs` 是 GPUI 和 Taffy 之间的桥梁。它不暴露 Taffy 的内部类型——而是将 GPUI 的 `Style` 映射为 Taffy 能理解的结构：

```
GPUI Style                    Taffy
───────────                   ─────
Size<Length>          ──→     taffy::Dimension
DefiniteLength        ──→     taffy::LengthPercentage
AbsoluteLength        ──→     taffy::LengthPercentage
FlexDirection         ──→     taffy::FlexDirection
AlignItems            ──→     taffy::AlignItems
JustifyContent        ──→     taffy::JustifyContent
GridPlacement         ──→     taffy::GridPlacement
...
```

关键的类型转换在于长度单位：

```rust
// GPUI 的长度枚举
pub enum Length {
    Definite(DefiniteLength),
    Auto,
}

pub enum DefiniteLength {
    Absolute(AbsoluteLength),  // 固定值
    Fraction(f32),             // flex-grow 比例
}

pub enum AbsoluteLength {
    Pixels(Pixels),
    Rems(Rems),
    Em(Em),
}
```

## REM 和 EM 的解析

`rem` 和 `em` 是相对单位，需要运行时上下文才能转换为像素：

- **REM**（root em）：相对于窗口的默认字体大小。`1rem` = `window.default_font_size`。
- **EM**：相对于当前元素的字体大小。`1em` = 当前元素的 `font_size`。

```rust
impl AbsoluteLength {
    pub fn to_pixels(
        &self,
        rem_size: Pixels,     // 窗口的默认字体大小
        parent_font_size: Option<Pixels>,  // 当前元素的字体大小
        scale_factor: f32,    // HiDPI 缩放因子
    ) -> Pixels {
        match self {
            AbsoluteLength::Pixels(px) => *px,
            AbsoluteLength::Rems(rems) => rems.0 * rem_size,
            AbsoluteLength::Em(em) => {
                let parent_size = parent_font_size.unwrap_or(rem_size);
                em.0 * parent_size
            }
        }
    }
}
```

在 `request_layout` 阶段，GPUI 遍历 `text_style_stack` 来确定每个元素在求解时的 `parent_font_size`。这使得 `em` 能像 CSS 一样正确继承。

## `AvailableSpace` 的传递

Taffy 的布局计算需要每个节点的可用空间信息。GPUI 在 `request_measured_layout` 中处理两种模式：

```rust
// 固定尺寸——元素告诉 Taffy 它应该多大
window.request_measured_layout(style, known_size: Size<AvailableSpace>);

// 弹性尺寸——由 Taffy 决定
window.request_layout(style, children: &[LayoutId]);
```

`AvailableSpace` 枚举：

```rust
pub enum AvailableSpace {
    Definite(DefiniteLength),  // 精确尺寸
    MinContent,                // 内容的最小尺寸
    MaxContent,                // 内容的最大尺寸
}
```

## 布局求解

`Window::compute_layout()` 内部调用 Taffy 的求解器：

```rust
pub(crate) fn compute_layout(&mut self, viewport_size: Size<Pixels>) {
    let taffy = &mut self.layout_engine;
    taffy.compute_layout(
        root_layout_id,
        taffy::Size {
            width: taffy::AvailableSpace::Definite(viewport_size.width.into()),
            height: taffy::AvailableSpace::Definite(viewport_size.height.into()),
        },
    );
    // 每个 LayoutId 现在都有计算后的 Bounds<Pixels>
}
```

布局求解后，每个 `LayoutId` 获得一个 `LayoutNode`（位置 + 尺寸）。GPUI 在 `prepaint` 阶段读取这些计算结果来确定每个元素的 `bounds`。

## Styled trait：Tailwind 风格 API 的实现

`Styled` trait（`src/styled.rs`）是所有链式 API 的提供者：

```rust
pub trait Styled: Sized {
    fn px(mut self, value: impl Into<AbsoluteLength>) -> Self { ... }
    fn py(mut self, value: impl Into<AbsoluteLength>) -> Self { ... }
    fn bg(mut self, color: impl Into<Hsla>) -> Self { ... }
    fn text_color(mut self, color: impl Into<Hsla>) -> Self { ... }
    fn rounded(mut self, radius: impl Into<AbsoluteLength>) -> Self { ... }
    fn border(mut self) -> Self { ... }
    fn shadow(mut self) -> Self { ... }
    fn flex(mut self) -> Self { ... }
    fn flex_col(mut self) -> Self { ... }
    fn grid(mut self) -> Self { ... }
    fn grid_cols(mut self, cols: impl Into<Vec<DefiniteLength>>) -> Self { ... }
    fn items_center(mut self) -> Self { ... }
    fn justify_center(mut self) -> Self { ... }
    fn gap(mut self, value: impl Into<AbsoluteLength>) -> Self { ... }
    fn cursor(mut self, style: CursorStyle) -> Self { ... }
    fn absolute(mut self) -> Self { ... }
    fn relative(mut self) -> Self { ... }
    fn visible(mut self, visible: bool) -> Self { ... }
    fn overflow_hidden(mut self) -> Self { ... }
    fn z_index(mut self, z: u16) -> Self { ... }
    // ... 还有大约 100+ 方法
}
```

每个方法修改内部 `StyleRefinement` 的一个字段。所有链式调用合并为一个 `StyleRefinement`，在 `request_layout` 时应用。

## StyleRefinement 的合并

```rust
pub struct StyleRefinement {
    pub display: Option<Display>,
    pub overflow: Option<Overflow>,
    pub margin: Option<EdgesRefinement>,
    pub padding: Option<EdgesRefinement>,
    pub size: Option<SizeRefinement>,
    pub min_size: Option<SizeRefinement>,
    pub max_size: Option<SizeRefinement>,
    pub flex_grow: Option<f32>,
    pub flex_shrink: Option<f32>,
    pub flex_basis: Option<DefiniteLength>,
    pub border: Option<BorderRefinement>,
    pub corner_radii: Option<Corners<AbsoluteLength>>,
    pub background: Option<Hsla>,
    pub box_shadow: Option<BoxShadow>,
    pub cursor_style: Option<CursorStyle>,
    pub z_index: Option<u16>,
    // ...
}
```

每个字段都是 `Option<T>`——`None` 表示"未设置，使用默认值"。`StyleRefinement` 通过 `Refineable` derive宏生成的 `refine()` 方法与基础 `Style` 合并：

```rust
let final_style = base_style.refine(&refinement);
// 将 refinement 中的 Some 覆盖到 base_style 上，None 保持原值
```

---

**如果你只记住一件事：** GPUI 的布局由 Taffy 求解，Style → StyleRefinement → Taffy Style 的映射在 `request_layout` 阶段发生。REM/EM 等相对单位需要运行时上下文（窗口字体大小、父字体大小）来解析。
