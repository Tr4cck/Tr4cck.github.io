---
title: 「StreamCipher」线性同余生成器
date: 2021-05-20 22:51:29
tags:
- Cryptography
categories:
- TECHNOLOGY
---


<!-- more -->

## Method One

程序的大概意思就是一个猜数游戏，如果连续猜中若干次，就算会拿到 flag，背后的生成相应数的核心代码如下

```python
class SecurePrng(object):
    def __init__(self):
        # generate seed with 64 bits of entropy
        self.p = 4646704883L
        self.x = random.randint(0, self.p)
        self.y = random.randint(0, self.p)

    def next(self):
        self.x = (2 * self.x + 3) % self.p
        self.y = (3 * self.y + 9) % self.p
        return (self.x ^ self.y)
```

显然, 我们猜出前两轮还是比较容易的, 毕竟概率也有 0.25. 这里当我们猜出前两轮后, 使用 Z3 来求解出初始的 x 和 y, 那么我们就可以顺利的猜出剩下的值了. 不妨进行一波数学推导: 

$$
p = 4646704883 \\
x_0 = random.randint(0, p) \\
y_0 = random.randint(0, p) \\
x_1 = (2x_0 + 3) \% p \\
y_1 = (3y_0 + 9) \% p \\
sol_1 = x_1 \bigoplus y_1 \\
x_2 = (2x_1 + 3) \% p = (2((2x_0 + 3) \% p) + 3) \% p \\
y_2 = (3y_1 + 9) \% p = (3((3x_0 + 9) \% p) + 9) \% p \\
sol_2 = x_2 \bigoplus y_2
$$

思路大概就是, 如果能猜到 $sol_1$ 和 $sol_2$, 就能直接得到 $x_0$ 和 $y_0$, 如此以来, 就能算得所有的值:

```python
from z3 import *
import sys

s1cor = int(sys.argv[1]) # 运行程序的时候直接附上猜得的两个值
s2cor = int(sys.argv[2])
dimVector =35 # TODO: 事实上p只有33bits，不知道这里为什么要设成35bits
x = BitVec('x', dimVector)
y = BitVec('y', dimVector)
p = BitVec('p',dimVector)
s1 = BitVec('s1',dimVector)
s2 = BitVec('s2',dimVector)
s = Solver()
s.add(p == 4646704883L)
s.add(s1 == s1cor)
s.add(s2== s2cor)
s.add( ( ( ( 2 * x + 3 ) % p ) ^ ( ( 3 * y + 9 ) % p ) )==s1)
s.add(( ( ( 2 * ( ( 2 * x + 3 ) % p )  + 3 ) % p ) ^ ( ( 3 * ( (  3 * y + 9 ) % p) + 9 ) % p ) )==s2)
while s.check() == sat:
    class SecurePrng(object):
        def __init__(self,x,y):
            self.p = 4646704883L
            self.x = x
            self.y = y
        def next(self):
            self.x = (2 * self.x + 3) % self.p
            self.y = (3 * self.y + 9) % self.p
            return (self.x ^ self.y)    
        def getX(self):
            return self.x
        def getY(self):
            return self.y
    m = s.model() # 解向量
    pMy = 4646704883L
    myObj = SecurePrng(int(str(m[x]))%pMy,int(str(m[y]))%pMy) # 直接将解传进去，得到下一个结果。事实上这一步是验证解，因为可能有多组解
    mySol1 = myObj.next()
    mySol2 = myObj.next()
    if mySol1 == s1cor and mySol2 == s2cor and int(str(m[x]))<= pMy and int(str(m[y])) <= pMy :
        print "x = " + str(m[x]) + " ; y = " + str(m[y]) 
    s.add(Or(x != s.model()[x], y != s.model()[y]))
```

下面可以重写PRNG，达到利用的目的:

```python
class SecurePrng(object):
    def __init__(self):
        self.i = 0
        self.p = 4646704883L
        self.x = 3714993585 % self.p
        self.y = 2248563082 % self.p

    def next(self):
        print self.i
        self.i += 1
        self.x = (2 * self.x + 3) % self.p
        self.y = (3 * self.y + 9) % self.p
        return (self.x ^ self.y)

    def getX(self):
        return self.x

    def getY(self):
        return self.y
```

## Method Two

上面的方法是很好理解的，但CTFWiki上提供了另外一种方法，目前看来是一种更加值得记录下来，好好理解的方法

下面是原文摘录

