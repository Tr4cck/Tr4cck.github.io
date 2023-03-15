---
title: 「DefcampCTF2022」Rev 部分题解
date: 2022-03-31 18:48:51
tags:
- Rev
categories:
- TECHNOLOGY
---

## algorithm
给了源代码：
```python
flag = '[test]'
hflag = flag.encode('hex')
iflag = int(hflag[2:], 16)

def polinom(n, m):
    i = 0
    z = []
    s = 0
    while n > 0:
        if n % 2 != 0:
           z.append(2 - (n % 4))
        else:
           z.append(0)
        n = (n - z[i])/2
        i = i + 1
    z = z[::-1]
    l = len(z)
    for i in range(0, l):
        s += z[i] * m ** (l - 1 - i)
    return s

i = 0
r = ''
while i < len(str(iflag)):
    d = str(iflag)[i:i+2]
    nf = polinom(int(d), 3)
    r += str(nf)
    i += 2

print(r)

# 242712673639869973827786401934639193473972235217215301
```
可以看出是 10 进制 flag 每两位传入 polinom，考虑直接打表开爆：
```python
import binascii
from pprint import pprint

def polinom(n, m):
    i = 0
    z = []
    s = 0
    while n > 0:
        if n % 2 != 0:
           z.append(2 - (n % 4)) # odd -> 1 or -1
        else:
           z.append(0) # even -> 0
        n = (n - z[i]) // 2
        i = i + 1
    z = z[::-1]
    l = len(z)
    for i in range(0, l):
        s += z[i] * m ** (l - 1 - i) # z_{l-1}*m^{l-1} + z_{l-2}*m^{l-2} + ... + z_0
    return s

ftable = {i : polinom(i, 3) for i in range(100)}
re_ftable = {v : k for k, v in ftable.items()}

cipher = '242712673639869973827786401934639193473972235217215301'
num = ''
n = 0
while n < len(cipher):
    for i in range(4, 0, -1):
        t = int(cipher[n : n + i], 10)
        try:
            lala = str(re_ftable[t])
            if len(lala) == 1:
                lala = '0' + lala
            num += lala
            n += i
            break
        except KeyError:
            continue
print(num)
```
注意最后一位可能是 1 个数，也可能是 2 个数，那只需要挨个去试一下就好了

## rsa-factory-RE部分
> 题目使用了 cython 中的 `cythonize` 功能将 python 转译成 c 的实现，然后再编译出 **共享目标文件** 。

### 大致框架
具体而言，将文件拖进 IDA，通过 Strings Window 可以发现不少位于 rodata 段的 python 字符串，即 `__pyx_string_tab` ，并且通过 xref 可以定位到关键函数 `_pyx_pymod_exec_test`。里面首先检查了一下 python 版本为 3.6，大概知道意思就不用扣细节了，因为有很多不太重要的边界检查之类的操作。

下面是 import 一些模块：
```c
// from Crypto.Util.number import *
v29 = (__int64 *)_Pyx_Import_constprop_24(_pyx_n_s_Crypto_Util_number, v8);

// from math import gcd
v62 = _pyx_n_s_gcd;
v63 = (__int64 *)v29[3];
++*(_QWORD *)_pyx_n_s_gcd;
*v63 = v62;
v8 = (__int64 *)_Pyx_Import_constprop_24(_pyx_n_s_math, v29);

// from flag import FLAG
v29 = (__int64 *)_Pyx_Import_constprop_24(_pyx_n_s_flag, v8);
if ( !v29 ) // 很多这种异常处理，其实可以就不看也没啥关系
{
    v59 = 0LL;
    v60 = 0LL;
    v178 = 0LL;
    v176 = 7;
    v177 = 2865;
    goto LABEL_158;
}
v44 = (*v8)-- == 1;
if ( v44 )
    (*(void (__fastcall **)(__int64 *))(v8[1] + 48))(v8);
v67 = _Pyx_ImportFrom(v29, _pyx_n_s_FLAG);
```

