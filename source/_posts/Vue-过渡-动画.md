---
title: Vue 过渡&动画
date: 2018-05-06 20:56:26
tags:
    - 前端
---

## 单元素/组件的过渡

使用 transition 封装组件：

```html
<transition name="fade">
    <p v-if="show">hello</p>
</transition>
```

<!-- more -->

### 过渡类名

使用 transition 之后当触发过渡动画时，会添加相应的 class，没有 name 属性时使用 v-，否则使用定义的{$name}-，共有 6 个 class 切换：

1.  v-enter：定义进入过渡的开始状态。在元素被插入之前生效，在元素被插入之后的下一帧移除。
2.  v-enter-active：定义进入过渡生效时的状态。在整个进入过渡的阶段中应用，在元素被插入之前生效，在过渡/动画完成之后移除。这个类可以被用来定义进入过渡的过程时间，延迟和曲线函数。
3.  v-enter-to: 2.1.8 版及以上 定义进入过渡的结束状态。在元素被插入之后下一帧生效 (与此同时 v-enter 被移除)，在过渡/动画完成之后移除。
4.  v-leave: 定义离开过渡的开始状态。在离开过渡被触发时立刻生效，下一帧被移除。
5.  v-leave-active：定义离开过渡生效时的状态。在整个离开过渡的阶段中应用，在离开过渡被触发时立刻生效，在过渡/动画完成之后移除。这个类可以被用来定义离开过渡的过程时间，延迟和曲线函数。
6.  v-leave-to: 2.1.8 版及以上 定义离开过渡的结束状态。在离开过渡被触发之后下一帧生效 (与此同时 v-leave 被删除)，在过渡/动画完成之后移除。

具体过程如下：

![vue_transition](/assets/img/vue_transition.png)

### JavaScript 钩子

可以在属性中声明钩子:

```html
<transition
  v-on:before-enter="beforeEnter"
  v-on:enter="enter"
  v-on:after-enter="afterEnter"
  v-on:enter-cancelled="enterCancelled"

  v-on:before-leave="beforeLeave"
  v-on:leave="leave"
  v-on:after-leave="afterLeave"
  v-on:leave-cancelled="leaveCancelled"
>
  <!-- ... -->
</transition>
```

> 当只用 JavaScript 过渡的时候， 在 enter 和 leave 中，回调函数 done 是必须的 。否则，它们会被同步调用，过渡会立即完成。

> 推荐对于仅使用 JavaScript 过渡的元素添加 v-bind:css="false"，Vue 会跳过 CSS 的检测。这也可以避免过渡过程中 CSS 的影响。

## 多个元素的过渡

### 过渡  模式

* in-out：新元素先进行过渡，完成之后当前元素过渡离开。
* out-in：当前元素先进行过渡，完成之后新元素过渡进入。

## 平滑过渡(列表过渡)

使用 `<transition-group>`组件，可以使列表过渡起来比较平滑，只需要使用`v-move`即可，具体例子如下：

```html
<transition-group name="flip-list" tag="ul">
    <li v-for="item in items" v-bind:key="item">
      {{ item }}
    </li>
</transition-group>
```

```css
.flip-list-move {
  transition: transform 1s;
}
```

> 读者有兴趣可以研究一下洗牌算法，可以保证每个数出现的唯一性，shuffle 算法。

这个看起来很神奇，内部的实现，Vue 使用了一个叫 FLIP 简单的动画队列使用 transforms 将元素从之前的位置平滑过渡新的位置。

需要注意的是使用 FLIP 过渡的元素不能设置为 display: inline 。作为替代方案，可以设置为 display: inline-block 或者放置于 flex 中。

---

未完待续
