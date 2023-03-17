---
title: 「SUSCTF2022」Rev 部分题解
date: 2022-02-27 20:46:31
tags:
- Rev
categories:
- TECHNOLOGY
---

## DigitalCircuits

走流程得到源代码:

```python
import time

def f1(a, b): # and
    if a == '1' and b == '1':
        return '1'
    return '0'


def f2(a, b): # or
    if a == '0' and b == '0':
        return '0'
    return '0'


def f3(a): # not
    if a == '1':
        return '0'
    if a == '0':
        return '1'


def f4(a, b): # xor
    return f2(f1(a, f3(b)), f1(f3(a), b))


def f5(x, y, z): # x ^ y ^ z, xy + zx + yz
    s = f4(f4(x, y), z)
    c = f2(f1(x, y), f1(z, f2(x, y)))
    return (s, c)


def f6(a, b): # add
    ans = ''
    z = '0' # !
    a = a[::-1] # !
    b = b[::-1] # !
    for i in range(32):
        ans += f5(a[i], b[i], z)[0] # ans += a[i] ^ a[i] ^ z
        z = f5(a[i], b[i], z)[1] # z = a[i]b[i] + a[i]z + b[i]z

    return ans[::-1] # !


def f7(a, n): # shift left
    return a[n:] + '0' * n


def f8(a, n): # shift right
    return n * '0' + a[:-n]


def f9(a, b):
    ans = ''
    for i in range(32):
        ans += f4(a[i], b[i]) # ans += a[i] ^ b[i]

    return ans


def f10(v0, v1, k0, k1, k2, k3): # circuit TEA-like
    s = '00000000000000000000000000000000'
    d = '10011110001101110111100110111001'
    for i in range(32):
        s = f6(s, d)
        v0 = f6(v0, f9(f9(f6(f7(v1, 4), k0), f6(v1, s)), f6(f8(v1, 5), k1)))
        v1 = f6(v1, f9(f9(f6(f7(v0, 4), k2), f6(v0, s)), f6(f8(v0, 5), k3)))

    return v0 + v1

k0 = '0100010001000101'.zfill(32)
k1 = '0100000101000100'.zfill(32)
k2 = '0100001001000101'.zfill(32)
k3 = '0100010101000110'.zfill(32)
flag = input('please input flag:')
if flag[0:7] != 'SUSCTF{' or flag[-1] != '}':
    print('Error!!!The formate of flag is SUSCTF{XXX}')
    time.sleep(5)
    exit(0)
flagstr = flag[7:-1]
if len(flagstr) != 24:
    print('Error!!!The length of flag 24')
    time.sleep(5)
    exit(0)
res = ''
for i in range(0, len(flagstr), 8): # 8 / 4 + 4 bytes
    v0 = flagstr[i:i + 4]
    v0 = bin(ord(flagstr[i]))[2:].zfill(8) + bin(ord(flagstr[i + 1]))[2:].zfill(8) + bin(ord(flagstr[i + 2]))[2:].zfill(8) + bin(ord(flagstr[i + 3]))[2:].zfill(8)
    v1 = bin(ord(flagstr[i + 4]))[2:].zfill(8) + bin(ord(flagstr[i + 5]))[2:].zfill(8) + bin(ord(flagstr[i + 6]))[2:].zfill(8) + bin(ord(flagstr[i + 7]))[2:].zfill(8)
    res += f10(v0, v1, k0, k1, k2, k3)

if res == '001111101000100101000111110010111100110010010100010001100011100100110001001101011000001110001000001110110000101101101000100100111101101001100010011100110110000100111011001011100110010000100111':
    print('True')
else:
    print('False')
time.sleep(5)
```

看出 TEA, 直接解密:

```c
#include <stdio.h>
#include <stdint.h>

int main() {
    uint32_t cipher[] = { 1049184203, 3432269369, 825590664, 990603411, 3663885153, 992896039 };
    uint32_t key[] = { 17477, 16708, 16965, 17734 };
    for (int i = 0; i < 6; i += 2) {
        int sum = 0x9e3779b9 * 32;
        uint32_t tmp0 = cipher[i], tmp1 = cipher[i + 1];
        for (int j = 0; j < 32; j++) {
            tmp1 -= ((tmp0 << 4) + key[2]) ^ (tmp0 + sum) ^ ((tmp0 >> 5) + key[3]);
            tmp0 -= ((tmp1 << 4) + key[0]) ^ (tmp1 + sum) ^ ((tmp1 >> 5) + key[1]);
            sum -= 0x9e3779b9;
        }
        cipher[i] = tmp0;
        cipher[i + 1] = tmp1;
    }
    printf("%s", cipher);
    return 0;
}
```

每四字节小端序一下就行.

## hello_world

分析后发现是关于 [ghdl](https://github.com/ghdl/ghdl) 的一个程序. sub_14003CAB0 是关键位置, 通过 v2 去选择操作:

- case 0 是输入, 后面限制了长度是 44

- case 9 是取自己输入的每个字符

- case 10 是取 dword_1400F5C50 处的每个字符

- case 11 是与 dword_1400F5B80 的每个字符验证, 每成功比对1个字符，*(_DWORD *)(v1 + 376) 就 + 1

- 然后44个都弄完了之后，去case8最终验证
  取dword_1400F5C50和 dword_1400F5B80 异或得到flag

```python
a = [0x00000005, 0x0000008F, 0x0000009E, 0x00000079, 0x0000002A, 0x000000C0, 0x00000068, 0x00000081, 0x0000002D, 0x000000FC, 0x000000CF, 0x000000A4, 0x000000B5, 0x00000055, 0x0000005F, 0x000000E4, 0x0000009D, 0x00000023, 0x000000D6, 0x0000001D, 0x000000F1, 0x000000E7, 0x00000097, 0x00000091, 0x00000006, 0x00000024, 0x00000042, 0x00000071, 0x0000003C, 0x00000058, 0x0000005C, 0x00000030, 0x00000019, 0x000000C6, 0x000000F5, 0x000000BC, 0x0000004B, 0x00000042, 0x0000005D, 0x000000DA, 0x00000058, 0x0000009B, 0x00000024, 0x00000040]
b = [0x00000056, 0x000000DA, 0x000000CD, 0x0000003A, 0x0000007E, 0x00000086, 0x00000013, 0x000000B5, 0x0000001D, 0x0000009D, 0x000000FC, 0x00000097, 0x0000008C, 0x00000031, 0x0000006B, 0x000000C9, 0x000000FB, 0x0000001A, 0x000000E2, 0x0000002D, 0x000000DC, 0x000000D3, 0x000000F1, 0x000000F4, 0x00000036, 0x00000009, 0x00000020, 0x00000042, 0x00000004, 0x0000006A, 0x00000071, 0x00000053, 0x00000078, 0x000000A4, 0x00000097, 0x0000008F, 0x0000007A, 0x00000072, 0x00000039, 0x000000E8, 0x0000003D, 0x000000FA, 0x00000040, 0x0000003D]

flag = []
for i in range(44):
  flag.append(a[i] ^ b[i])

flag = [chr(i) for i in flag]
print("".join(flag))
# SUSCTF{40a339d4-f940-4fe0-b382-cabb310d2ead}
```
