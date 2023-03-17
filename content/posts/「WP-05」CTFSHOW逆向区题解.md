---
title: 「CTFSHOW」Rev 随缘更新
date: 2021-06-26 10:55:29
tags:
- Rev
categories:
- TECHNOLOGY
---

## 卷王杯

只记录一下 BabyOC, 第二次遇到这种类型的, 运气好拿了个一血

### BabyOC

题目使用 Objective-C 编写. 在 ArchLinux 下想要运行附件需要 `sudo pacman -S gnustep-base` . 
由于没有相关开发经验, 只能简单查看 offical website 中给出的 oc 中 oop 的核心概念. 结合题目, 我个人理解的是使用一种类似 **通信** 的方式来实现函数调用等等. 同时函数调用时除了函数名, 貌似会生成一种 **对象签名**（也是我自己理解编出来的词汇, 不准确请见谅）来标识实际被调用者, 这一点也是后面用到的.
简单 IDA 里看一下:
![](https://s2.loli.net/2022/02/27/wip35KzbmWu7ayj.png)

创建了两个对象, 并在 input 方法里面进行初始化, 通过 xref 可以找到对象结构和调用函数:
![](https://s2.loli.net/2022/02/27/QaCByvigDw86157.png)

一种链式的调用过程:
![](https://s2.loli.net/2022/02/27/hQRL3AXamo4WJF8.png)

发现是分奇偶位进行异或. 同样分析 calc 函数, 这里注意需要调试一下, 因为 + 和 ^ 是先后调用的.

keygen 如下:

```python
data = [1270, 2767, 5549, 9672, 11938, 16093, 29864, 30379, 22184, 20690, 25002,
        65039, 65793, 97983, 100411, 67904, 88053, 28147, 18776, 71764, 127654,
        39994, 30276, 33151, 49377, 62682, 128398, 32406]

key = [0x4E1 * i for i in range(28)]

for i in range(1, 28):
    if i % 2 == 1:
        data[i] ^= data[i - 1]
    else:
        data[i] -= data[i - 1]

for i in range(26, -1, -1):
    if i % 2 == 0:
        data[i] ^= data[i + 1]
    else:
        data[i] -= data[i + 1]

for i in range(28):
    print(chr(data[i] ^ key[i]), end='')
```