下面的 `_pyx_d` 是一个全局字典，可以通过它去索引一般的 PyObject：
```c
PyDict_SetItem(_pyx_d, _pyx_n_s_gcd, v64)
```

然后设置了两个 CyFunction 分别是 `keygen` 和 `encrypt`：
```c
v68 = _Pyx_CyFunction_New_constprop_22(
    &_pyx_mdef_4test_1keygen,
    _pyx_n_s_keygen,
    _pyx_n_s_test,
    _pyx_d,
    _pyx_codeobj__3);

v69 = _Pyx_CyFunction_New_constprop_22(
    &_pyx_mdef_4test_3encrypt,
    _pyx_n_s_encrypt,
    _pyx_n_s_test,
    _pyx_d,
    _pyx_codeobj__5);
```

然后是设置一些变量的值：
```c
// nbit = 1024 dbit = 256
v29 = (__int64 *)_pyx_int_1024;
v8 = (__int64 *)_pyx_int_256;
v70 = _pyx_n_s_nbit;
v71 = _pyx_d;
++*(_QWORD *)_pyx_int_1024;
++*v8;
if ( (int)PyDict_SetItem(v71, v70, v29) < 0 )
```

下面的内容其实也大同小异，直接给出大概的翻译结果：
```python
e, n1, n2 = keygen(dbit, nbit)
FLAG = long(FLAG.encode("utf-8").hex(), 16)
c1 = encrypt(FLAG, (e, n1))
c2 = encrypt(FLAG, (e, n2))
print(f'e = {e}')
print(f'n1 = {n1}')
print(f'n2 = {n2}')
print(f'enc1 = {c1}')
print(f'enc2 = {c2}')
```

### encrypt
参数是两个：
```c
v22 = _PyDict_GetItem_KnownHash(a3, _pyx_n_s_msg, *(_QWORD *)(_pyx_n_s_msg + 24));
v23 = (_QWORD *)_PyDict_GetItem_KnownHash(a3, _pyx_n_s_pubkey, *(_QWORD *)(_pyx_n_s_pubkey + 24));
```

其中 pubkey 是一个二元组，然后关键在于：
```c
v14 = (_QWORD *)PyNumber_Power(v8, v11, v12);
```

大概能看到是一个比较常规的 rsa 加密过程，所以：
```python
def encrypt(msg, pubkey : tuple):
    return pow(msg, pubkey[0], pubkey[1])
```

