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

## 面向并发的内存模型

### Goroutine

在 `Go` 语言中启动一个 `Goroutine` 不仅和调用函数一样简单，而且 `Goroutine` 之间调度代价也很低。

### 互斥锁 & 原子操作

互斥锁和其他语言一样，`sync.Mutex` 的 `Lock()` 与 `Unlock()`。但是用互斥锁来保护一个数值型的共享资源，麻烦且效率低下。标准库的 `sync/atomic` 包对原子操作提供了丰富的支持。在 1.4 版本以前，`atomic`只支持几种基本数据类型的原子操作，之后加入了 `Value` (`interface{}`)类型解决了多种场景适配问题。

使用原子操作实现单例模式：

```go
type Once struct {
	m    Mutex
	done uint32
}

func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 1 {
		return
	}

	o.m.Lock()
	defer o.m.Unlock()

	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

```go
var (
	instance *singleton
	once     sync.Once
)

func Instance() *singleton {
	once.Do(func() {
		instance = &singleton{}
	})
	return instance
}
```

### 初始化顺序

![初始化顺序](/assets/img/golang_init_flow.png)

要注意的是，在 `main.main` 函数执行之前所有代码都运行在同一个 `Goroutine` 中，也是运行在程序的主系统线程中。如果某个 `init` 函数内部用 `go` 关键字启动了新的 `Goroutine` 的话，新的 `Goroutine` 和 `main.main` 函数是并发执行的。

因为**所有的 `init` 函数和 `main` 函数都是在主线程完成**，它们也是满足顺序一致性模型的。

### Channel

对于带缓冲的 `Channel`，对于 `Channel` 的第 K 个开始接收发生在第 K+C 个发送操作完成之前，其中 C 是 `Channel` 的缓存大小。举个例子：

```go
//make的第二个参数为缓冲大小，不填为0
var done = make(chan bool)
var msg string

func aGoroutine() {
	msg = "你好, 世界"
	done <- true
}

func main() {
	go aGoroutine()
	<-done
	println(msg)
}

//类似于消费者模型，先启动消费者监听，再开始生产消息，在Channel中表现为先<-done开始监听信号，再 done<-true 发送信号，发送完成后接收操作才紧跟着完成。
//因此对于无缓存的Channel(即make的第二个参数为空)，将<-done和done<-true互换位置也能实现同步顺序（但这非常危险）
```

## 并发

### 两种实现并发方式

- `Channel` 方式

```go
func main() {
	done := make(chan int, 1) // 带缓存的管道

	go func(){
		fmt.Println("你好, 世界")
		done <- 1
	}()

	<-done
}
```

- `sync.WaitGroup` 方式

这种方式不会一个个去接收返回值，适合多个异步处理操作，同时又不关心返回值的情况。

```go
func main() {
	var wg sync.WaitGroup

	// 开N个后台打印线程
	for i := 0; i < 10; i++ {
		wg.Add(1)

		go func() {
			fmt.Println("你好, 世界")
			wg.Done()
		}()
	}

	// 等待N个后台线程完成
	wg.Wait()
}
```

### 赢者为王

我们经常会遇到这样的场景：需要同不同工具去执行一向任务，取最新执行完的结果。假设我们想快速地搜索“golang”相关的主题，我们可能会同时打开 Bing、Google 或百度等多个检索引擎。当某个搜索最先返回结果后，就可以关闭其它搜索页面了。可以使用带缓存的通道来实现：

```go
func main() {
	ch := make(chan string, 32)

	go func() {
		ch <- searchByBing("golang")
	}()
	go func() {
		ch <- searchByGoogle("golang")
	}()
	go func() {
		ch <- searchByBaidu("golang")
	}()

	fmt.Println(<-ch)
}
```

### 并发的安全退出

`Go` 语言中不同 `Goroutine` 之间主要依靠管道进行通信和同步。要同时处理多个管道的发送或接收操作，我们需要使用 `select` 关键字（这个关键字和网络编程中的 `select` 函数的行为类似）。当 `select` 有多个分支时，会随机选择一个可用的管道分支，如果没有可用的管道分支则选择 `default` 分支，否则会一直保存阻塞状态。

```go
func worker(cannel chan bool) {
	for {
		select {
		default:
			fmt.Println("hello")
			// 正常工作
		case <-cannel:
			// 退出
		}
	}
}

func main() {
	cancel := make(chan bool)

	for i := 0; i < 10; i++ {
		go worker(cancel)
	}

	time.Sleep(time.Second)
	close(cancel)
}
//只要close cancel这个通道，那么所有goroutine都会收到一个0值，如果这里的close改为输入一个值（cancel<-true），那么只能停止其中一个goroutine
```

当然，也可以使用 `sync.WaitGroup` 改进上面的机制，以防部分清理工作（goroutine 结束时）还未完成：

```go
func worker(wg *sync.WaitGroup, cannel chan bool) {
	defer wg.Done()

	for {
		select {
		default:
			fmt.Println("hello")
		case <-cannel:
			return
		}
	}
}

func main() {
	cancel := make(chan bool)

	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go worker(&wg, cancel)
	}

	time.Sleep(time.Second)
	close(cancel)
	wg.Wait()
}
```

## 错误和异常

空指针不等于 `nil`，错误是一种接口类型，接口信息包含了原始类型和值，只有当类型和值都为空时才等于 `nil`，当然，类型为空的时候值一定为空，反之则不成立。看一个例子：

```go
func returnsError() error {
	var p *MyError = nil
	if bad() {
		p = ErrBad
	}
	return p // Will always return a non-nil error.
}
```

上述函数的返回值在正常情况下并不是 `nil` 而是一个 `MyError` 类型的空指针，也就是说 `returnsError() != nil`。

### 剖析异常

`panic` 和 `recover` 可认为是一对相反操作，`panic` 抛出异常，而 `recover` 捕获异常，可以从他们接口的定义看出：

```go
func panic(interface{})
func recover() interface{}
```

`recover` 函数的返回值即是最近一个 `panic` 的输入参数。当函数调用 `panic` 抛出异常，函数将停止执行后续的普通语句，但是之前注册的 `defer` 函数调用仍然保证会被正常执行，然后再返回到调用者。

> 注意！`recover` 函数不能被包装，必须在 `defer` 函数中直接调用 `recover` 才有效，例如:

```go
func main() {
	defer func() {
		// 无法捕获异常，必须有且只有一层调用才可捕获异常
		if r := MyRecover(); r != nil {
			fmt.Println(r)
		}
	}()
	panic(1)
}

func MyRecover() interface{} {
	log.Println("trace...")
	return recover()
}
```