> 这里我们考虑另外一种方法，**依次从低比特位枚举到高比特位获取 x 的值**，之所以能够这样做，是依赖于这样的观察
> 
> - a + b = c，c 的第 i 比特位的值只受 a 和 b 该比特位以及更低比特位的影响。**因为第 i 比特位进行运算时，只有可能收到低比特位的进位数值。**
> - a - b = c，c 的第 i 比特位的值只受 a 和 b 该比特位以及更低比特位的影响。**因为第 i 比特位进行运算时，只有可能向低比特位的借位。**
> - a * b = c，c 的第 i 比特位的值只受 a 和 b 该比特位以及更低比特位的影响。因为这可以视作多次加法。
> - a % b = c，c 的第 i 比特位的值只受 a 和 b 该比特位以及更低比特位的影响。因为这可视为多次进行减法。
> - a ^ b = c，c 的第 i 比特位的值只受 a 和 b 该比特位的影响。这一点是显而易见的。
> 
> **注：个人感觉这个技巧非常有用。**
> 
> 此外，我们不难得知 p 的比特位为 33 比特位。具体利用思路如下
> 
> 1. 首先获取两次猜到的值，这个概率有 0.25。
> 2. 依次从低比特位到高比特位依次枚举**第一次迭代后的 x 的相应比特位**。
> 3. 根据自己枚举的值分别计算出第二次的值，只有当对应比特位正确，可以将其加入候选正确值。需要注意的是，这里由于取模，所以我们需要枚举到底减了多少次。
> 4. 此外，在最终判断时，仍然需要确保对应的值满足一定要求，因为之前对减了多少次进行了枚举。

**注：我们主要关注利用思路的第二、三、四三步**

具体利用代码如下

```python
import os
import random
from itertools import product


class SecurePrng(object):
    def __init__(self, x=-1, y=-1): # 题外话，默认值参数真是一个美妙的东西，利用好了可以用来做cache、condition啥的
        # generate seed with 64 bits of entropy
        self.p = 4646704883L  # 33bit
        if x == -1:
            self.x = random.randint(0, self.p)
        else:
            self.x = x
        if y == -1:
            self.y = random.randint(0, self.p)
        else:
            self.y = y

    def next(self):
        self.x = (2 * self.x + 3) % self.p
        self.y = (3 * self.y + 9) % self.p
        return (self.x ^ self.y)


def getbiti(num, idx): # idx表达一个类似长度的量
    return bin(num)[-idx - 1:]


def main():
    sp = SecurePrng()
    targetx = sp.x
    targety = sp.y
    print "we would like to get x ", targetx
    print "we would like to get y ", targety

    # suppose we have already guess two number
    guess1 = sp.next()
    guess2 = sp.next()

    p = 4646704883

    # newx = tmpx*2+3-kx*p
    for kx, ky in product(range(3), range(4)): # 双重循环
        candidate = [[0]]
        # only 33 bit
        for i in range(33): # 第三层循环
            #print 'idx ', i
            new_candidate = []
            for old, bit in product(candidate, range(2)): # 最深双层循环
                #print old, bit
                oldx = old[0]
                #oldy = old[1]
                tmpx = oldx | ((bit & 1) << i) # TODO: 这个遍历方法还需要理解...
                #tmpy = oldy | ((bit / 2) << i)
                tmpy = tmpx ^ guess1 # seed_x与第一次猜测的结果异或，得到的就是seed_y
                newx = tmpx * 2 + 3 - kx * p + (1 << 40) # TODO: 后面判断的时候也加上了这个条件，这不等价于啥也没干？暂时不太懂
                newy = tmpy * 3 + 9 - ky * p + (1 << 40)
                tmp1 = newx ^ newy
                #print "tmpx:    ", bin(tmpx)
                #print "targetx: ", bin(targetx)
                #print "calculate:     ", bin(tmp1 + (1 << 40))
                #print "target guess2: ", bin(guess1 + (1 << 40))
                if getbiti(guess2 + (1 << 40), i) == getbiti( # 验证条件，并加入候选解
                        tmp1 + (1 << 40), i):
                    if [tmpx] not in new_candidate:
                        #print "got one"
                        #print bin(tmpx)
                        #print bin(targetx)
                        #print bin(tmpy)
                        new_candidate.append([tmpx])
            candidate = new_candidate
            #print len(candidate)
            #print candidate
        print "candidate x for kx: ", kx, " ky ", ky
        for item in candidate: # 依次验证每个候选解是否由线性同余生成器生成
            tmpx = candidate[0][0]
            tmpy = tmpx ^ guess1
            if tmpx >= p or tmpx >= p:
                continue
            mysp = SecurePrng(tmpx, tmpy)
            tmp1 = mysp.next()
            if tmp1 != guess2:
                continue
            print tmpx, tmpy
            print(targetx * 2 + 3) % p, (targety * 3 + 9) % p


if __name__ == "__main__":
    main()
```
