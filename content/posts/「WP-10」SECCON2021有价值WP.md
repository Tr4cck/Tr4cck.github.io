---
title: 「SECCONCTF2021」Rev | 汇编调试与马尔可夫图灵机
date: 2022-01-13 19:01:15
tags:
- Rev
categories:
- TECHNOLOGY
---

## pyast64++.rev

题目实现了一个简单的 py 汇编器。将可执行文件拖进 IDA 可以看到非常多成对的 push-pop 操作，并且简单调试一下发现函数调用栈是一种奇怪的结构，即：

```
--------
|length|
--------
|canary|
--------
|param1|
--------
|param2|
........
```

猜测出题人没有对编译出来的汇编代码做优化，函数实现、传参以及用于栈保护的 canary 也是一种比较原始的实现。大概有两种思路：

- 分析附件给出的 py，写汇编器的恢复
- 直接逆二进制文件，比较耗时但应该可做

比赛时选择了后者，因为觉得是一种比较好上手的思路，后面可以研究这个汇编器的实现，看看能否恢复出抽象语法树。

程序流程大致如下：

```
read flag -> check -> compare
```

比较套路化的过程，主要是调试的时候由于赋值是通过 push-pop 对，比较耗时耗力。然后由于栈结构的变化，使得 IDA 的 F5 基本是不能细看，推荐主要看汇编，不太明白的地方 F5 参考一下。代码恢复结果：

```python
S = []
T = []
NEW = [0] * 64
key = 'SECCON2021'
sec_to_plain = [0x61] * 40
for i in range(256):
    S.append(255 - i)
    T.append(ord(key[i % len(key)]))

j = 0
for i in range(256):
    j = (j + S[i] + T[i]) % 256
    (S[i], S[j]) = (S[j], S[i])

for cnt in range(10):
    # S
    for i in range(64):
        sec_to_plain[i] = S[sec_to_plain[i]]

    # P
    for i in range(8):
        for j in range(8):
            tmp = sec_to_plain[8*i + j]
            for k in range(8): # 二进制编码
                NEW[k + 8 * j] = tmp % 2
                tmp //= 2
        for j in range(64):
            res = (j * j * j )% 0x43 % 0x40 # 比特交换
            (NEW[res], NEW[j]) = (NEW[j], NEW[res])
        for j in range(8): # 二进制译码
            v78 = 0
            v74 = 1
            for k in range(8):
               v78 |= NEW[k + 8 * j] * v74
               v74 *= 2
            v70 = j + 8 * i
            sec_to_plain[v70] = v78

    for i in range(64):
        sec_to_plain[i] ^= ord(key[cnt])

enc = [0x4B, 0xCB, 0xBE, 0x7E, 0xB8, 0xA9, 0x1B, 0x4A, 0x23, 0x53, 0x71, 0x41, 0xCF, 0xC1, 0x1B, 0x89, 0x25, 0x62, 0x00, 0x44, 0xDB, 0x71, 0x15, 0xB4, 0xDF, 0x87, 0x05, 0x81, 0xBD, 0xC8, 0xF5, 0x64, 0x75, 0x3E, 0xC0, 0x65, 0xEF, 0x5C, 0xB6, 0x88, 0x9F, 0xEB, 0xA6, 0x5A, 0x4A, 0x85, 0x53, 0x4E, 0x06, 0xE1, 0x65, 0x67, 0x52, 0x4E, 0x90, 0xCD, 0x82, 0xEE, 0xAF, 0xF5, 0xAC, 0x3E, 0x9D, 0xB0]
```

exp：

