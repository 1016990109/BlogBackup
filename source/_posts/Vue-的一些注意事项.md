---
title: Vue 的一些注意事项
date: 2018-04-17 15:47:16
tags:
    - 前端
---

最近准备学习一波 Vue，因为有 React 的基础，所以学起来倒也不是很吃力。下面是一些在学习中遇到的可能需要注意的地方。

## 模板语法

### 插值

> 1.  绝对不要使用用户的输入作为插值，可能造成 XSS 攻击。
> 2.  每个绑定只能包含单个表达式，下面表达式都不会生效。

<!-- more -->

```html
<!-- 这是语句，不是表达式 -->
{{ var a = 1 }}

<!-- 流控制也不会生效，请使用三元表达式 -->
{{ if (ok) { return message } }}
```

### 指令

.prevent 修饰符告诉 v-on 指令对于触发的事件调用 event.preventDefault():

```html
<form v-on:submit.prevent="onSubmit">...</form>
```

## 计算属性和侦听器

### 计算属性

默认只有 getter，但是也可以有 setter

### 侦听器

何时使用侦听器（watch）:当需要在数据变化时执行异步或开销较大的操作。

## 条件渲染

### key

Vue 默认情况下是会复用元素的，例如切换用户名或邮箱登录，如果两者都有 input 元素，那么在切换的时候 input 不会被替换掉，只会更改 placeholder 之类的属性。

如果添加了唯一的 key 值，Vue 将不会再复用元素， 使用 key 来判断是否元素变更。

### v-show

注意，v-show 不支持 <template\> 元素，也不支持 v-else，如下：

```html
<template v-show="!show">
    <!-- will show 'template-show' -->
    <div>template-show</div>
</template>

<div v-show="!show">
    if
</div>
<!-- can't use v-else after v-show -->
```

### v-for 与 v-if

v-for 的优先级更高，所以可以对每一项进行 if 判断是否显示。

## 数组更新检测

直接改变数组内容的称为变异方法，如 push、pop、shift、unshift、splice、sort 等，可以响应更新；而直接生成新数组的方法如 slice、concat、filter 等则需要对 data 进行赋值，如

```javascript
example1.items = example1.items.filter(function(item) {
  return item.message.match(/Foo/)
})
```

### 不能检测更新

利用索引更改，arr[index] = newValue，或者改变数组长度，arr.length = newLength，可以用下面两种方式实现更新：

```javascript
// Vue.set
Vue.set(vm.items, indexOfItem, newValue)
// Array.prototype.splice
vm.items.splice(indexOfItem, 1, newValue)
//change length
vm.items.splice(newLength)
```

## 事件处理

@keyup.ctrl 控制时，仅仅按下 ctrl 并弹起是无用的，其他键必须同时按下才有效。

## 表单输入绑定

### 基础用法

v-model 绑定时，如果是基于输入法（中文、日文等）不会实时更新，只是输入结束后才会更新。

## 组件

### DOM 模板  解析注意事项

> 当使用 DOM 作为模板时 (例如，使用 el 选项来把 Vue 实例挂载到一个已有内容的元素上)，你会受到 HTML 本身的一些限制，因为 Vue 只有在浏览器解析、规范化模板之后才能获取其内容。尤其要注意，像 <ul\>、<ol\>、<table\>、<select\> 这样的元素里允许包含的元素有限制，而另一些像 <option\> 这样的元素只能出现在某些特定元素的内部。在自定义组件中使用这些受限制的元素时会导致一些问题，例如：

```html
<table>
  <my-row>...</my-row>
</table>
```

自定义组件 <my-row> 会被当作无效的内容，因此会导致错误的渲染结果。变通的方案是使用特殊的 is 特性：

```html
<table>
  <tr is="my-row"></tr>
</table>
```

但是在.vue 文件或者使用字符串模板（template: '<div\>123</div\>'）则不会有这个问题。

### Prop

class 和 style 会合并属性，父组件值和组件内的值进行合并

### 给组件绑定原生事件

```html
<my-component v-on:click.native="doTheThing"></my-component>
```

### .sync 修饰符

2.0 移除但是 2.3 版本又加了回来，但是变成了编译的语法糖，会自动添加 v-on 绑定，如下代码：

```html
<comp :foo.sync="bar"></comp>
```

会被扩展为：

```html
<comp :foo="bar" @update:foo="val => bar = val"></comp>
```

当子组件需要更新 foo 的值时，它需要显式地触发一个更新事件：

```js
this.$emit('update:foo', newValue)
```

### 非父子组件之间的通信

```js
var bus = new Vue()

// 触发组件 A 中的事件
bus.$emit('id-selected', 1)

// 在组件 B 创建的钩子中监听事件
bus.$on('id-selected', function(id) {
  // ...
})
```

### 使用插槽分发内容

> 除非子组件模板包含至少一个 **<slot\>** 插口，否则父组件的内容将会被**丢弃**。

```html
<div>
  <h2>我是子组件的标题</h2>
  <slot>
    只有在没有要分发的内容时才会显示。
  </slot>
</div>
```

### 作用域插槽

> **<slot\>** 元素可以用一个特殊的特性 name 来进一步配置如何分发内容。多个插槽可以有不同的名字。具名插槽将匹配内容片段中有对应 slot 特性的元素。

> 在 2.5.0+，**slot-scope** 能被用在任意元素或组件中而不再局限于 **<template\>**。

### 解构

slot-scope 支持解构，如下：

```html
<child>
  <span slot-scope="{ text }">{{ text }}</span>
</child>
```

其中 { text } 将子组件传递来的值进行解构，比如子组件有值 obj: {text: 'test message'}。

### 动态组件

> 通过使用保留的 <component> 元素，并对其 is 特性进行动态绑定，你可以在同一个挂载点动态切换多个组件：

```js
var vm = new Vue({
  el: '#example',
  data: {
    currentView: 'home' //也可以是对象组件
  },
  components: {
    home: {
      /* ... */
    },
    posts: {
      /* ... */
    },
    archive: {
      /* ... */
    }
  }
})
```

```html
<component v-bind:is="currentView">
  <!-- 组件在 vm.currentview 变化时改变！ -->
</component>
```

### 组件间的循环引用

常见的情况是文件系统，文件夹(folder)包含的内容(content)可能包含文件夹(folder)，这样就形成了循环引用，当使用 Vue.component 将这两个组件注册为全局组件的时候，框架会自动为你解决这个矛盾。然而，如果你使用诸如 webpack 或者 Browserify 之类的模块化管理工具来 require/import 组件的话，就会报错了：

```
Failed to mount component: template or render function not defined.
```

我们可以选择在文件夹(folder)组件中声明在 beforeCreate 时才注册组件：

```js
beforeCreate: function () {
  this.$options.components.TreeFolderContents = require('./tree-folder-contents.vue').default
}
```

### v-once

使用 v-once 指令使得模板只会渲染一遍而不会监听数据改变。

***

写论文去了，之后再更新...
