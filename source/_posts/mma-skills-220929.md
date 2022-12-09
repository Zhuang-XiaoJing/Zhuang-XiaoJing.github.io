---
title: Mathematica杂技
date: 2022-09-29 21:47:39
tags: [Wolfram]
categories: [Notes]
cover: https://m1.im5i.com/2022/09/30/UFqOVw.png
top_img: https://m1.im5i.com/2022/09/27/UFnPpo.jpg
---

## Array函数

构造向量最常用的 Table:

```mathematica
I[]:= Table[i^2 + i + 1, {i, 10}]
O[]:= {3, 7, 13, 21, 31, 43, 57, 73, 91, 111}
```

Table虽然好用，但引入了多余的变量: i，下面记录几种不需要多余变量就能生成数组的方法。

* 使用 Array + 匿名函数：

```mathematica
I[]:= Array[#^2 + # + 1 &, 10]
O[]:= {3, 7, 13, 21, 31, 43, 57, 73, 91, 111}
```

* 使用 匿名函数 + mapping :

  ```mathematica
  I[]:= #^2 + # + 1 & /@ Range[10]
  O[]:= {3, 7, 13, 21, 31, 43, 57, 73, 91, 111}
  ```

介绍 Array 之前，先回顾一下 mapping 的用法：

```mathematica
I[]:= f /@ Range[10]
O[]:= {f[1], f[2], f[3], f[4], f[5], f[6], f[7], f[8], f[9], f[10]}
```

注意是将 f 作用到/@后面的表达式中**第一层**的每一个元素。如果要作用到更深层，则需要使用 /@ 的Fullform: Map[*f*, *expr*, *levelspec*], 将 *f*  应用到 *levelspec* 指定的 *expr* 部分中，如下：

```mathematica
I[]:= Map[f, {{a, b}, {c, d, e}}, 2](*作用直到第2层*)
O[]:= {f[{f[a], f[b]}], f[{f[c], f[d], f[e]}]}
```

```mathematica
I[]:= Map[f, {{a, b}, {c, d, e}}, {2}](*作用到第2层*)
O[]:= {{f[a], f[b]}, {f[c], f[d], f[e]}}
```

还可以作用到负数层，更多细节参照帮助文档。

Array生成一维数组：

```mathematica
I[]:= Array[f, 10](*起始点默认从1开始，10代表从起始点往上10个数*)
O[]:= {f[1], f[2], f[3], f[4], f[5], f[6], f[7], f[8], f[9], f[10]}

I[]:= Array[# &, 10, 0](*设置起始点为0*)
O[]:= {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
```

Array自带遍历数组所有组合的属性，因此可以生成高维数组：

```mathematica
Array[Subscript[a, #1, #2] &, {3, 3}] // MatrixForm (*{3,3}代表数组的两个轴都是从1开始数三个数的List,Array就是要对这两组List的元素做全部组合，变成Array第一个参数的自变量*)
```

![UFZXnh.png](https://m1.im5i.com/2022/09/29/UFZXnh.png)

将上面的匿名函数换做 f 发现，是这样的自变量：

![UFZm5X.png](https://m1.im5i.com/2022/09/29/UFZm5X.png)

即f的自变量有多个，是一个Sequence，因此可以写为双 Slot 的形式（##：SlotSequence）

```mathematica
Array[Subscript[a, ##] &, {3, 3}]// MatrixForm 
```

以达到同样的效果。

SlotSequence 表示提供给一个纯函数的参数序列，##n表示从第n个参数开始提供，例：

```mathematica
I[]:= f[x, ##, y, ##] &[a, b, c, d]
O[]:= f[x, a, b, c, d, y, a, b, c, d]

I[]:= f[##2] &[a, b, c, d] (*从第二个参数开始*)
O[]:= f[b, c, d]
```

Array的二维数组生成也可以像一维一样定义起始点：

```mathematica
Array[f, {2, 3}, {0, 4}] // MatrixForm
```

![UFZHwf.png](https://m1.im5i.com/2022/09/29/UFZHwf.png)

前面的花括号分别存储两轴的维度，后面的花括号分别存储两轴的起始指标。

Array还可以“换头”，即：Array[*f*, *dims*, *origin*, *h*]，表示对数组的每一层使用头部 *h*, 而不是 List , 例: 

```mathematica
I[]:= Array[# &, 100, 1, Plus]
O[]:= 5050
```



记一个细节：

```mathematica
I[]:= f[Range[2]]
O[]:= f[{1,2}]

I[]:= f[Sequence @@ Range[2]]
O[]:= f[1,2]
```

Sequence表示将参数序列自动拼接到函数参数。

完全压平所有参数列表到函数中：

```mathematica
I[]:= f[{{a, b}, {c, d}, {a}}] /. List -> Sequence
O[]:= f[a, b, c, d, a]
```

