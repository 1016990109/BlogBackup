---
title: Web Components
date: 2018-11-28 15:37:35
tags:
  - 前端
---

# 自定义元素(Custom Elements)

## 简介

借助自定义元素，网络开发者可以创建新的 `HTML` 标记、扩展现有 `HTML` 标记，或者扩展其他开发者编写的组件。

## 定义新元素

定义一个新元素需要使用 `customElements.define()`：

<!-- more -->

```js
// 一般情况，普通类
class AppDrawer extends HTMLElement {
  constructor() {
    super()
    this.private = 'private'
  }

  static get observedAttributes() {
    return ['disabled', 'open']
  }

  // 开放属性的 getter 和 setter
  get open() {
    console.log('get property')
    return this.getAttribute('open')
  }

  set open(val) {
    console.log('set property')
    if (val) {
      this.setAttribute('open')
    } else {
      this.removeAttribute('open')
    }
  }
}

// 必须使用短横线
customElements.define('app-drawer', AppDrawer)

// 匿名类
window.customElements.define('app-drawer', class extends HTMLElement {...});
```

> 名称必须包含短横线（不能无短横线，也不能是下划线）。
> 不能多次注册同一标记。否则，将产生 `DOMException`。让浏览器了解新标记后，它就这样定了下来，不能撤回。
> 自定义元素不能自我封闭，因为 `HTML` 仅允许少数元素自我封闭。必须编写封闭标记 (`<app-drawer></app-drawer>`)。

## 扩展元素

可扩展自定义的元素，也可以扩展原生元素，使用 `extends` 来扩展。

```js
class FancyDrawer extends AppDrawer {
  constructor() {
    super(); // always call super() first in the constructor. This also calls the extended class' constructor.
    ...
  }
}

customElements.define('fancy-app-drawer', FancyDrawer);
```

扩展原生元素需要继承内置的元素比如：`HTMLButtonElement`、`HTMLQuoteElement`，且 `define` 必须传第三个参数以声明是扩展哪个具体的元素：

```js
// 扩展内置 html 元素
class CustomButton extends HTMLButtonElement {
  constructor() {
    super()

    this.style.backgroundColor = 'blue'

    this.addEventListener('click', () => {
      console.log('click')
    })
  }
}

// 扩展原生元素第三个参数需要告知是哪个原生组件
customElements.define('custom-button', CustomButton, { extends: 'button' })
```

自定义内置元素的用户有多种方法来使用该元素。他们可以通过在原生标记上添加 `is=""` 属性来声明(某些浏览器不推荐使用 `is`)：

```html
<!-- This <button> is a custom button. -->
<!-- 某些浏览器不推荐使用is  -->
<button is="custom-button" disabled>Custom button!</button>
```

或者在 `JavaScript` 中创建实例：

```js
// Custom elements overload createElement() to support the is="" attribute.
let button = document.createElement('button', { is: 'fancy-button' })
button.textContent = 'Fancy button!'
button.disabled = true
document.body.appendChild(button)
```

或者使用 `new` 运算符：

```js
let button = new FancyButton()
button.textContent = 'Fancy button!'
button.disabled = true
```

## 创建限制

1.创建自定义元素或扩展元素必须在 `</body>` 前，不允许放在 `<head></head>` 中。

2.自定义元素构造函数约束：

- 一个无参数的调用 `super()` 必须在构造函数体的第一条语句，树立正确的原型链，这是运行任何进一步的代码前提。
- 一个 `return` 语句不能在构造函数体内的任何地方出现，除非它是一个简单的返回（`return` 或 `return this`）。
- 构造函数中不能使用 `document.write()` 或 `document.open()` 方法。
- 该元素的属性和 `children` 获取不了，如不升级(即 `customElements.define()`)的情况下，是不会存在的。
- 该元素不能获得任何属性或 `children`，因为这违反了使用 `createElement` 或 `createElementNS` 方法的限制。
- 在一般情况下，工作应尽可能推迟到 `connectedCallback`，尤其是抓取资源或渲染。但是请注意，`connectedCallback` 可以调用一次以上，从而需要保证只运行了一次，以阻止其运行两次任何初始化工作。
- 一般来说，构造函数是用来建立初始状态和默认值的，并设置事件侦听器和可能的 `Shadow Root`。

