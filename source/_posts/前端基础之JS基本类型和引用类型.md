---
title: 前端基础之JS基本类型和引用类型
date: 2018-09-12 09:24:47
tags:
  - 前端
  - 基本数据类型
---

`JS` 中一个变量可以存放两种类型的值：基本类型和引用类型。

<!-- more -->

`JS` 数据类型有 7 种(`ES6` 新增一种 `Symbol`):

基本类型(原始类型)：

- `Boolean`
- `Null`
- `Undefined`
- `Number`
- `String`
- `Symbol` (`ECMAScript` 6 新定义)，符号类型是唯一的并且是不可修改的

引用类型(复杂类型)：

- `Object`

## 基本类型

基本类型的值是按值访问

### 基本类型的值是不可变的

```js
var str = '123hello321'
str.toUpperCase() // 123HELLO321
console.log(str) // 123hello321
```

### 基本类型的比较是它们的值的比较

不同类型之间也可以比较，因为做了隐式转换。涉及隐式转换最多的两个运算符 `+` 和 `==`。

隐式转换中主要涉及到三种转换：

1. 将值转为原始值，`ToPrimitive()`。
2. 将值转为数字，`ToNumber()`。
3. 将值转为字符串，`ToString()`。

**通过 ToPrimitive 将值转换为原始值**

`ToPrimitive(input, PreferredType?)`

`input` 是要转换的值，`PreferredType` 是可选参数，可以是 `Number` 或 `String` 类型。他只是一个转换标志，转化后的结果并不一定是这个参数所值的类型，但是转换结果一定是一个原始值（或者报错）。

如果 `PreferredType` 被标记为 `Number`，那么先是调用 `valueOf` 方法，如果返回值是原始值则结束；否则重新调用 `toString` 方法，如果是原始值则结束，不然就会抛出 `TypeError` 异常。
如果 `PreferredType` 被标记为 `String`，那么先是调用 `toString` 方法，如果返回值是原始值则结束；否则重新调用 `valueOf` 方法，如果是原始值则结束，不然就会抛出 `TypeError` 异常。(两者相反)

> `valueOf()`：返回最适合该对象类型的原始值；
> `toString()`: 将该对象的原始值以字符串形式返回。

> 没有 `PrefferedType` 时，按照下面规则：如果该对象为 `Date` 类型，则 `PreferredType` 被设置为 `String`；否则，`PreferredType` 被设置为 `Number`。

**通过 ToNumber 将值转换为数字**

| 参数      | 结果                                       |
| --------- | ------------------------------------------ |
| undefined | NaN                                        |
| null      | +0                                         |
| 布尔值    | true 转为 1，false 转为                    |
| 字符串    | 能解析则变为数字，否则 NaN                 |
| 对象      | 先 ToPrimitive(input, Number)，再 ToNumber |

**通过 ToString 将值转换为字符串**

| 参数      | 结果                                       |
| --------- | ------------------------------------------ |
| undefined | 'undefined'                                |
| null      | 'null'                                     |
| 布尔值    | true 转为 'true'，false 转为 'false'       |
| 数字      | 直接转，NaN 变为 'NaN'                     |
| 对象      | 先 ToPrimitive(input, String)，再 ToString |

举例：

```js
console.log({} + {}) //"[object Object][object Object]"

//需要转换为原始类型，且没有指定 PrefferedType，所以优先 ToNumber
//{}.valueOf() 结果还是 {}
//继续用ToString，{}.toString() 结果为 "[object Object]"
```

```js
console.log(2 * {}) //NaN

//* 只能在 number 上运算，所以 PerfferedType 为 Number，ToPrimitive({}, Number)
//{}.valueOf() 结果还是 {}，继续 ToString
//{}.toString() 结果为 "[object Object]"
//再将 "[object Object]" 转为 Number，结果为 NaN
//最后 2*NaN 结果还是 NaN
```

**`==` 运算符时隐式转换规则**

1. x,y 为 `null`、`undefined` 两者中一个 // 返回 true
2. x、y 为 `Number` 和 `String` 类型时，则转换为 `Number` 类型比较。
3. 有 `Boolean` 类型时，`Boolean` 转化为 `Number` 类型比较。
4. 一个 `Object` 类型，一个 `String` 或 `Number` 类型，将 `Object` 类型进行原始转换后，按上面流程进行原始值比较。

```js
const a = {
  i: 1,
  toString: function() {
    return a.i++
  }
}
if (a == 1 && a == 2 && a == 3) {
  //会打印
  console.log('hello world!')
}
```

### 基本类型的变量是存放在栈内存（Stack）里的

```js
var a, b
a = 'zyj'
b = a
console.log(a) // zyj
console.log(b) // zyj
a = '呵呵' // 改变 a 的值，并不影响 b 的值
console.log(a) // 呵呵
console.log(b) // zyj
```

**栈内存中包括了变量的标识符和变量的值。**

![js_primitive_type_stack](/assets/img/js_primitive_type_stack.png)

## 引用类型

除了 6 种基本数据类型外，还要剩下的引用类型，即 `Object` 类型。细分的话，有：`Object 类型`、`Array 类型`、`Date 类型`、`RegExp 类型`、`Function 类型` 等。

引用类型的值是按引用访问的。

- 引用类型的值是可变的

```js
var obj = { name: 'zyj' } // 创建一个对象
obj.name = 'percy' // 改变 name 属性的值
obj.age = 21 // 添加 age 属性
obj.giveMeAll = function() {
  return this.name + ' : ' + this.age
} // 添加 giveMeAll 方法
obj.giveMeAll()
```

- 引用类型的比较是引用的比较

```js
var obj1 = {} // 新建一个空对象 obj1
var obj2 = {} // 新建一个空对象 obj2
console.log(obj1 == obj2) // false
console.log(obj1 === obj2) // false
```

- 引用类型的值是保存在堆内存（`Heap`）中的对象（`Object`）

与其他编程语言不同，`JavaScript` 不能直接操作对象的内存空间（堆内存）。

```js
var a = { name: 'percy' }
var b
b = a
a.name = 'zyj'
console.log(b.name) // zyj
b.age = 22
console.log(a.age) // 22
var c = {
  name: 'zyj',
  age: 22
}
```
