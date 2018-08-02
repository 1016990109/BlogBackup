---
title: 前端基础之JS数组
date: 2018-07-19 15:18:41
tags:
    - 前端
    - js
    - 数组
---

## 基本用法

1.  `push(val1,val2,...)`、`pop` 操作，增加和删除，后进先出。
2.  `unshift(val1,val2,...)`、`shift()` 对应在开头添加元素和删除元素。
3.  `splice(start,deleteCount,val1,val2,...)` 从开始位置删除一定数量的元素，并从这个位置插入新的元素。
4.  `reverse()` 反向。
5.  `sort([orderfunction])` 排序。
6.  `slice([start] [,end])` 返回子数组，拷贝后的，复制一个数组简单地 `slice()`。
7.  `join(seperator)` 返回以 `seperator` 作为间隔的字符串，默认为 `,`。
8.  `concat([arr1][,arr2]...)` 连接数组并返回新的数组，`concat()` 复制数组。
9.  其他：`indexOf(val)`,`includes(val)`,`toString`(同 `join()` 或 `join(",")`),`forEach()`,`every`,`findIndex`,`map`,`reduce`,`keys`,`values`。

<!-- more -->

## 高级

1.字符串反向：例：`123abc` => `cba321`:

```js
'123abc'
  .split('')
  .reverse()
  .join('')
```

2.拍平数组：

```js
const arr = [1, [2, [3], 4], 5]
function flatten(arr) {
  for (let i in arr) {
    if (Array.isArray(arr[i])) {
      arr.splice(i, 1, ...flatten(arr[i]))
    }
  }
  return arr
}
flatten(arr)
```

3.打印数组全排列：

```js
function allRange (arr, path, res) {
    if (!arr.length) {
        res.push(path)
        return
    }
    arr.forEach((v, idx) => {
        const t = arr.slice()
        const p = path.slice()
        t.splice(idx, 1)
        p.push(v)
        allRange(t, p, res)
    })
}

let a = [1, 2, 3, 4]
const b = []
allRange(a, [], b)
console.log(b)
```