## 自定义元素响应

自定义元素可以定义特殊生命周期钩子，以便在其存续的特定时间内运行代码。

| 名称                                               | 调用时机                                                                                                                                       |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| constructor                                        | 创建或升级元素的一个实例。用于初始化状态、设置事件侦听器或创建 Shadow DOM。参见规范，了解可在 constructor 中完成的操作的相关限制。             |
| connectedCallback                                  | 元素每次插入到 DOM 时都会调用。用于运行安装代码，例如获取资源或渲染。一般来说，应将工作延迟至合适时机执行。                                    |
| disconnectedCallback                               | 元素每次从 DOM 中移除时都会调用。用于运行清理代码（例如移除事件侦听器等）。                                                                    |
| attributeChangedCallback(attrName, oldVal, newVal) | 属性添加、移除、更新或替换。解析器创建元素时，或者升级时，也会调用它来获取初始值。Note: 仅 observedAttributes 属性中列出的特性才会收到此回调。 |
| adoptedCallback()                                  | 自定义元素被移入新的 document（例如，有人调用了 document.adoptNode(el)）。或者从一个 iframe 移动到另一个 iframe。                              |

> 浏览器对在 `attributeChangedCallback()` 数组中添加到白名单的任何属性调用 `observedAttributes`（请参阅[保留对属性](#保留对属性的更改)的更改）。实际上，这是一项性能优化。当用户更改一个通用属性（如 `style` 或 `class`）时，不希望出现大量的回调。

> 响应回调是同步的。如果对的元素调用 `el.setAttribute(...)`，浏览器将立即调用 `attributeChangedCallback()`。 同理，从 `DOM` 中移除元素（例如用户调用 `el.remove()`）后，就会立即收到 `disconnectedCallback()`。

## 属性和特性（property and attribute）

### 将属性(property)映射为特性(attribute)

`HTML` 属性通常会将其值以 `HTML` 特性的形式映射回 `DOM`。例如，如果 `hidden` 或 `id` 的值在 `JS` 中发生变更：

```js
div.id = 'my-id'
div.hidden = true
```

值将以特性的形式应用于活动 `DOM`：

```html
<div id="my-id" hidden></div>
```

这称为“将属性映射为特性”。几乎所有的 `HTML` 属性都会如此。为何？特性也可用于以声明方式配置元素，且无障碍功能和 `CSS` 选择器等某些 `API` 依赖于特性工作。关于 `property` 和 `attribute` 的关系可以查看 [前端基础之 JS——attribute 和 property](/2018/09/08/前端基础之JS/#attribute-和-property)。

### 保留对属性的更改

`HTML` 属性可方便地让用户声明初始状态：

```js
<app-drawer open disabled />
```

元素可通过定义 `attributeChangedCallback` 来对属性的更改作出响应。对于 `observedAttributes` 数组中列出的每一属性更改，浏览器都将调用此方法。

```js
class AppDrawer extends HTMLElement {
  ...

  static get observedAttributes() {
    return ['disabled', 'open'];
  }

  get disabled() {
    return this.hasAttribute('disabled');
  }

  set disabled(val) {
    if (val) {
      this.setAttribute('disabled', '');
    } else {
      this.removeAttribute('disabled');
    }
  }

  // Only called for the disabled and open attributes due to observedAttributes
  attributeChangedCallback(name, oldValue, newValue) {
    // When the drawer is disabled, update keyboard/screen reader behavior.
    if (this.disabled) {
      this.setAttribute('tabindex', '-1');
      this.setAttribute('aria-disabled', 'true');
    } else {
      this.setAttribute('tabindex', '0');
      this.setAttribute('aria-disabled', 'false');
    }
    // TODO: also react to the open attribute changing.
  }
}
```

## 元素升级

### 自定义元素可以在定义注册之前使用

渐进式增强是自定义元素的一项特点。换句话说，可以在页面声明多个 `<app-drawer>` 元素，并在等待较长的时间之后才调用 `customElements.define('app-drawer', ...)`。之所以会这样，原因是浏览器会因为存在未知标记而采用不同方式处理潜在自定义元素。调用 `define()` 并将类定义赋予现有元素的过程称为“元素升级”。

要了解标记名称何时获得定义，可以使用 `window.customElements.whenDefined()`。它提供可在元素获得定义时进行解析的 `Promise`。

```js
customElements.whenDefined('app-drawer').then(() => {
  // 当组件加载完，比如获取完需要的资源时候进行的操作
  console.log('app-drawer defined')
})
```

## 元素定义的内容

### 创建使用 Shadow DOM 的元素

元素的内容推荐使用 `Shadow DOM` 的 `API` 来创建，因为使用其他的 `API` 来填充元素内容会覆盖用户元素的子项(`<custom-button><div></div</custom-button>` 中的 `div` 会被舍弃)，这和用户预期的可能不太一样。

`Shadow DOM` 提供了一种方法，可让元素以独立于页面其余部分的方式拥有和渲染 `DOM` 并设置其样式。 甚至可以使用一个标记来隐藏整个应用：

```html
<!-- chat-app 具体的实现细节隐藏在 Shadow DOM. -->
<chat-app></chat-app>
```

要在自定义元素中使用 `Shadow DOM`，可在 `constructor` 内调用 `this.attachShadow`，`slot` 标签包含的内容就是用户自定义的内容。

```js
customElements.define(
  'x-foo-shadowdom',
  class extends HTMLElement {
    constructor() {
      super() // always call super() first in the constructor.

      // Attach a shadow root to the element.
      let shadowRoot = this.attachShadow({ mode: 'open' })
      // innerHTML 或者其他 API 使用就略显丑陋，推荐 template
      shadowRoot.innerHTML = `
      <style>:host { ... }</style> <!-- look ma, scoped styles -->
      <b>I'm in shadow dom!</b>
      <slot></slot>
    `
    }
    // ...
  }
)
```

### 通过 <template> 创建元素

`template` 片段在页面加载时解析并驻留，且于后续运行时激活。它是网页组件家族中的另一 `API` 原语。**模板是声明自定义元素结构的理想之选**。

使用示例：

```html
<!-- 这部分是不会渲染的 -->
<template id="x-foo-from-template">
  <style>
    p {
      color: orange;
    }
  </style>
  <p>I'm in Shadow DOM.My markup was stamped from a &lt;template&gt;.</p>
</template>

<script>
  customElements.define(
    'x-foo-from-template',
    class extends HTMLElement {
      constructor() {
        super() // always call super() first in the constructor.
        let shadowRoot = this.attachShadow({ mode: 'open' })
        const t = document.querySelector('#x-foo-from-template')
        const instance = t.content.cloneNode(true)
        shadowRoot.appendChild(instance)
      }
      // other code
    }
  )
</script>
```

## 设置自定义元素样式

自定义元素样式和原生元素样式使用方式一样：

```css
app-drawer {
  background: red;
}
```

> 用户定义样式优先级大于 `Shadow DOM` 中定义的样式。

### 预设置未注册元素的样式

还没有调用 `customElements.define` 的未定义元素可以使用 `CSS` 中 `:defined` 伪类来定义目标。

在定义前隐藏元素：

```css
app-drawer:not(:defined) {
  /* Pre-style, give layout, replicate app-drawer's eventual styles, etc. */
  display: inline-block;
  height: 100vh;
  opacity: 0;
  transition: opacity 0.3s ease-in-out;
}
```

## 其他

浏览器支持非标准元素，例如 `<randomtagthatdoesntexist></<randomtagthatdoesntexist>` 在浏览器中也能正常解析，`HTML` 规范允许这样。规范没有定义的元素作为 `HTMLUnknownElement` 进行解析。自定义元素则并非如此。如果在创建时使用有效的名称（包含“-”），则潜在的自定义元素将解析为 `HTMLElement`，就是说 `<has-line></has-line>` 会被解析为 `HTMLElement`，这和普通非标准元素不同，所以严格要求自定义元素名称必须包含中横线。

## 结论（Web Fundamentals 自定义元素 v1：可重用网络组件）

自定义元素提供了一种新工具，可让我们在浏览器中定义新 `HTML` 标记并创建可重用的组件。将它们与 `Shadow DOM` 和 `<template>` 等新平台原语结合使用，我们可开始实现网络组件的宏大图景：

- 创建和扩展**可重复**使用组件的跨浏览器（网络标准）。
- 无需库或框架即可使用。**原生** `JS/HTML` 威武！
- 提供熟悉的编程模型。仅需使用 `DOM/CSS/HTML`。
- 与其他网络平台功能良好匹配（`Shadow DOM`、`<template>`、`CSS` 自定义属性等）
- 与浏览器的 `DevTools` 紧密集成。
- 利用现有的无障碍功能。

# Shadow DOM(Shadow DOM v1 规范)

## 简介

`Shadow DOM` 解决了构建网络应用的脆弱性问题。脆弱性是由 `HTML`、`CSS` 和 `JS` 的全局性引起的，例如同一个 `class` 可能会在多处定义，造成了用户不期望的覆盖，这就逼迫开发者使用 `!important`，最终使得代码可读性变得很差。

`Shadow DOM` 修复了 `CSS` 和 `DOM`。它在网络平台中引入作用域样式。无需工具或命名约定，即可使用原生 `JavaScript` 捆绑 `CSS` 和标记、隐藏实现详情以及编写独立的组件。

`Shadow DOM` 作用：

- 隔离 `DOM`：组件的 `DOM` 是独立的（例如，`document.querySelector()` 不会返回组件 `Shadow DOM` 中的节点）。
- 作用域 `CSS`：`Shadow DOM` 内部定义的 `CSS` 在其作用域内。样式规则不会泄漏，页面样式也不会渗入。
- 组合：为组件设计一个声明性、基于标记的 `API`。
- 简化 `CSS`： 作用域 `DOM` 意味着可以使用简单的 `CSS` 选择器，更通用的 `id/类` 名称，而无需担心命名冲突。
- 效率： 将应用看成是多个 `DOM` 块，而不是一个大的（全局性）页面。

## 什么是 Shadow DOM

`Shadow DOM` 与普通 `DOM` 相同，但有两点区别：

1. 创建/使用的方式；
2. 与页面其他部分有关的行为方式。

通常，创建 `DOM` 节点并将其附加至其他元素作为子项。借助于 `Shadow DOM`，可以创建作用域 `DOM` 树，该 `DOM` 树附加至该元素上，但与其自身真正的子项分离开来。这一作用域子树称为影子树。被附着的元素称为影子宿主。在影子中添加的任何项均将成为宿主元素的本地项，包括 `<style>`。 这就是 `Shadow DOM` 实现 `CSS` 样式作用域的方式。

## 创建 Shadow DOM

影子根是附加至“宿主”元素的文档片段。使用 `element.attachShadow()` 创建 `Shadow DOM`：

```js
const header = document.createElement('header')
const shadowRoot = header.attachShadow({ mode: 'open' })
shadowRoot.innerHTML = '<h1>Hello Shadow DOM</h1>' // Could also use appendChild().

// header.shadowRoot === shadowRoot
// shadowRoot.host === header
```

规范定义了元素列表，这些元素无法托管影子树，可托管的元素查看 [https://developer.mozilla.org/en-US/docs/Web/API/Element/attachShadow](https://developer.mozilla.org/en-US/docs/Web/API/Element/attachShadow) 元素之所以在所选之列，其原因如下：

- 浏览器已为该元素托管其自身的内部 `shadow DOM`（`<textarea>`、`<input>`）。
- 让元素托管 `shadow DOM` 毫无意义 (`<img>`)。

为自定义元素创建 `shadow DOM` 在上文[自定义元素](#自定义元素-Custom-Elements)中已经提到了。

## 组合和 slot

### Light DOM

组件用户编写的标记。该 `DOM` 不在组件 `shadow DOM` 之内。 它是元素实际的子项。

```html
<button is="better-button">
  <!-- the image and span are better-button's light DOM -->
  <img src="gear.svg" slot="icon" /> <span>Settings</span>
</button>
```

### Shadow DOM

该 `DOM` 是由组件的作者编写。`Shadow DOM` 对于组件而言是本地的，它定义内部结构、作用域 `CSS` 并封装实现详情。它还可定义如何渲染由组件使用者编写的标记。

```html
#shadow-root
<style>
  ...;
</style>
<slot name="icon"></slot> <span id="wrapper"> <slot>Button</slot> </span>
```

### 扁平的 DOM 树

浏览器将用户的 `light DOM` 分布到 `shadow DOM` 的结果，对最终产品进行渲染。扁平树是指在 `DevTools` 中最终看到的树以及在页面上渲染的对象。

```html
<button is="better-button">
  #shadow-root
  <style>
    ...;
  </style>
  <slot name="icon"> <img src="gear.svg" slot="icon" /> </slot>
  <slot> <span>Settings</span> </slot>
</button>
```

### <slot> 元素

`Shadow DOM` 使用 `<slot>` 元素将不同的 `DOM` 树组合在一起。`slot` 是组件内部的占位符，用户可以使用自己的标记来填充。

通过定义一个或多个 `slot`，可将外部标记引入到组件的 `shadow DOM` 中进行渲染。这相当于在说“在此处渲染用户的标记”。

**这个和 `Vue` 中的 `slot` 很像。**

组件可在其 `shadow DOM` 中定义零个或多个 `slot`。`slot` 可以为空，或者提供回退内容。如果用户不提供 `light DOM` 内容，`slot` 将对其备用内容进行渲染。

```html
<!-- Default slot. If there's more than one default slot, the first is used. -->
<slot></slot>

<!-- 备用内容的 slot -->
<slot>Fancy button</slot>
<!-- default slot with fallback content -->

<slot>
  <!-- default slot entire DOM tree as fallback -->
  <h2>Title</h2>
  <summary>Description text</summary>
</slot>
```

还可以创建指定名称的 `slot`:

```html
#shadow-root
<div id="tabs"><slot id="tabsSlot" name="title"></slot></div>
<div id="panels"><slot id="panelsSlot"></slot></div>
```

```html
<fancy-tabs>
  <!-- 三个 button 会插入到 id 为 tabsSlot 的 slot 中 -->
  <button slot="title">Title</button>
  <button slot="title" selected>Title 2</button>
  <button slot="title">Title 3</button>
  <!-- 其他的 light DOM 都会插入到默认 slot 中，即 id 为 panelsSlot 的 slot 中 -->
  <section>content panel 1</section>
  <section>content panel 2</section>
  <section>content panel 3</section>
</fancy-tabs>
```

结果(扁平的 DOM 树)：

```html
<fancy-tabs>
  #shadow-root
  <div id="tabs">
    <slot id="tabsSlot" name="title">
      <button slot="title">Title</button>
      <button slot="title" selected>Title 2</button>
      <button slot="title">Title 3</button>
    </slot>
  </div>
  <div id="panels">
    <slot id="panelsSlot">
      <section>content panel 1</section>
      <section>content panel 2</section>
      <section>content panel 3</section>
    </slot>
  </div>
</fancy-tabs>
```

## 设定样式

### 组件定义的样式

通常的做法就是在 `template` 中定义样式，也可以使用样式表。如果需要为组件自身定义样式可以使用 `:host` 伪类：

```html
<style>
  :host {
    display: block; /* by default, custom elements are display: inline */
    contain: content; /* CSS containment FTW. */
  }
</style>
```

使用 `:host` 的一个问题是，父页面中的规则较之在元素中定义的 `:host` 规则具有更高的特异性。也就是说，外部样式优先。这可让用户从外部替换的顶级样式。此外，`:host` 仅在影子根范围内起作用，因此无法在 `shadow DOM` 之外使用，也就是必须在 `template` 中使用或者对 `shadow-root` 使用其他样式 `API` 来完成。

使用 `:host` 的一些例子：

```html
<style>
  :host {
    opacity: 0.4;
    will-change: opacity;
    transition: opacity 300ms ease-in-out;
  }
  :host(:hover) {
    opacity: 1;
  }
  :host([disabled]) {
    /* style when host has disabled attribute. */
    background: grey;
    pointer-events: none;
    opacity: 0.4;
  }
  :host(.blue) {
    color: blue; /* color host when it has class="blue" */
  }
  :host(.pink) > #tabs {
    color: pink; /* color internal #tabs node when host has class="pink". */
  }
</style>
```

### 基于情景设定样式

如果 `:host-context(<selector>)` 或其任意父级与 `<selector>` 匹配，它将与组件匹配。一个常见用途是根据组件的环境进行主题化。其中的 `<selector>` 是父级的选择器。

`:host-context()` 对于主题化很有用，但更好的方法是使用 `CSS` 自定义属性创建样式钩子。

### 为分布式节点设定样式

`::slotted(<compound-selector>)` 与分布到 `<slot>` 中的节点匹配。

### 从外部为组件设定样式

直接使用标记名称作为选择器即可。

## 高级主题

### 创建闭合影子根（应避免）

`shadow DOM` 的另一情况称为“闭合”模式。创建闭合影子树后，在 `JavaScript` 外部无法访问组件的内部 `DOM`。

创建的时候指定 `mode` 为 `close` 即可：

```js
const div = document.createElement('div')
const shadowRoot = div.attachShadow({ mode: 'closed' }) // close shadow tree
// div.shadowRoot === null 获取不到阴影，因为是关闭的
// shadowRoot.host === div
```

任何时候都不要使用 `{mode: 'closed'}` 来创建网络组件，有以下几点原因：

1. 人为的安全功能。没有什么能够阻止攻击者入侵 `Element.prototype.attachShadow`。
2. 闭合模式阻止自定义元素代码访问其自己的 `shadow DOM`。(这根本没用)
3. 闭合模式使组件对最终用户的灵活性大为降低。

### 在 JS 中使用 slot

#### slotchange 事件

当 `slot` 的分布式节点发生变化时，`slotchange` 事件会触发。例如，当用户从 `light DOM` 中添加/移除子项时。

```js
const slot = this.shadowRoot.querySelector('#slot')
slot.addEventListener('slotchange', e => {
  console.log('light dom children changed!')
})
```

> 当组件的实例首次初始化时，slotchange 不触发。

#### 哪些元素在 slot 中进行渲染

调用 `slot.assignedNodes()` 可查看 `slot` 正在渲染哪些元素。`{flatten: true}` 选项将返回 `slot` 的备用内容（前提是没有分布任何节点）。

#### 元素分配给哪个 Slot

`element.assignedSlot` 返回元素绑定的 `slot`。

### Shadow DOM 事件模型

当事件从 `shadow DOM` 中触发时，其目标将会调整为维持 `shadow DOM` 提供的封装。 也就是说，事件的目标重新进行了设定，因此这些事件看起来像是来自组件，而不是来自 `shadow DOM` 中的内部元素。

有些事件甚至不会从 `shadow DOM` 中传播出去。

确实会跨过影子边界的事件有：

- 聚焦事件：`blur`、`focus`、`focusin`、`focusout`
- 鼠标事件：`click`、`dblclick`、`mousedown`、`mouseenter`、`mousemove`，等等
- 滚轮事件：`wheel`
- 输入事件：`beforeinput`、`input`
- 键盘事件：`keydown`、`keyup`
- 组合事件：`compositionstart`、`compositionupdate`、`compositionend`
- 拖放事件：`dragstart`、`drag`、`dragend`、`drop`，等等

## 重置可继承样式

可继承样式（`background`、`color`、`font` 以及 `line-height` 等）可在 `shadow DOM` 中继续继承。也就是说，默认情况下它们会突破 `shadow DOM` 作用域限制，从自定义组件外部继承样式下来。如果想从头开始，可在它们超出影子边界时，使用 `all: initial;` 将可继承样式重置为初始值。
