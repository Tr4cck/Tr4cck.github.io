---
title: 「Lambda-Calculus」学习随笔
date: 2021-06-19 23:06:35
tags:
- CS
categories:
- TECHNOLOGY
---

## About Lambda Calculus

### Syntax

#### 标识符引用

举个例子

```lambda
lambda x.x+2
```

这个x实际上就被称为标识符

#### 柯里化

一个lambda表达式只接受一个参数，为了实现具有两个参数的函数，我们必须使用**柯里化**：

>  即写一个函数，它返回一个函数

#### 自由标识符 vs. 绑定标识符

要看你在哪个视角，或者说你站在lambda表达式中的哪一层来看这个问题。从全局来看，前者对应所谓全局变量，后者对应局部变量

### Grammar

#### Alpha translate

它是一个重命名操作，即：

> 变量的名称是不重要的（莫名想到离散数学

#### Beta rule

其实就是可以将值代入的意思，值得注意的是：

> 我们只有在不引起绑定标识符和自由标识符之间的任何冲突的情况下，才可以做Beta规约：如果标识符“`z`”在“`e`”中是自由的，那么我们就需要确保，Beta规约不会导致“`z`”变成绑定的。如果在“`B`”中绑定的变量和“`e`”中的自由变量产生命名冲突，我们就需要用Alpha转换来更改标识符名称，使之不同。

**特别地：参数大部分都是自由的**

#### 一个例子

```lambda
lambda x y.x y
```

这个实际上是一个简写，它等价于下面：

```lambda
lambda x.(lambda y.x)
```

其中x是一个lambda表达式，就是说接受两个参数x、y，返回将y带入x后的结果

### Number and Calculation

目前我们有的只有函数，而没有定义数字和算术运算的概念，因此我们需要用函数来定义数字，是的你没有看错。我们下面就要介绍**丘奇数**。

#### Church

所有丘奇数都是带有两个参数的函数

- 0是`lambda s z.z`
- 1是`lambda s z.s z`
- 2是`lambda s z.s (s z)`
- 对于任何数“`n`”，它的丘奇数是将其第一个参数应用到第二个参数上“`n`”次的函数。

> 一个很好的理解办法是将“`z`”作为是对于零值的命名，而“`s`”作为后继函数的名称。因此，0是一个仅返回“0”值的函数；1是将后继函数运用到0上一次的函数；2则是将后继函数应用到零的后继上的函数，以此类推。

利用丘奇数，我们可以完全利用lambda表达式定义`x+y`，下面先给出结果，然后进行详细解释：

```lambda
let add = lambda s z x y.x s (y s z)
```

按上面说的，我们先进行柯里化：

```lambda
let add = lambda x y.(lambda s z.(x s (y s z)))
```

从内到外分析，即为了将x、y相加，先`(y s z)`。这个的意思是，y是一个函数，将s，z应用到y上，意义上得到y的值；x也是一个函数，把s和内层应用到x上，得到类似x+y的值。

> 为了将`x`和`y`相加，先用参数“`s`”和“`z`”创建（正则化的）丘奇数“`y`”。然后应用`x`到丘奇数`y`上，这时候使用由“`s`”和“`z`”定义的丘奇数`y`。也就是说，我们得到的结果是一个函数，这个函数把自己加到另一个数字上。（要计算`x + y`，先计算 `y` 是 `z` 的几号后继，然后计算`x` 是 `y`的几号后继。）

**然后我们可以举一个例子：**

2 + 3：

```lambda
step1：translate
	add (lambda s z.s (s z)) (lambda s z.s (s (s z)))
step2: alpha
	add (lambda s2 z2 . s2 (s2 z2)) (lambda s3 z3 . s3 (s3 (s3 z3)))
step3: replace
	lambda x y.(lambda s z.(x s y s z))) (lambda s2 z2 . s2 (s2 z2)) (lambda s3 z3 . s3 (s3 (s3 z3)))
step4: beta to add
	lambda s z.(lambda s2 z2.s2 (s2 z2)) s (lambda s3 z3 . s3 (s3 (s3 z3)) s z)
step5: beta to 3
	lambda s z.(lambda s2 z2.s2 (s2 z2)) s (s (s (s z)))
step6: beta to 2
	lambda s z.s (s (s (s (s z))))
```

**DONE！**

#### Branch and loop

> 众所周知，其实我们只需要数据、分支跳转这两类，就能够表达任意计算了，其实编程语言基本都是从这三类指令开始向外扩展的
