---
title: 前端基础之CSS
date: 2018-09-04 21:43:53
tags:
---

## CSS 选择器的优先级

内联样式 > `ID` 选择器 > 类选择器、属性选择器和伪类选择器 > 标签（类型）选择器和伪元素选择器

相同则比较数量每一级的数量总和。

<!-- more -->

当出现优先级相等的情况时，最晚出现的样式规则会被采纳。

## 重置（resetting）CSS 和 标准化（normalizing）CSS 的区别

- **重置（Resetting）**： 重置意味着除去所有的浏览器默认样式。对于页面所有的元素，像 `margin`、`padding`、`font-size` 这些样式全部置成一样。你将必须重新定义各种元素的样式。
- **标准化（Normalizing）**： 标准化没有去掉所有的默认样式，而是保留了有用的一部分，同时还纠正了一些常见错误。

## Float 定位

浮动（`float`）是 `CSS` 定位属性。浮动元素从网页的正常流动中移出，但是保持了部分的流动性，会影响其他元素的定位（比如文字会围绕着浮动元素）。这一点与绝对定位不同，绝对定位的元素完全从文档流中脱离。

`CSS` 的 `clear` 属性通过使用 `left`、`right`、`both`，让该元素向下移动（清除浮动）到浮动元素下面。

如果父元素只包含浮动元素，那么该父元素的高度将塌缩为 0。我们可以通过清除（`clear`）从浮动元素后到父元素关闭前之间的浮动来修复这个问题。一般方法是给父元素一个 `clearfix` 的类，内容如下：

```css
.clearfix::after {
  content: '';
  display: block;
  clear: both;
}
```

值得一提的是，把父元素属性设置为 `overflow: auto` 或 `overflow: hidden`，会使其内部的**子元素**形成块格式化上下文（`Block Formatting Context`），并且父元素会扩张自己，使其能够包围它的子元素。

## z-index

`CSS` 中的 `z-index` 属性控制重叠元素的垂直叠加顺序。**`z-index` 只能影响 `position` 值不是 `static` 的元素**。

没有定义 `z-index` 的值时，元素按照它们出现在 `DOM` 中的顺序堆叠（层级越低，出现位置越靠上）。非静态定位的元素（及其子元素）将始终覆盖静态定位（`static`）的元素，而不管 `HTML` 层次结构如何。

层叠上下文元素有如下特性：

- 层叠上下文的层叠水平要比普通元素高；
- 层叠上下文可以阻断元素的混合模式；
- 层叠上下文可以嵌套，内部层叠上下文及其所有子元素均受制于外部的层叠上下文。
- 每个层叠上下文和兄弟元素独立，也就是当进行层叠变化或渲染的时候，只需要考虑后代元素。
- 每个层叠上下文是自成体系的，当元素发生层叠的时候，整个元素被认为是在父层叠上下文的层叠顺序中。

**普通元素的层叠水平优先由层叠上下文决定**，因此，层叠水平的比较只有在当前层叠上下文元素中才有意义。

层叠顺序：

![stacked order](/assets/img/stacked_order.png)

层叠上下文的创建：

- 页面根元素
- 对于包含有 `position:relative` / `position:absolute` / `position:fixed` / `position:sticky` 的定位元素，且 `z-index` 不是 `auto`
- `z-index` 值不为 `auto` 的 `flex` 项(父元素 `display:flex|inline-flex`)，是 `flex` 布局的**子元素**
- 元素的 `opacity` 值不是 1
- 元素的 `transform` 值不是 `none`
- 元素 `mix-blend-mode` 值不是 `normal`
- 元素的 `filter` 值不是 `none`
- 元素的 `isolation` 值是 `isolate`
- `will-change` 指定的属性值为上面任意一个
- 元素的 `-webkit-overflow-scrolling` 设为 `touch`

