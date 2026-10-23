# tiu 布局语法与几何语义设计 —— 显式尺寸代数与四容器模型
*EN: tiu Layout Syntax and Geometry Semantics — explicit size algebra over a four-container model*

**日期** / Date: 2026-10-23 · **类型** / Type: 语法与语义设计（tiu 全量实现的几何契约）
**依据** / Basis: 下列约束共同定义几何契约——控件工厂为唯一真源、布局语法按显式尺寸意图重设计、文本走 TTF 光栅、允许修改引擎软件内核、事件轴一次到位、`ui/src/app.tie` 承担单帧循环、布局属性类型化 · `docs/designs/tiu-render-engine.md` · `docs/designs/tiu-event-system.md` · `docs/designs/tiu-drawing-api.md`
**关联实现** / Implementation: `ui/src/layout2.tie` · `ui/src/props.tie` · `ui/src/tree.tie` · `ui/src/paint_bridge.tie` · `ui/src/paint_state.tie` · `ui/src/ui_bridge.tie` · `ui/src/attr_parse.tie` · `ui/src/app.tie` · `ui/src/view.tie` · `engine/src/{font,glyph,raster}.tie` · `api/src/drawlist.tie` · `lib/front/pstmt_ui.tie`（tie 仓）

> EXEC BRIEF: Defines the geometry contract for tiu's UI layer. Three commitments:
> (1) **one solver** — four containers (`row`/`column`/`grid`/`overlay`) share one size
> algebra and one alignment vocabulary, replacing the five mode-specific rule sets of
> the web; (2) **explicit size intent** — every axis answers "content / remaining /
> definite" with a distinct form (`hug` / `fill` / `480`), so `auto` vs `100%` vs
> `fit-content` ambiguity disappears; (3) **deterministic, explainable geometry** —
> fixed-point solving with largest-remainder rounding, constraint provenance kept per
> node for `ui.explain(id)`. Deliberate omissions: no margin, no cascade, no
> measurement API, no breakpoint language.

---

## 1. 立场与边界 / Stance and boundaries

三条硬线 / Three hard lines：

* **一个求解器**：不存在 block/inline/flex/grid/float/positioned 的分裂；四种容器共用尺寸代数、对齐词汇与求解顺序。
* **尺寸意图显式**：每个轴的尺寸必须回答"内容 / 剩余 / 确定值"，三者在语法上不同形。
* **几何可解释且确定**：任何矩形可追到约束来源；定点求解、写死取整规则，跨平台同结果。

非目标 / Non-goals：不做 CSS 兼容、不做浮动与文本环绕、不引入断点语言、不提供运行期测量 API、不实现动画对布局的反向驱动。

---

## 2. 尺寸代数 / Size algebra

每轴一个尺寸值 / one size value per axis：

| 写法 | 名称 | 语义 | 允许位置 |
|---|---|---|---|
| `hug` | 内容 | 文本度量或子项之和 + pad | `w`/`h` |
| `fill` | 剩余 | 占父内容区该轴剩余空间（等价 `1fr`） | `w`/`h` |
| `200`、`200px` | 逻辑像素 | 1/96 英寸，按 DPI 缩放 | 一切长度 |
| `60%` | 百分比 | 父**内容盒**同轴尺寸的 60%；父该轴须定尺寸 | `w`/`h`/`min-*`/`max-*` |
| `2fr` | 分数 | 剩余空间按权重分配 | `w`/`h` |
| `1.5em` | 字号倍数 | 当前字号的 1.5 倍 | 一切长度 |
| `12pt` | 点 | 4/3 逻辑像素（字号与印刷常用） | 一切长度 |
| `50vw`、`50vh` | 窗口百分比 | 窗口内容区宽/高的 50% | 一切长度 |
| `ratio(16:9)` | 比例 | 另一轴定尺寸时按比例 | `w`/`h` |

每轴两个约束 / per-axis clamps：`min-w`、`max-w`、`min-h`、`max-h`，取值 `n`、`n%`、`n em`、`n pt`、`n vw`、`n vh`。`fit-content` 不需要——它就是 `hug` 加 `min/max`。

规则 / Rules：