### keygen
同样是两个参数 dbit 和 nbit，分析流程和上述大概相同，就是长了很多，并且多次用到了前一千行处赋值的变量，尽量多给变量重命名可以提升一点点效率 :( ：
```python
def keygen(nbit, dbit):
    if dbit * 2 < nbit:
        while True:
            p = getRandomNBitInteger(dbit)
            q = getRandomNBitInteger(nbit // 2 - dbit)
            if isPrime(q * p + 1):
                break

        while True:
            p2 = getRandomNBitInteger(dbit)
            q2 = getRandomNBitInteger(nbit // 2 - dbit)
            if isPrime(p * q2 + 1) and isPrime (p2 * q2 + 1):
                break

        while True:
            while True:
                p3 = getRandomNBitInteger(dbit)
                if gcd(p * q * p2 * q2, p3) == 1:
                    break

            u = p * q
            v = p2 * q2
            if not isPrime(((inverse(u * v, p3) * p3 - 1) // (u * v)) * q + 1):
                continue

            return p3, (p * q + 1) * (p2 * q2, 1), (p * q2 + 1) * (((inverse(u * v, p3) * p3 - 1) // (u * v)) * q + 1)
```

## can-you-crack-this
要求输入一个 `public key` 和一个 `serial key`，关键明显在于 `verify_serial` 中，我们期望它返回 1。
```c
while ( 1 )
{
  v15 = std::string::find(a2, v17, 0LL);      // find "-" in serial key
  if ( v15 == -1 )
    break;
  ++v13;
  if ( v15 != 3 )
  {
    v18 = 0;
    v12 = 1;
    goto LABEL_12;
  }
  std::string::substr(v11, a2, 0LL, 3LL);
  std::string::operator=(v14, v11);
  std::string::~string(v11);
  std::string::basic_string(subStr, v14);
  v2 = idx++;
  v8 = (char *)std::string::at(a1, v2);
  v7 = !verify_char((__int64)subStr, *v8);
  std::string::~string(subStr);
  if ( v7 )
  {
    v18 = 0;
    v12 = 1;
    goto LABEL_12;
  }
  v6 = v15;
  v3 = std::string::length(v17);
  std::string::erase(a2, 0LL, v3 + v6);
}
```
一个主循环，大意是 `serial key` 是被 `-` 分割开的，我们将每组 split 后的结果连同 `public key` 的一个字符传入 `verify_char` 函数。
```c
_BOOL8 __fastcall verify_char(__int64 a1, char a2)
{
  const char *v2; // rax
  int v3; // eax
  unsigned __int64 v5; // [rsp+0h] [rbp-20h]
  unsigned __int64 v6; // [rsp+8h] [rbp-18h]
  signed __int64 v7; // [rsp+10h] [rbp-10h]

  v2 = (const char *)std::string::c_str(a1);
  v7 = strtol(v2, 0LL, 16);
  v6 = fib(v7);                                 // fib_{n+1} / 0x64
  v5 = v6;
  do
  {
    do
      --v5;
    while ( !check_arm(v5) );
  }
  while ( (unsigned int)notused(v5) || v5 > 0x10E47F4C575565LL );
                                                // v5 cannot be reused
                                                // v5 <= 0x10E47F4C575565
  if ( v7 < 100 )
    return 0;
  if ( !(unsigned int)isprint(a2) )             // printable
    return 0;
  v3 = UN++;
  used[v3] = v5;
  return v5 && v6 && v5 != v6 && (v6 - v5) % 0x64 == a2;
}
```
约束大概是这么几条：
- v5 必须是 `countSetBits` 之后返回奇数
- v5 不能重复
- v5 必须小于等于 0x10E47F4C575565
- v2 必须是 printable 字符
- 然后 fib(x) - v5 % 0x64 == a2

keygen 也不是特别难写，枚举一下就好了。虽然远程环境连不上了不知道对不对（逃：
```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>

__uint128_t fib(__uint128_t n) {
    __uint128_t v5[0x1000] = {0};
    v5[0] = 0;
    v5[1] = 1;
    for (int i = 2; i <= n; i++) {
        v5[i] = v5[i - 1] + v5[i - 2];
    }
    return v5[n] / 0x64;
}

int64_t countSetBits(int64_t x) {
    uint32_t v2;
    v2 = 0;
    while ( x ) {
      x &= x - 1;
      ++v2;
    }
    return v2;
}

bool check_arm(uint64_t v5) {
    return (countSetBits(v5) & 1) == 1;
}

void filter(int64_t *v5) {
    do {
        --*v5;
    } while(!check_arm(*v5));
    return ;
}

int main() {
    int64_t v5 = 0x10E47F4C575565;
    int serial_key[100];
    char public_key[20];
    for (int i = 0; i < 20; i++) {
        filter(&v5);
        for (int j = 100; j < 114514; j++) {
            int64_t maybe = (fib(j) - v5) % 0x64;
            if (maybe > 0x20) {
                serial_key[i] = j;
                public_key[i] = maybe;
                break;
            }
        }
    }
    printf("public key: ");
    for (int i = 0; i < 20; i++) {
        printf("%c", public_key[i]);
    }
    printf("\n");
    printf("serial key: ");
    for (int i = 0; i < 19; i++) {
        printf("%03x-", serial_key[i]);
    }
    printf("bss\n");
    return 0;
}
```
