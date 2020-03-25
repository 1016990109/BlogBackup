---
title: TensorFlow学习笔记（一）
date: 2019-01-29 17:44:41
tags:
  - tensorflow
---
## TensorFlow开发基本步骤
以 y=2x 为例

1. 定义 `TensorFlow` 的输入节点。这里为`x`。分为直接定义、占位符和字典定义：
   直接定义（很少用）：将定义好的 `Python` 变量直接使用。
   ```python
   train_X = np.linspace(-1, 1, 100)
   train_Y = 2 * train_X + np.random.randn(*train_X.shape) * 0.3
   ```
   占位符定义(使用 `tf.placeholder`，一般情况)：
   ```python
   X = tf.placeholder("float")
   Y = tf.placeholder("float")
   ```
   字典定义（适用于参数比较多，相当于 `js` 中的 `Object`）：
   ```python
   inputdict = {
       'x': tf.placeholder("float"),
       'y': tf.placeholder("float")
   }
   ```
2. 定义“学习参数”的变量，这里为 `w` 和 `b`，分别为一维的数字。分为直接定义和字典定义，同输入节点。
3. 定义“运算”。定义正向传播模型，这里为 `z = x * w + b`；定义损失函数，计算输出值与目标值的误差，配合反向传播修正参数用的。
4. 优化函数，优化目标。
5. 初始化所有变量。
6. 迭代更新参数到最优解，建立 `session`。
7. 测试模型。
8. 使用模型。

## TensorFlow 编程基础

### Session

多数情况使用 `with`。

```python
import tensorflow as tf

a = tf.constant(3)
b = tf.constant(4)

with tf.Session() as session:
    print(session.run(a * b))
```

### 注入机制

使用占位符定义时，可以使用 `feed` 填充数据，只在调用方法内有效。使用方法如下：

```python
add = tf.add(a, b)
session.run(add, feed_dict={a: 3, b: 4})
```


