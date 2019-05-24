---
title: Retina屏幕处理
date: 2019-05-23 11:26:50
tags:
  - 前端
---

## 何为视网膜（Retina）屏

```
α＝2tan-1（h/2d）
```

这个公式建立在对于人类视力的研究基础上，其中“α”代表视角，理论上认为人眼能辨识所视物的最小视角是 0.78 弧分度（1 弧分度＝ 1/60 度）。在理论数据的基础上，考虑到环境光线对成像质量的影响，数据上通常取人眼的最小视角为 1 弧分度（1/60 度）。

<!-- more -->

另外，需要说明的是，1 弧分度数据主要基于视力 20/20（等效于我们熟知的 1.0）的统计样本，视力超常的（如 2.0 的视力）个体无疑会有着更出色的视觉能力，能辨识外物的最小视角会更小。

![retina](/assets/img/retina1.jpg)

基于以上的数据，在人类的最小视角（α）为定值的前提下，在某个视距（d，即设备离人眼的距离），简单说，当屏幕的像素间距小于此时（视距一定）的临界像素间距（可根据图中的公式来计算），或者说屏幕的分辨率（ppi）大于此时根据公式计算出的理论临界分辨率（ppi），即可认为该屏幕为视网膜屏。

也就是说，是否为 `Retina` 屏幕，不仅仅决定于 ppi（分辨率，或者说像素间距 h；或者说 1 英寸对角线长的正方形中含有的像素点个数；1 英寸/像素间距 h 英寸＝ ppi；简单公式：ppi = √(横向长度(px)² + 纵向长度(px)²) / 屏幕尺寸(inch)），还要看使用设备时与人眼的距离 d

1） 10 寸

h = tan(1/(2x60)) x (2x10) = tan(1/120) x 20 = tan(0.008333333) x 20 = 0.000145444 x 20 = 0.00290888

ppi = 1 英寸/像素间距 h 英寸 = 1/h = 1/0.00290888 = 343.774923682 = 344

2） 12 寸

h = tan(1/(2x60)) x (2x12) = tan(0.008333333) x 24 = 0.003490656

ppi = 1 英寸/像素间距 h 英寸 = 1/h = 1/0.003490656 = 286.479103068 = 286

3） 15 寸

h = tan(1/(2x60)) x (2x15) = tan(0.008333333) x 30 = 0.00436332

ppi = 1 英寸/像素间距 h 英寸 = 1/h = 1/0.00436332 = 229.183282455 = 229

## Web 中的 Retina

`CSS` 像素是一个抽象概念，设备无关像素，简称-“DIPS”，`device-independent` 像素，主要使用在浏览器上，用来精确的度量（确定）`Web` 页面上的内容。

在标准情况下一个 `CSS` 像素对应一个设备像素。

```css
.box {
  width: 200px;
  height: 300px;
  font-size: 12px;
}
```

上面的代码，将会在显示屏设备上绘制一个 200x300 像素的盒子，在标准屏幕下，它占据的就是 200x300 设备像素。但是在 `Retina` 屏幕下，相同的 div 却使用了 400x600 设备像素，保持相同的物理尺寸显示，导致每个像素点实际上有 4 倍的普通像素点。一个 `CSS` 像素点实际分成了四个，造成颜色肯定会存在偏差（非全保真的显示），于是，我们看上去就变得模糊了（特别是图片，非常的明显）。

**devicePixelRatio 设备像素比**

`window.devicePixelRatio` 是设备上物理像素和设备独立像素(`device-independent pixels` (dips))的比例。

公式表示就是：

```
window.devicePixelRatio = 物理像素 / dips
```

- 普通密度桌面显示屏的 devicePixelRatio=1
- 高密度桌面显示屏(Mac Retina)的 devicePixelRatio=2
- 主流手机显示屏的 devicePixelRatio=2 或 3

## Web 适配 Retina

有两种方式，一种 `CSS` 媒体查询，一种 `JS` 查询。

### CSS 媒体查询

这种方式需要在每个需要适配 `Retina` 屏幕的地方都需要加上比较麻烦（当然只有一些小图片需要适配，通常大图不会变的很模糊），

```css
#element {
  background-image: url('hires.png');
}

@media only screen and (min-device-pixel-ratio: 2) {
  #element {
    background-image: url('hires@2x.png');
  }
}

@media only screen and (min-device-pixel-ratio: 3) {
  #element {
    background-image: url('hires@3x.png');
  }
}
```

**重点来了~~**

不可能在用到图片的地方都写上这么一大坨吧，太不友好了。用过 `postcss` 的读者应该比较熟悉 `autoprefixer`，会自动帮我们补全一些针对特定浏览器的前缀，幸运的是 `postcss` 有个官方插件 `postcss-at2x` 可以帮我们省去上述一堆操作，只需要在图片后面加上 `at-2x` 标签即可：

配置插件（这里只展示 `postcss.config.js` 的方式，其他方式读者可根据 `postcss` 文档自行修改）：

```js
module.exports = {
  plugins: [require('postcss-at2x')]
}
```

输入：

```css
.multi {
  background: url(http://example.com/image.png), linear-gradient(
      to right,
      rgba(255, 255, 255, 0),
      rgba(255, 255, 255, 1)
    ), green, url(/public/images/cool.png) at-2x;
}
```

输出：

```css
.multi {
  background: url(http://example.com/image.png), linear-gradient(
      to right,
      rgba(255, 255, 255, 0),
      rgba(255, 255, 255, 1)
    ), green, url(/public/images/cool.png);
}

@media (min-device-pixel-ratio: 1.5),
  (min-resolution: 144dpi),
  (min-resolution: 1.5dppx) {
  .multi {
    background-image: url(http://example.com/image.png), linear-gradient(
        to right,
        rgba(255, 255, 255, 0),
        rgba(255, 255, 255, 1)
      ), none, url(/public/images/cool@2x.png);
  }
}
```

是不是非常好用！奉上 `Github` 地址 [postcss-at2x](https://github.com/simonsmith/postcss-at2x)。

### JS 查询

除了第一反应用的 `CSS` 媒体查询，其实也可以使用 `JS` 查询，自然是判断 `window.devicePixelRatio` 是否大于 1 了，但是这个有浏览器兼容性问题，具体查看 [https://caniuse.com/#feat=devicepixelratio](https://caniuse.com/#feat=devicepixelratio)，基本上现代浏览器都是能使用的。

通常使用的库为 `retinajs`，`Github` 地址 [retinajs](https://github.com/strues/retinajs)。使用方式有很多种，可以用 `JavaScript` 引入的方式，也可以在 `CSS` 预处理器中加入。

1.JavaScript

- 旧方式

  ```html
  <script type="text/javascript" src="/scripts/retina.min.js"></script>
  ```

  然后在任何想要重置的时候调用 `retinajs()` 即可。

  ```js
  retinajs()
  // Finds all images not previously processed and processes them.

  retinajs([img, img, img])
  // Only attempts to process the images in the collection.

  retinajs($('img'))
  // Same.

  retinajs(document.querySelectorAll('img'))
  // Same.
  ```

- 新方式

直接使用 `import` 或 `require`，一般针对 `Gulp/Webpack/Grunt` 等打包工具打包应用时使用。

```js
import retina from 'retina'

window.addEventListener('load', retina)
```

当然也可以像老方式一样使用。

2.CSS 预处理器（**推荐使用！**）

如 `SCSS`、`LESS` 等都有 `retinajs` 的 `mixin`，可以在 [https://github.com/strues/retinajs/tree/master/src](https://github.com/strues/retinajs/tree/master/src) 下载。

这种方式较为方便，省去了好多麻烦~~
