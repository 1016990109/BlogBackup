---
title: kryo vs avro vs protobuf vs thrift vs jce
date: 2018-06-20 16:16:03
tags:
    - 序列化
    - 反序列化
    - kryo
    - avro
    - protobuf
    - jce
categories:
    - 学习
---

我们知道结构化数据存储方式多种多样，有 `json`、`xml`、`kryo`等等，而 `json` 与 `xml` 虽然可读性较强，但是需要的额外空间太多，当数据量过大的时候很浪费性能，所以就需要压缩率更高的编码方式，这里对比比较流行的几种存储（编码）方式：`kryo`、`avro`、`protobuf`、`thrift`、`jce`：

<!-- more -->

## protobuf

### 结构化数据：

需要预定义结构体（`.proto` 文件），消息经过序列化后会成为一个二进制数据流，该流中的数据为一系列的 `Key-Value` 对，定义好结构体的优势就是不用多余的数据来分隔不同的键值对。`Key` 用来标识具体的 `field`，在解包的时候，`Protocol Buffer` 根据 `Key` 就可以知道相应的 `Value` 应该对应于消息中的哪一个 `field`。

`Key` 定义：(field_number << 3) | wire_type

`field_number` 代表在 `.proto` 中定义的编号，1~15 用一个字节，16~2047 用两个字节，结合公式 （field_number << 3）| wire_type ，如果 `filed_number` 大于等于 16，两个字节共 16 位，去掉移位的 3 位，去掉两个字节中第一个比特位(`msb`)，总共 16 个比特位只有 16-5=11 个比特位用来表示 `Key`，所以 `Key` 的 `filed_number` 要小于 2^11== 2048。

更大的以此类推，主要就是看一个字节的 `msb` 是否为 1，最大可以为 2^29 - 1。