1. **默认 `hug`**。撑满必须写 `fill`。
2. `%` 的分母是父内容盒（已扣 pad）；父在该轴不定尺寸则为错误，不是静默 0。
3. `fill` 是 `1fr` 的别名，不引入第二套拉伸语义。
4. **只有 `font-size` 继承**：`em` 取最近祖先的显式 `font-size`，无则取根默认 16。其余属性一律不继承，消灭隐式级联。
5. 内部求解用 26.6 定点（1 单位 = 1/64 逻辑像素），落树时四舍五入到整数像素；`explain` 保留定点值。
6. 余数分配用最大余数法，权重相同按子序，保证同一输入在任何平台得到同一组整数。

---

## 3. 容器与轴 / Containers and axes

| 容器 | 主轴 | 交叉轴 | 特有属性 |
|---|---|---|---|
| `row` | x | y | `gap`、`x`、`y` |
| `column` | y | x | `gap`、`x`、`y` |
| `grid` | 列由 `cols` 决定 | 行 | `cols`、`rows`、`gap` |
| `overlay` | 无 | 无 | 子项 `anchor`、`dx`、`dy` |

对齐词汇按**绝对轴**绑定，不随方向翻转：

```
x: start | center | end | space-between | space-around | space-evenly
y: start | center | end | space-between | space-around | space-evenly
交叉轴额外允许：stretch
```

* 主轴上的 `space-*` 吸收剩余空间，此时 `gap` 作为最小间距。
* 交叉轴 `stretch` 让子项撑满交叉轴；默认 `start`。
* `@ row (y: space-between)` 非法：`row` 的交叉轴没有分布语义。工厂在布局期拒绝并报节点名（类型化落地后可提前到编译期）。

间距属性 / Spacing：

* `pad`、`pad-x`、`pad-y`、`pad-left/right/top/bottom`：父内边距，只接受长度。
* `gap`：主轴相邻子项间距，只接受长度。
* **无 margin**：没有外边距、没有塌陷、没有负值、没有"百分比按宽度解析"。单侧间隔用 `@ spacer` 或包一层带 `pad` 的容器。

`grid`：`cols: 3`（等分）、`cols: [120, fill, 2fr]`；行高默认 `hug`，可给 `rows: [48, hug]`；`gap` 同时作用于行列。跨列 `span` 暂不支持。

`overlay`：子项按 `anchor` 落位，见 §7。

---

## 4. 求解 / Solving

三趟 / three passes（沿用 `ui/src/layout.tie` 的 propagate/measure/place 骨架，替换规则）：

```
measure_hug(node, axis):
    text        -> 字体度量（advance 之和 / 行高）
    row/column  -> 主轴：Σ子项主轴尺寸 + gap·(n-1) + pad_start + pad_end
                   交叉轴：max(子项交叉尺寸) + pad
    grid        -> 列和（cols 求和）+ gap + pad；行高取该行最高子项
    overlay     -> 子项 anchor=fill 时取父尺寸，否则 max(子项 + 偏移)

distribute(container):
    avail   = 父内容盒轴长
    fixed   = pad_start + pad_end + gap·(n-1)
    for child:  Px/Em/Pt/Vw/Vh -> 转逻辑像素
                Pct          -> floor((avail - pad) · n / 100)
                Ratio        -> 另一轴尺寸 · a / b
                Hug          -> measure_hug(child, axis)
                Fill/Fr      -> flexible += weight
    rest    = avail - fixed - Σ己定尺寸
    if flexible > 0 and rest > 0: 按权重最大余数分配
    clamp 每个到 [min, max]
    overflow = Σ实际尺寸 > avail - pad（判定，不改几何）

place(container):
    主轴游标 = pad_start；按 x/y 分布词计算起始偏移或空隙
    交叉轴：start | center | end | stretch
```

关键约束 / Invariants：

* **clamp 让出的空间不回灌主轴**，留在主轴由分布词吸收——写死在规范里，避免"迭代到收敛"的不定点，也就没有 CSS 那种「改了 A 又回去改 B」的抖动。
* `rest < 0` 时不压缩 `hug` 子项，直接判定溢出并由 `overflow` 策略处置。
* 文本折行发生在宽度确定之后；行数变化会改变 `hug` 高度，故文本节点必须参与 measure 趟。
* 任何节点都记录约束来源：`(父名, 轴, 下发值, 该节点自身 min/max, 结果)`，供 `ui.explain(id)` 输出。

