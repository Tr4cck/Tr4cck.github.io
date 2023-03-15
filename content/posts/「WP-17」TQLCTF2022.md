---
title: 「TQLCTF2022」Rev | Tales-of-Arrow 利用 asciicode 最高比特为 0 进行序列排除
date: 2022-03-02 16:18:42
tags: Rev
mathjax: true
toc: true
categories: TECHNOLOGY
---

get_lit 返回 i + 1 / - i - 1 然后根据后面的去还原那个 01 序列，给了输入长度是 17 个 ascii 字符。数据里面有干扰

8bits 一组的话，第一个肯定是 0，这样可以得到一组负的索引。然后把 output 里的数每 3 个分成一组，肯定有一个是对的。这样之后可以排除一部分数据，然后这时候有一些组内只有一个数据，那就肯定是对的。按这样排除出来

```python
from Crypto.Util.number import *

def sieve(data:list, must_true:set):
    for i in data:
        for j in i:
            if -j in must_true:
                i.remove(j)
    
    return data

def get_true(data:list, must_true:set):
    for i in data:
        if len(i) == 1:
            must_true.add(i[0])

    return must_true

with open('output.txt', 'rb') as fin:
    data = [int(i.decode()) for i in fin.read().splitlines()]
    
must_true = set()
for i in data:
    if i < 0 and -(i + 1) % 8 == 0:
        must_true.add(i)
    
new_data = []
for i in range(0, len(data) - 3, 3):
    perg = []
    for j in data[i : i + 3]:
        if -j in must_true:
            pass
        else:
            perg.append(j)

    new_data.append(perg)

while len(must_true) < 136:
    must_true = get_true(new_data, must_true)
    new_data = sieve(new_data, must_true)

bits = [0] * 136
for i in must_true:
    if i > 0:
        bits[i - 1] = 1
    else:
        bits[-i - 1] = 0

print(long_to_bytes(int(''.join(map(str, bits)), 2)))
```

其它题 official writeup 给的过于简略，不过好在放出了源码，能稍微研究一下出题思路