`wire_type` 表示该数据的类型，有 `Vaint`、`64-bit`、`Length-delimi`、`Start group`、`End group`、`32-bit`共 6 中类型，具体可查看[官方文档](https://developers.google.com/protocol-buffers/docs/encoding#order)

### Varint

了解 `protobuf` 首先就要了解 `Varint`，是它的一大核心，变长整数存储。长整数存储多的位数，短整数存储少的位数，来减少空间的浪费。除了最后一字节外每字节第一位都是 `most significant big`(**msb**)，表示是否后面是否还有字节表示该整数。例如：

`0000 0001` 就表示 1

`1010 1100 0000 0010` 第一个字节第一位为 1 表示后面还有数据，直到字节第一位为 0（这里就是第二个字节 `0000 0010`），将字节顺序逆向，变为 `0000 0010 010 1100`：`100101100`(300)

### ZigZag

而有符号整数则使用 `ZigZag` 编码方式，用无符号的整数同时代表正负两种数：

| Signed Original | Encoded As |
| --------------- | ---------- |
| 0               | 0          |
| -1              | 1          |
| 1               | 2          |
| -2              | 3          |
| 2147483647      | 4294967294 |
| -2147483648     | 4294967295 |

这样就大大减少了占用的位数，算法使用：

`(n << 1) ^ (n >> 31)` sint32
`(n << 1) ^ (n >> 63)` sint64

## float double

不压缩，多少位就多少位存储。

### string

`string` 则是在 `Value` 中多加了一个或多个字节表示长度（`msb`标识），剩下的内容才是真正的值。

### repeated

这里介绍压缩率更高的 `Packed Repeated Fields`。

在 `2.1.0` 版本以后，`protocol buffers` 引入了该种类型，其与 `repeated` 字段一样，只是在末尾声明了 `[packed=true]`。类似 `repeated` 字段却又不同。在 `proto3` 中 `Repeated` 字段默认就是以这种方式处理。

例如有如下 `message` 类型：

```proto
message Test4 {  
  repeated int32 d = 4 [packed=true];
}
```

构造一个 `Test4` 字段，并且设置 `repeated` 字段 d 3 个值：3，270 和 86942，编码后：

```
22 // tag 0010 0010(field number 010 0 = 4, wire type 010 = 2)

06 // payload size (设置的length = 6 bytes)

03 // first element (varint 3)

8E 02 // second element (varint 270)

9E A7 05 // third element (varint 86942)  
```

形成了 Tag - Length - Value - Value - Value …… 对，增加了压缩率。

## jce

### 结构

也需要定义结构体，头部+内容，与 `protobuf` 类似，`tag + type`(4+4) 形式，`tag` 类似 `filed_number` 也是超过 15 后继续下一字节，但是由于不是通过第一位标识后面是否有数据，故而之后后面再多一个字节，最大为 255。

```
tag(4) type(4) [tag(8)]
```

### int

直接定义 4 种，而不是变长，int8,int16,int32,int64

### string

两种，string1 和 string4，分别表示 8 位代表字符串长度，4\*8=32 位表示字符串长度，所以字符串长度最大 2^32 - 1。

### map

存放时分开存放，先存 size（大小），再根据大小存对应个数的 `key-value`，`key` 的 `tag` 为 0，`value` 的 `tag` 为 1。

### 结构体

用不同的 `tag` 标识开头和结尾即可。`protobuf` 不需要，是根据定义的 `.proto` 来自动解析结构体的，类似于 `map`。

## avro

`avro` 支持两种编码，一种 `json` 一种 `binary`，`json` 一般用于 `Web` 应用等需要易读性高的场景，大多数情况下都是二进制编码的。

同其他高效序列化-反序列化库一样，也是需要定义一个数据结构，只不过是用 `json` 定义的，内容如下：

- type: 类型（基本的和复杂的）
- name: 字段名称
- 其他的一些属性

详情查看[官方文档](https://avro.apache.org/docs/1.8.2/spec.html)

举个例子：

```json
{
  "type": "record",
  "name": "MyStruct",
  "fields": [
    {
      "type": "int",
      "name": "id"
    },
    {
      "type": "string",
      "name": "description"
    }
  ]
}
```

上面的 `json` 等价于定义了一个类似于 `protobuf` 这样的结构体：

```proto
Message MyStruct {
  uint32 id = 1,
  string description =2
}
```

### int, long

使用 `varlength zigzag`，同时使用变长和 `zigzag`，类似于 `protobuf`。

### string, array, map

也是先声明长度，接着才是真正的内容。

## thrift

### int, long

先进行 `zigzag`，再 `var int`。

### 其他

与 `avro`、`protobuf`大同小异，具体可查看[官方文档](https://github.com/apache/thrift/blob/master/doc/specs/thrift-compact-protocol.md)

`map` 每个键值对都有 `key-type` 和 `value-type`。

## kryo

`kryo` 是一种快速高效的 `Java` 对象图（`Object graph`）序列化框架，要实现跨语言是比较困难的。

### int, long

也使用 `varint` 变长存储，减少空间。

### class

不像 `Java` 自带的序列化工具携带了很多信息，`kryo` 只携带了 **标识+类名+字段** 三部分，更加简单，而且还不用自定义结构体，省去了 `.proto` 类似文件编写的麻烦。

## 总结

可以发现高效的序列化库都对 `int` 和 `long` 这类整型做了变长的压缩（`jce` 定义不同长度整型也是变长的一种），而对结构体的存储基本上也是存储 **键值对** ，每一个键值对再标记值的类型，这样能大大压缩数据的空间，提高传输的效率。

需要预先定义数据格式的有：`protobuf`、`jce`、`avro`、`thrift`。

不需要预先定义数据格式的有：`kryo`。

跨语言：`protobuf`、`jce`、`avro`、`thrift`。

性能上`protobuf`、`jce`、`avro`、`thrift` 几者差别不大，`kryo` 序列化和反序列化较慢（但是不用预先定义数据格式，可省去麻烦），但是 `kryo` 的编码后的大小略微小一点。

### Google protobuf

#### 优点

- 二进制消息，性能好/效率高（空间和时间效率都很不错）
- `proto` 文件生成目标代码，简单易用
- 序列化反序列化直接对应程序中的数据类，不需要解析后在进行映射(`XML`,`JSON`都是这种方式)
- 支持向前兼容（新加字段采用默认值）和向后兼容（忽略新加字段），简化升级
- 支持多种语言（可以把 `proto` 文件看做 `IDL` 文件）
- `Netty` 等一些框架集成

#### 缺点

- 官方只支持 `C++`,`JAVA` 和 `Python` 语言绑定
- 二进制可读性差（貌似提供了 `Text_Fromat` 功能）
- 二进制不具有自描述特性
- 默认不具备动态特性（可以通过动态定义生成消息类型或者动态编译支持）
- 只涉及序列化和反序列化技术，不涉及 `RPC` 功能（类似 `XML` 或者 `JSON` 的解析器）

### Apache Thrift

#### 优点

- 支持非常多的语言绑定
- thrift` 文件生成目标代码，简单易用
- 消息定义文件支持注释
- 数据结构与传输表现的分离，支持多种消息格式
- 包含完整的客户端/服务端堆栈，可快速实现 `RPC`
- 支持同步和异步通信

#### 缺点

- 和 `protobuf` 一样不支持动态特性

### Apache Avro

#### 优点

- 二进制消息，性能好/效率高
- 使用 `JSON` 描述模式
- 模式和数据统一存储，消息自描述，不需要生成 `stub` 代码（支持生成 `IDL`）
- `RPC` 调用在握手阶段交换模式定义
- 包含完整的客户端/服务端堆栈，可快速实现 `RPC`
- 支持同步和异步通信
- 支持动态消息
- 模式定义允许定义数据的排序（序列化时会遵循这个顺序）
- 提供了基于 `Jetty` 内核的服务基于 `Netty` 的服务

### 缺点

- 只支持 `Avro` 自己的序列化格式
- 语言绑定不如 `Thrift` 丰富