---

## 5. 溢出 / Overflow

`overflow: clip | scroll | expand | error`（默认 `clip`）

| 值 | 行为 |
|---|---|
| `clip` | 裁到内容盒；文本可配 `ellipsis` |
| `scroll` | 裁剪 + 可滚；滚动量是节点状态，命中按滚动量补偿 |
| `expand` | 在溢出轴上改为 `hug`（牺牲父给的尺寸） |
| `error` | 布局期报诊断：节点名、期望/实际尺寸、约束来源链 |

`error` 的定位是开发与探针专用；发布构建默认 `clip` 以避免崩溃式失败，但诊断通道仍可在 `explain` 里取到溢出记录。

---

## 6. 文本 / Text

| 属性 | 默认 | 说明 |
|---|---|---|
| `wrap` | `label`/`heading` 为 true，`button`/`input` 为 false | 宽度确定后贪心断行 |
| `overflow` | `clip` | 文本可配 `ellipsis` |
| `text-align` | `start` | `start`/`center`/`end` |
| `font-size` | 16 | `em` 的基准，唯一可继承属性 |

度量与光栅共用同一份字形 advance（TTF `hmtx`），布局宽度不再乘系数——这是现状 33% 偏差的根因。度量结果按 `(字体, 字号, 文本)` 缓存；断行结果按 `(字体, 字号, 宽度, 文本)` 缓存。

---

## 7. 锚点 / Anchors（`overlay` 子项专用）

```
anchor: top-start | top | top-end
        start     | center | end
        bottom-start | bottom | bottom-end
        fill            // 撑满父内容盒
```

`dx`/`dy` 为相对锚点的偏移，可为负。九点锚点取代 top/left/right/bottom 组合：落位只由「锚点 + 偏移」决定，不再有"同时给 left 与 right 会拉伸"这类隐式规则（那是 `anchor: fill` 的职责）。

---

## 8. 样式与主题 / Styles and themes

```
style card {
    pad: 12
    gap: 6
    radius: 8
    bg: surface
    border: 1
}

@ panel "settings" (style: card, w: fill) { ... }
```

* 样式按名**静态引用**，没有选择器、没有 specificity、没有 `!important`。
* 合并顺序固定：元素内联属性 > 引用样式 > 控件默认值。
* 同层重复设置同一属性（元素内联出现两次，或两个样式叠加设同一键）为错误，报键名与来源。
* 主题 token 由 `theme` 单元提供；`bg: accent` 引用 token，`bg: "#33aa77"` 为字面量，二者在绘制期统一解析为 ARGB。

---

## 9. 语法与降级 / Syntax and desugaring

`lib/front/pstmt_ui.tie`（tie 仓）只做语法识别与降级，不认识布局语义。属性键携带类型：

| 键 | 类型 | 降级 |
|---|---|---|
| `w`/`h`/`min-w`/`max-w`/`min-h`/`max-h` | `Size` | `ui.w(Size.fill())` 等 |
| `x`/`y` | `Dist` | `ui.x(Dist.center)` |
| `overflow` | `Overflow` | `ui.overflow(Overflow.scroll)` |
| `anchor` | `Anchor` | `ui.anchor(Anchor.top_end)` |
| `pad*`/`gap`/`dx`/`dy`/`radius`/`font-size`/`border` | `i64`/长度 | `ui.pad(12)` |
| `bg`/`fg` | `Token` 或颜色字面量 | `ui.bg(Token.surface)` |
| `style` | 样式名 | `ui.style("card")` |
| `on_*` | 表达式 | `ui.on("click", handler)`（现状不变） |
| 未登记键 | — | `ui.attr_str(key, 原文)`，保留控件私有扩展 |

值侧语法糖 / value sugar（仅属性值位置）：`fill`/`hug` → `Size.fill()`/`Size.hug()`；`60%` → `Size.pct(60)`；`2fr` → `Size.fr(2)`；`1.5em` → `Size.em(1.5)`；`12pt` → `Size.pt(12)`；`50vw`/`50vh` → `Size.vw(50)`/`Size.vh(50)`；`ratio(16:9)` → `Size.ratio(16, 9)`；裸整数 → `Size.px(n)`。