```python
ans = []
S = [243, 106, 132, 232, 161, 31, 46, 140, 221, 19, 203, 158, 173, 183, 157, 76, 43, 237, 13, 209, 130, 81, 39, 235, 198, 122, 123, 77, 62, 63, 79, 146, 196, 215, 93, 145, 38, 187, 2, 14, 163, 195, 171, 242, 178, 167, 164, 236, 220, 212, 102, 3, 162, 192, 168, 230, 153, 252, 124, 229, 210, 159, 71, 45, 34, 6, 127, 84, 143, 113, 57, 96, 66, 100, 44, 204, 30, 87, 250, 185, 118, 131, 65, 47, 174, 54, 24, 98, 137, 213, 69, 52, 223, 110, 129, 255, 75, 53, 99, 36, 58, 197, 251, 175, 244, 51, 21, 217, 211, 169, 201, 141, 234, 83, 16, 49, 12, 92, 1, 166, 8, 206, 186, 126, 239, 240, 108, 29, 144, 194, 238, 105, 120, 224, 179, 32, 119, 228, 138, 231, 125, 26, 165, 133, 73, 40, 107, 249, 227, 248, 74, 176, 184, 20, 10, 116, 78, 80, 111, 60, 155, 104, 41, 68, 33, 142, 112, 59, 136, 149, 85, 128, 48, 177, 0, 150, 241, 94, 72, 160, 253, 139, 135, 154, 56, 233, 7, 11, 28, 202, 172, 27, 50, 82, 246, 193, 205, 22, 200, 247, 9, 208, 216, 115, 121, 189, 190, 134, 218, 152, 219, 5, 15, 86, 61, 23, 188, 35, 117, 88, 151, 4, 70, 207, 89, 180, 222, 214, 55, 245, 156, 91, 226, 254, 191, 181, 182, 148, 170, 225, 199, 97, 67, 101, 64, 103, 114, 25, 147, 18, 95, 42, 37, 17, 109, 90]
enc = [0x4B, 0xCB, 0xBE, 0x7E, 0xB8, 0xA9, 0x1B, 0x4A, 0x23, 0x53, 0x71, 0x41, 0xCF, 0xC1, 0x1B, 0x89, 0x25, 0x62, 0x00, 0x44, 0xDB, 0x71, 0x15, 0xB4, 0xDF, 0x87, 0x05, 0x81, 0xBD, 0xC8, 0xF5, 0x64, 0x75, 0x3E, 0xC0, 0x65, 0xEF, 0x5C, 0xB6, 0x88, 0x9F, 0xEB, 0xA6, 0x5A, 0x4A, 0x85, 0x53, 0x4E, 0x06, 0xE1, 0x65, 0x67, 0x52, 0x4E, 0x90, 0xCD, 0x82, 0xEE, 0xAF, 0xF5, 0xAC, 0x3E, 0x9D, 0xB0]
key = 'SECCON2021'
NEW_SUB = [0] * 64

for cnt in range(10):
    for i in range(64):
        enc[i] ^= ord(key[10 - cnt - 1])

    for i in range(8):
        for j in range(8):
            tmp = enc[j + i*8]
            for k in range(8):
                NEW_SUB[k + 8*j] = tmp % 2
                tmp //= 2

        for j in range(63, -1, -1):
            res = (j**3) % 0x43 % 0x40
            NEW_SUB[res], NEW_SUB[j] = NEW_SUB[j], NEW_SUB[res]

        for j in range(8):
            v78 = 0
            v74 = 1
            for k in range(8):
                v78 |= NEW_SUB[k + 8*j] * v74
                v74 *= 2
            enc[j + 8*i] = v78

    for i in range(64):
        enc[i] = S.index(enc[i])


print(bytes(enc))
```

## sed-programming
题目利用实现 sed 工具的核心算法 markov-algorithm 的图灵完备性，实现了一门编程语言。下面先来了解一下 markov-algorithm。
### markov-algorithm
是一个字符串重写系统，数学上被证明是图灵完备的，意味着可以基于它实现通用计算模型、数学表达式、编程语言实现等。它主要由两个部分组成：可应用的字母表 + 映射方法集合$\{D=f(L) | L \rarr D\}$，其中 L 和 D 均表示由字母表中符号组成的任意的字符串，我们假设 $\rarr$ 不存在于字母表中，否则需要重新找一个分割字符串。

举个 🌰 方便理解：

alphabet = {|, *, a, b, c}
$$
f = 
\begin{cases}
|b \rarr ba| \\
ab \rarr ba \\
b \rarr \\
*| \rarr b* \\
* \rarr c \\
|c \rarr c \\
ac \rarr c| \\
c \rarr .

\end{cases}
$$
将该算法应用字母表中的任意字符串 V 的过程是一个离散的基本步骤序列。我们假设 V′ 是在算法的前一步中得到的词（或者是原始词 V，如果当前的步骤是第一步）。如果替换公式中没有包含在 V′ 中的左手边，那么算法就终止了，其工作结果被认为是字符串 V′。否则，将选择左手边包含在 V′ 中的 **第一个** 替换公式。如果这个替换公式的形式是 L→D，那么在R L S形式的字符串 V′ 的所有可能的表示中，选择一个最短的 R，之后字符串R D S被认为是当前步骤的结果，需要在下一步骤中进一步处理。

