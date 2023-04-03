---
title: 「Writeup」零碎 WP 整理
date: 2021-12-12 16:06:41
tags:
- Rev
categories:
- TECHNOLOGY
---


<!-- more -->

## HITCTF 2021

### BabyRNG

```python
from Crypto.Util.number import *

matrix = [ 0x737A074B2A5FFC4F, 0x321F3772B75371F0, 0xE4EE355BFC5A53DB, 0xA77F3B3A1705B5EB,
           0x862767F503168BF7, 0xA6BE8C5F94612BE0, 0x5CF08CFB54522960, 0x26E77E8A70492CF6,
           0x72D75624BC03E433, 0x8CA020BCAD669BF7, 0xCD2CC3F1998A67BB, 0x885FC05F86ABFB0,
           0x9599CC320D3FE57C, 0xCC9458420B27A204, 0x64CD34844C4880DB, 0x8C40FEB2DB68D66C]

result = [0x473D522ACD6D711D, 0x2749BD99CAA1E332, 0xDBD902B6C82D673D,
          0x203C7FADB24D4391, 0x92A2DAF7C5C4B7F0, 0x86A68A7AD1F1F475]

local = lambda a, b : matrix[b] ^ b ^ a ^ 0xdeadbeef

def genSecret(front, tail):
    subKey = []
    subKey.append(front)
    subKey.append(tail)
    for i in range(16):
        subKey.append(local(subKey[i + 1], i) ^ subKey[i])

    return subKey[-2], subKey[-1]

def deSecret(front, tail):
    subKey = []
    subKey.append(tail)
    subKey.append(front)
    for i in range(15, -1, -1):
        subKey.append(local(subKey[15 - i + 1], i) ^ subKey[15 - i])

    return subKey[-2:][::-1]

def encrypt(input):
    enc = []
    key = [0xDEADBEEFDEADBEEF, 0xBAADF00DBAADF00D]
    for i in range(0, 6, 2):
        temp1, temp2 = genSecret(input[i] ^ key[i], input[i + 1] ^ key[i + 1])
        enc.append(temp1)
        enc.append(temp2)
    key.append(temp1, temp2)
    return enc

def decrypt():
    msg = []
    key = [0xDEADBEEFDEADBEEF, 0xBAADF00DBAADF00D]
    for i in range(0, 6, 2):
        temp1, temp2 = deSecret(result[i], result[i + 1])
        msg.append(temp1 ^ key[0])
        msg.append(temp2 ^ key[1])
        key = result[i], result[i + 1]

    return msg

if __name__ == '__main__':
    plain = decrypt()
    for i in range(len(plain)):
        try:
            print(long_to_bytes(plain[i])[::-1].decode(), end='')
        except:
            continue
```

## NUAACTF 2021

### IDA Start

拖进 IDA 有明文 flag

### Warm up

init 函数指针中存着初始化 s1 的过程, 照着跑一遍得到 s1, 然后异或密文:

```python
s1 = [ord(ch) for ch in list('qasxcytgsasxcvrefghnrfghnjedfgbhn')]
s1.append(0)
print(s1)
s1_proced = [s1[i] ^ (2*i + 65) for i in range(34)]
s2 = [0x56, 0x4E, 0x57, 0x58, 0x51, 0x51, 0x09, 0x46, 0x17, 0x46, 0x54, 0x5A, 0x59, 0x59, 0x1F, 0x48, 0x32, 0x5B, 0x6B, 0x7C, 0x75, 0x6E, 0x7E, 0x6E, 0x2F, 0x77, 0x4F, 0x7A, 0x71, 0x43, 0x2B, 0x26, 0x89, 0xFE]
print(bytes([x ^ y for x, y in zip(s1_proced, s2)]))
```

### elf

修 4 个字节. 其中 3 个为 elf 文件的 `magic number`, 还有一个是第 17 个字节——因为该文件有 PHT 表, 所以肯定不是可重定位文件, 改成可执行文件. 然后拖进 IDA, 发现算了一下 0 的 MD5, 取成一个 8*16 的 byte 数组然后异或.

## MTCTF 2021

### Random

random-key 生成过程中使用触发除 0 异常使控制流改变, 可以静态分析出大概的触发链是啥. 好在出题人没在里面加反调试, 于是可以在 IDA 调试过程中手动改变 EIP 的值 (未曾设想的道路), 使程序往预期的方向走, 然后就把 key 调出来了, 然后就没了.