词法层新增：`%` 后缀、`<num> <ident>` 复合单位（`fr`/`em`/`pt`/`vw`/`vh`）识别——仅当数字紧邻单位标识符且在属性值位置生效；普通表达式中仍写 `Size.pct(60)`。

示例 / Example：

```
@ row "acc" (w: fill, pad: 12, gap: 8, y: center) {
    @ label (text: "账号", w: 56)
    @ input (bind: user, w: fill)
}
```

降级为 / desugars to：

```
ui.begin("row", "acc")
ui.w(Size.fill())
ui.pad(12)
ui.gap(8)
ui.y(Dist.center)
ui.begin("label", "")
ui.text("账号")
ui.w(Size.px(56))
ui.end()
ui.begin("input", "")
ui.bind("user")
ui.w(Size.fill())
ui.end()
ui.end()
```

---

## 10. 静态检查边界 / What is checked when

| 阶段 | 检查项 |
|---|---|
| 编译期 | 属性名拼错、单位与类型不符（`w: "abc"`）、重复属性、样式名未定义、`anchor` 出现在非 `overlay` 子项、`space-*` 用在交叉轴（类型化后再收紧） |
| 布局期 | `%` 落在不定尺寸轴、`overflow: error`、负剩余、`explain` 链、`hug` 递归成环 |
| 探针 | 黄金矩形表：四种容器 × 各尺寸形态，断言 `x/y/w/h` 精确值；文本度量与绘制宽度一致 |

诚实边界：几何是否装得下取决于运行期文本与数据，编译期无法判定；因此编译期只保证"词汇与类型正确"，几何正确性靠布局期断言与探针。

---

## 11. 与现有实现的差距 / Gaps against current code

| 现状 | 目标 |
|---|---|
| `ui/src/layout.tie:89-115` 仅 Row/Column 求和 + Absolute，无 pad/gap/align/fr/% | 按 §4 重写求解器，保留三趟骨架与 `layout_dirty` |
| `ui/src/layout.tie:160-167` 与 `ui/src/paint_bridge.tie:28-30` 用 8px/字符、16 行高估算 | 改为 TTF 度量，布局与绘制共用同一份 advance |
| `ui/src/paint_bridge.tie:15-20` 只为 Text/Button 产记录，容器无背景，回避 `rrect` | 控件工厂产出全部类型；`raster` 支持 rrect 抗锯齿与 clip |
| `ui/src/ui_bridge.tie` 只解释 `text/size/x/y/w/h` 六类属性 | 桥消费类型化 props，其余交给工厂 |
| `ui/src/view.tie`（`vw`）与桥并存两套真源 | `vw` 收敛为同一控件工厂的语法糖 |
| `ui/src/diff.tie` 有脏矩形但帧循环不消费 | `app.tie` 消费脏矩形，空闲零重绘 |
| `ui/src/hit.tie` 无焦点/键盘/hover | 事件轴按 `tiu-event-system.md` 补齐 |
| `engine/src/glyph.tie` 5×7 位图，`raster.tie` 忽略 `TEXT_SIZE` | TTF `glyf` 解析 + 扫描线覆盖率抗锯齿 + atlas 缓存 |

---

## 12. 验收 / Acceptance

* 黄金矩形探针：`tests/layout_golden.tie`，四种容器 × 每种尺寸形态各一条，断言精确矩形。
* 端到端三示例：表单、列表（滚动 + 截断）、卡片网格（overlay 角标）。
* 文字几何一致：同一字符串在探针里断言「布局宽 = 绘制宽 = 度量宽」。
* 差分生效：改一个按钮文本，绘制记录只落在其脏矩形内。
* 空闲零重绘：无输入时连续 60 帧绘制记录数为 0。

---

## 13. 未决 / Open items

* `grid` 的 `span` 与自动流密集填充（暂不支持，先按固定列 + 行优先）。
* 动画与布局的关系（变换不触发重排；尺寸动画走后置几何）。
* CJK 字体：TrueType 轮廓可用系统 `simsun.ttc`/`msyh.ttc`，需 TTC 解析；CFF/OTF 大字库不在范围内。
* `ch`（字符宽）单位：需要字符度量约定，暂不提供。