具体的一些比较可查看 [深入理解 CSS 中的层叠上下文和层叠顺序](https://www.zhangxinxu.com/wordpress/2016/01/understand-css-stacking-context-order-z-index/)

## 格式化上下文(Block Formatting Context)

一个 HTML 盒（Box）满足以下任意一条，会创建块格式化上下文：

- `float` 的值不是 `none`.
- `position` 的值不是 `static` 或 `relative`.
- `display` 的值是 `table-cell`、`table-caption`、`inline-block`、`flex`、或 `inline-flex`。
- `overflow` 的值不是 `visible`。

### `BFC` 特性：

1. 内部的 `Box` 会在垂直方向，从顶部开始一个接一个地放置。
2. `Box` 垂直方向的距离由 `margin` 决定。属于同一个 `BFC` 的两个相邻 `Box` 的 `margin` 会发生叠加。(用来解决边距叠加问题，给会重叠的元素加一个 `BFC` 的父元素，那么这个 `BFC` 块就不会和下一个元素的边距重叠了，**水平方向的 `margin` 是不会叠加的**)
3. 每个元素的 `margin box` 的左边， 与包含块 `border box` 的左边相接触(对于从左往右的格式化，否则相反)。即使存在浮动也是如此。
4. `BFC` 的区域不会与 `float box` 叠加。(解决布局问题，一个块元素与浮动块重叠了的问题)
5. `BFC` 就是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面的元素，反之亦然。
6. 计算 `BFC` 的高度时，浮动元素也参与计算。(经常用来解决浮动元素导致父元素坍塌的问题)

## 不同浏览器的样式兼容性问题

1. 在确定问题原因和有问题的浏览器后，使用单独的样式表，仅供出现问题的浏览器加载。这种方法需要使用服务器端渲染。
2. 使用已经处理好此类问题的库，比如 `Bootstrap`。
3. 使用 `autoprefixer` 自动生成 `CSS` 属性前缀。(`webpack` 打包工具可使用 `postcss` 来完成)
4. 使用 `Reset.css` 或 `Normalize.css`。

## 如何为功能受限的浏览器提供页面

- 优雅的降级：为现代浏览器构建应用，同时确保它在旧版浏览器中正常运行。
- 渐进式增强：构建基于用户体验的应用，但在浏览器支持时添加新增功能。
- 利用 [caniuse.com](https://caniuse.com) 检查特性支持。
- 使用 `autoprefixer` 自动生成 `CSS` 属性前缀。
- 使用 [Modernizr](https://modernizr.com) 进行特性检测。

## 用 CSS 隐藏页面元素

1. `opacity` 设为 0。
2. `visibility` 设为 `hidden`。
3. `display` 设为 `none`。
4. `position` 设为 `absolute`，然后移动到不可见区域，`left: -99999px`。
5. `width: 0; height: 0; overflow: hidden`：使元素不占用屏幕上的任何空间，导致不显示它。
6. `text-indent: -9999px`：这只适用于 `block` 元素中的文本。
7. `clip(clip-path):rect()/inset()/polygon()`，注意只对 `position` 值为 `absolute` 或 `fixed` 的元素有效。
8. `transform: scale(0,0)`。
9. 利用 `z-index` 来隐藏。

## 媒体查询

```html
<!-- link元素中的CSS媒体查询 -->
<link rel="stylesheet" media="only screen and (max-width: 800px)" href="example.css" />

<!-- 样式表中的CSS媒体查询 -->
<style>
@media only screen and (max-width: 600px) {
  .facet_sidebar {
    display: none;
  }
}
</style>
```

`and` 关键字用于合并多个媒体属性或合并媒体属性与媒体类型。

媒体查询中使用逗号分隔效果等同于 `or` 逻辑操作符。

`not` 关键字应用于整个媒体查询，在媒体查询为假时返回真

更多的设置查看 [CSS 媒体查询](https://developer.mozilla.org/zh-CN/docs/Web/Guide/CSS/Media_queries)

## 解释浏览器如何确定哪些元素与 CSS 选择器匹配

浏览器从最右边的选择器（关键选择器）根据关键选择器，浏览器从 `DOM` 中筛选出元素，然后向上遍历被选元素的父元素，判断是否匹配。选择器匹配语句链越短，浏览器的匹配速度越快。

例如，对于形如 `p span` 的选择器，浏览器首先找到所有 `<span>` 元素，并遍历它的父元素直到根元素以找到 `<p>` 元素。对于特定的 `<span>`，只要找到一个 `<p>`，就知道已经匹配并停止继续匹配

## box-sizing

改变计算元素 `height` 与 `width` 的方法。

元素默认应用了 `box-sizing: content-box`，元素的宽高只会决定内容（`content`）的大小。

`box-sizing: border-box` 改变计算元素 `width` 和 `height` 的方式，`border` 和 `padding` 的大小也将计算在内。

## block、inline 与 inline-block 区别

|                                     | block                                                         | inline-block                               | inline                                                                                                             |
| ----------------------------------- | ------------------------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| 大小                                | 填充其父容器的宽度。                                          | 取决于内容。                               | 取决于内容。                                                                                                       |
| 定位                                | 从新的一行开始，并且不允许旁边有 `HTML` 元素（除非是`float`） | 与其他内容一起流动，并允许旁边有其他元素。 | 与其他内容一起流动，并允许旁边有其他元素。                                                                         |
| 能否设置 `width` 和 `height`        | 能                                                            | 能                                         | 不能。设置会被忽略。                                                                                               |
| 可以使用 `vertical-align` 对齐      | 不可以                                                        | 可以                                       | 可以                                                                                                               |
| 边距（`margin`）和填充（`padding`） | 各个方向都存在                                                | 各个方向都存在                             | 只有水平方向存在。垂直方向会被忽略。 尽管 border 和 padding 在 content 周围，但垂直方向上的空间取决于'line-height' |
| 浮动（`float`）                     | -                                                             | -                                          | 就像一个 `block` 元素，可以设置垂直边距和填充。                                                                    |

## relative、fixed、absolute 和 static

经过定位的元素，其 `position` 属性值必然是 `relative`、`absolute`、`fixed` 或 `sticky`。

- `static`：默认定位属性值。该关键字指定元素使用正常的布局行为，即元素在文档常规流中当前的布局位置。此时 `top`, `right`, `bottom`, `left` 和 `z-index` 属性无效。
- `relative`：该关键字下，元素先放置在未添加定位时的位置，再在不改变页面布局的前提下调整元素位置（因此会在此元素未添加定位时所在位置留下空白）。
- `absolute`：不为元素预留空间，通过指定元素相对于最近的非 `static` 定位祖先元素的偏移，来确定元素位置。绝对定位的元素可以设置外边距（`margins`），且不会与其他边距合并。
- `fixed`：不为元素预留空间，而是通过指定元素相对于屏幕视口（`viewport`）的位置来指定元素位置。元素的位置在屏幕滚动时不会改变。打印时，元素会出现在的每页的固定位置。`fixed` 属性会创建新的层叠上下文。**当元素祖先的 `transform` 属性非 `none` 时，容器由视口改为该祖先。**
- `sticky`：盒位置根据正常流计算(这称为正常流动中的位置)，然后相对于该元素在流中的 `flow root`（`BFC`）和 `containing block`（最近的块级祖先元素）定位。在所有情况下（即便被定位元素为 `table` 时），该元素定位均不对后续元素造成影响。当元素 B 被粘性定位时，后续元素的位置仍按照 B 未定位时的位置来确定。`position: sticky` 对 `table` 元素的效果与 `position: relative` 相同。须指定 `top`, `right`, `bottom` 或 `left` 四个阈值其中之一，才可使粘性定位生效。否则其行为与相对定位相同。

详情以及例子可查看 [position](https://developer.mozilla.org/zh-CN/docs/Web/CSS/position)。

## 响应式设计与自适应设计不同

用一张图片来描述更合适：

_上面是响应式设计，下面是自适应设计_ ![rwd-vs-adapt-example](/assets/img/rwd-vs-adapt-example.gif)

响应式设计和自适应设计都以提高不同设备间的用户体验为目标，根据视窗大小、分辨率、使用环境和控制方式等参数进行优化调整。

响应式设计的适应性原则：网站应该凭借一份代码，在各种设备上都有良好的显示和使用效果。响应式网站通过使用媒体查询，自适应栅格和响应式图片，基于多种因素进行变化，创造出优良的用户体验。就像一个球通过膨胀和收缩，来适应不同大小的篮圈。

自适应设计更像是渐进式增强的现代解释。与响应式设计单一地去适配不同，自适应设计通过检测设备和其他特征，从早已定义好的一系列视窗大小和其他特性中，选出最恰当的功能和布局。与使用一个球去穿过各种的篮筐不同，自适应设计允许使用多个球，然后根据不同的篮筐大小，去选择最合适的一个。

## 视网膜分辨率处理

使用媒体查询，像 `@media only screen and (min-device-pixel-ratio: 2) { ... }`，然后改变 `background-image`。

对于图标类的图形，尽可能使用 `svg` 和图标字体，因为它们在任何分辨率下，都能被渲染得十分清晰。

还有一种方法是，在检查了 `window.devicePixelRatio` 的值后，利用 `JavaScript` 将 `<img>` 的 `src` 属性修改，用更高分辨率的版本进行替换。**注意：`IE` 和 `FireFox` 是不支持 `devicePixelRatio` 属性的。**

## translate vs postion absolute

`translate(x, y)` 是 `transform` 的一个值。改变 `transform` 或 `opacity` 不会触发浏览器重新布局（`reflow`）或重绘（`repaint`），只会触发组合（`compositions`）。而改变绝对定位会触发重新布局，进而触发重绘和复合。`transform` 使浏览器为元素创建一个 `GPU` 图层，但改变绝对定位会使用到 `CPU。` 因此 `translate()` 更高效，可以缩短平滑动画的绘制时间。

当使用 `translate()` 时，元素仍然占据其原始空间（有点像 `position：relative`），这与改变绝对定位不同。
