---
title: 5月28日学习笔记
date: 2020-03-24 17:37:00
tags:
  - GOLANG
categories:
  - 学习
---

## 函数、方法、接口

### 闭包

`GOLANG` 类似于 `Node.js` 可使用闭包，使得变量只在作用域有效。举个典型的例子：

```golang
func main() {
	for i := 0; i < 3; i++ {
		defer func(){ println(i) } ()
	}
}
// Output:
// 3
// 3
// 3
```

上述代码输出会是连续的三个 `3`，因为 `defer` 匿名函数中引用的是在 `main` 函数中的局部变量 `i`，而 `i` 在 `for` 循环结束后就已经变成 `3` 了，这和 `Node.js` 是类似的：

```js
function test() {
  //注意这里不能用 es6 的 let，因为 let 会使得 i 变为局部变量，在每一次循环中都会有独立的作用域（类似 GOLANG 中的 i:=i）
  for (var i = 0; i < 3; i++) {
    setTimeout(() => {
      console.log(i)
    }, 0)
  }
}
// Output:
// 3
// 3
// 3
```

解决上述问题的方法与 `Node.js` 类似，将 `i` 声明为局部变量，或者使用立即执行函数传参：

```go
//局部变量
func main() {
	for i := 0; i < 3; i++ {
    i := i
		defer func(){ println(i) } ()
	}
}
```

```go
// 立即执行函数
func main() {
	for i := 0; i < 3; i++ {
		defer func(i int){ println(i) } (i)
	}
}
```

### 切片的值传递

除非显示或隐式传入指针参数，否则函数都是通过值传递的。但是直接传入切片却看起来像是直接传的引用，这是怎么回事？这是因为切片在底层实际上是实现为一个 `struct`:

```go
// 整型例子
type IntSliceHeader struct {
	Data []int
	Len  int
	Cap  int
}
```

而实际上操作切片是操作这个 `Header` 的 `Data` 数据，这是指向数据的指针，所以看起来像直接修改了切片值。切片中的底层数组部分是通过隐式指针传递(指针本身依然是传值的，但是指针指向的却是同一份的数据)，所以被调用函数是可以通过指针修改掉调用参数切片中的数据。除了数据之外，切片结构还包含了切片长度和切片容量信息，这 2 个信息也是传值的。如果被调用函数中修改了 `Len` 或 `Cap` 信息的话，就无法反映到调用参数的切片中，这时候我们一般会通过返回修改后的切片来更新之前的切片。_这也是为何内置的 `append` 必须要返回一个切片的原因_。

## 方法

方法是绑定在某个类型上的特殊函数，函数与方法区别：

```go
//函数
func test(i int) {
  println(i)
}
//方法
type A struct {

}
func (a *A) test(i int) {
  println(i)
}
```

### “继承”(匿名组合)

`Go` 语言不支持传统面向对象中的继承特性，而是以自己特有的组合方式支持了方法的继承。`Go` 语言中，通过在结构体内置匿名的成员来实现继承：

```go
import "image/color"

type Point struct{ X, Y float64 }

type ColoredPoint struct {
	Point
	Color color.RGBA
}
```

上面代码可以简单看作为 `ColoredPoint` 继承了 `Point` 类型，可以使用 `Point` 类型的成员变量及方法，例如:

```go
coloredPoint := ColoredPoint{Point: Point{X: 1.0, Y: 2.0}}
//其实在编译期间换成了 coloredPoint.Point.X += 1
coloredPoint.X += 1
```

在 `Go` 语言中所实现的继承其实是在编译期间完成的，并非像传统面向对象语言（如 `Java`）是在运行时动态绑定的。