比如上面的 🌰 ，|*|| 的结果是 || ，可以自己验证一下。

### sed

[info sed](https://www.gnu.org/software/sed/manual/sed.html)，sed 功能强大，支持正则匹配，这道题用了三个：

-   `s`ubstitute

-    `t`est. Jump if a substitution is done.
-   `p`rint. Print the result after every substitution.

我们可以先从后面单个字符的替换开始：

```
s/S/1IlIl11IIll11IlIl11IIll11IlIl11IlIl11IIll11IIll1/;tt
s/T/1IlIl11IIll11IlIl11IIll11IlIl11IIll11IlIl11IlIl1/;tt
s/i/1IlIl11IIll11IIll11IlIl11IIll11IlIl11IlIl11IIll1/;tt
s/7/1IlIl11IlIl11IIll11IIll11IlIl11IIll11IIll11IIll1/;tt
s/y/1IlIl11IIll11IIll11IIll11IIll11IlIl11IlIl11IIll1/;tt
s/b/1IlIl11IIll11IIll11IlIl11IlIl11IlIl11IIll11IlIl1/;tt
s/q/1IlIl11IIll11IIll11IIll11IlIl11IlIl11IlIl11IIll1/;tt
```

发现长度一致且为 8 的倍数，猜测是 ascii code。尝试定义映射集如下：

```python
R = [
  ("1IlI1" , "#"),
  ("1I1"   , "-"),
  ("1IIlI1", "<"),
  ("1l1"   , ">"),
  ("1IllI1", "@"),
  ("1III1" , "a"),
  ("1IIII1", "b"),
  ("1IIIl1", "f"),
  ("1II1"  , "t"),
  ("1Ill1" , "x"),
  ("1Illl1", "y"),
  ("1IlIl1", "0"),
  ("1IIll1", "1"),
]

for line in open('checker/checker0.sed'):
  line = line[:-1]

  remove = False
  for _, v in R:
    if line.startswith(f"s/{v}/"): # avoid conflict
      remove = True
  if remove:
    line = "#"+line

  for k, v in R:
    line = line.replace(k, v)

  print(line)

  if line==":t":
    print("p") # log
```

其实可以随便定，然后就可以读一下变换出来的源码，发现程序是基于 cursor 的一种变换：

-   生成 cursor，然后不停做类似 rol 的操作
-   每 3bit 分为一组，并替换成 count('1') % 2 然后重复 0b1101 次

用 python 先把比较数提取出来，思路是一个正则比较，每次都从 bin(n) 去找 bin(n+1)，那么第二个 t 后面的一位就是待比较的数，如下：

```python
import re

r = re.compile(r"^s/t([01]+)t([01])/t[01]+t/;tt$")

target = {}
for line in open("checker/checker_unobfuscated.sed"):
  line = line[:-1]
  m = r.match(line)
  if m:
    target[int(m.group(1), 2)] = m.group(2) # just ()

print(target)
print("target:", "".join(target[k] for k in range(len(target))))
```

然后 exp 由于某些位已知，思路类似：

```python
N = 13
target = "110101101111111011101100110001110111101011110100101101011011110011000101001001011111000101001011100000100101001111001111111011100100001001011100110010000011111101110110110100000111000010101101"
n = len(target)

flag = "SECCON{"+"X"*(n//8-8)+"}"
flag = "".join(bin(ord(f)+0x100).replace("0b1", "") for f in flag) # confirm character -> 8 bit binary

T = [list(flag)] # original flag with all 13 loops results
for i in range(N):
  t = ["0"]+T[i]+["0"]
  T += [[]]
  for j in range(n):
    T[i+1] += [str(t[j:j+3].count("1")%2)]
T[N] = list(target)

for i in range(N)[::-1]:
  for j in range(1, n-1):
    T[i][j+1] = str((T[i][j-1:j+1]+[T[i+1][j]]).count("1")%2)

flag = "".join(T[0])
flag = "".join(chr(int("".join(flag[i:i+8]), 2)) for i in range(0, n, 8))
print("flag:", flag)
```
