---
title: 「HayyimCTF2022」Rev 部分题解
date: 2022-03-06 21:55:13
tags:
- Rev
categories:
- TECHNOLOGY
---


<!-- more -->

## ezrev

![](https://s2.loli.net/2022/03/06/7cDjRM6xHskCyL8.png)
调试到图示位置就好, 把经过加密的字符串拿出来, 异或常数回去, 再解 base64

## frontdoor

> 长亭大佬的博客：
> http://acez.re/the-weak-bug-exploiting-a-heap-overflow-in-vmware/
> 
> open-vm-tools源码：
> https://github.com/vmware/open-vm-tools.git

```
frontdoor: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=f47091b37f50d1af957cae20439f9847e30d4611, for GNU/Linux 3.2.0, stripped
```

无壳, 有 **ASLR**.

init 函数里面打开了 `/tmp/vm` 文件, 写入一段完整 elf 文件并赋予了可执行权限, dump 出来备用.

主函数中设置了 sigaction 信号处理变量, 将 SIGSEGV 与一个输出 "this binary must be executed inside vmware" 的函数绑定起来. 完成绑定后调用一个检测并初始化的函数:

```cpp
__int64 __fastcall sub_55D13CE0E723(int a1, __int64 a2, __int64 a3, __int64 a4)
{
  unsigned int i; // [rsp+2Ch] [rbp-54h]
  char v8[8]; // [rsp+30h] [rbp-50h] BYREF
  __int64 v9; // [rsp+38h] [rbp-48h]
  __int16 v10; // [rsp+40h] [rbp-40h]
  __int16 v11; // [rsp+42h] [rbp-3Eh]
  __int16 v12; // [rsp+4Ah] [rbp-36h]
  int v13; // [rsp+50h] [rbp-30h]
  int v14; // [rsp+58h] [rbp-28h]
  unsigned __int64 v15; // [rsp+78h] [rbp-8h]

  v15 = __readfsqword(0x28u);
  for ( i = 0x80000000; ; i = 0 )
  {
    v11 = 0;
    v9 = i | a1;
    v10 = 30;
    sub_55D13CE0E6F0((__int64)v8);
    if ( (v11 & 1) != 0 )
      break;
    if ( !i )
      return 0LL;
  }
  *(_WORD *)a2 = v12;
  *(_DWORD *)(a2 + 28) = v13;
  *(_DWORD *)(a2 + 32) = v14;
  *(_QWORD *)(a2 + 8) = a3;
  *(_QWORD *)(a2 + 16) = a4;
  *(_DWORD *)(a2 + 24) = a3 != 0;
  sub_55D13CE0E5D8();
  return 1LL;
}
```

主要看最后一次的调用, 里面是一个类似自解密的函数, 同时给一块内存进行了初始化. 试着调试的过程中遇到一些问题, 比如字符串初始化的时候直接触发信号导致无法调试, 或者直接跳转到输入的位置却发现 scanf 函数调用失败等等. 后来没办法, 还是用 vmware 起了个 ubuntu20.04 远程调试, 发现:

```cpp
v11 = 'a';
v12 = ' ';
for ( i = 0; i <= 25; ++i )
{
  s = (char *)&v11;
  v9 = (char *)&v12;
  v0 = strlen((const char *)&v11);
  v1 = strlen(v9);
  v10 = (char *)malloc(v0 + v1 + 100);
  v2 = sub_564E3006A378();
  v3 = (const char *)sub_564E3006A670((__int64)v2);
  sprintf(v10, v3, s, v9);
  sub_564E30067805(mem1, (__int64)v10, 4096LL);
  sub_564E30067A98(mem1, (__int64 *)v6, (unsigned __int64 *)v7);
  free(v10);
  LOBYTE(v11) = v11 + 1;
}
```

这个循环就是利用 RPCI 的 `info-set guestinfo.%c ` 初始化一系列小写字母的键值对为空, 完成输入为 26 * 'a' 之后进入一个类似的循环:

```cpp
var1C = 'a';
var1A = 'b';
for ( var4C = 0; var4C < strlen(a1); ++var4C )
{
  LOBYTE(var1A) = a1[var4C];
  var38 = (char *)&var1C;
  var30 = (char *)&var1A;
  rbx3 = strlen((const char *)&var1C);
  rax3 = strlen(var30);
  var28 = (char *)malloc(rbx3 + rax3 + 100);
  rax3a = sub_564E3006A378();
  rax3b = (const char *)sub_564E3006A670((__int64)rax3a);
  sprintf(var28, rax3b, var38, var30);
  sub_564E30067805(mem1, (__int64)var28, 4096LL);
  sub_564E30067A98(mem1, &var48, &var40);
  free(var28);
  LOBYTE(var1C) = var1C + 1;
}
```

再次调用 `info-set guestinfo.%c a` 设定值为 a, 即输入, 之后进入关键逻辑:

```cpp
v191 = __readfsqword(0x28u);
v107 = aElcvxntymrz;
s = (char *)malloc(0x65uLL);
v0 = sub_5581F9D264B4();
v1 = (const char *)sub_5581F9D2671E((__int64)v0);
sprintf(s, v1, aElcvxntymrz);
sub_5581F9D23805(mem1, (__int64)s, 4096LL);
sub_5581F9D23A98(mem1, &v105, (unsigned __int64 *)v106);
free(s);
```

调试发现是取出一个固定字符串为 key 的 value, 然后进行和字符串本身进行异或存入内存中. 之后进入一个循环:

```cpp
sum = 0;
v183 = 'a';
for ( i = 0; i <= 25; ++i )
{
  v159 = (char *)&v183;
  v52 = strlen((const char *)&v183);
  v160 = (char *)malloc(v52 + 100);
  v53 = sub_5581F9D264B4();
  v54 = (const char *)sub_5581F9D2671E((__int64)v53);
  sprintf(v160, v54, v159);
  sub_5581F9D23805(mem1, (__int64)v160, 4096LL);
  sub_5581F9D23A98(mem1, &v105, (unsigned __int64 *)v106);
  free(v160);
  sum += *(char *)(v105 + 2);
  LOBYTE(v183) = v183 + 1;
}
```

调试观察 sum 的变化发现作用是取出键 'a' ~ 'z' 对应的值, 即输入, 进行累加.

接下来一部分:

```cpp
sub_5581F9D233C9(
    ((xor_v | (xor_c << 8) | (xor_l << 16) | (xor_e << 24)) << (sum % 32 + 1)) | ((xor_v | (xor_c << 8) | (xor_l << 16) | (xor_e << 24)) >> (31 - sum % 32)),
    v184, // uninitialized memory
    0x10u);
v161 = "a1";
v162 = v184;
v55 = strlen(v184);
v163 = (char *)malloc(v55 + 102);
v56 = sub_5581F9D26378();
v57 = (const char *)sub_5581F9D26670((__int64)v56);
sprintf(v163, v57, v161, v162);
sub_5581F9D23805(mem1, (__int64)v163, 4096LL);
sub_5581F9D23A98(mem1, &v105, (unsigned __int64 *)v106);
free(v163);
```

`sub_5581F9D233C9` 第一个参数是一个循环 32-bit 的循环左移, 第二个参数是一块未初始化内存, 第三个参数表示 16 进制. 那我们可以大概猜测一下功能就是做一些操作之后写回内存:

```cpp
......

do
{
  v7 = v5 % a3;
  v5 /= (__int64)a3;
  v3 = v8++;
  if ( v7 <= 9 )
    *v3 = v7 + '0';
  else
    *v3 = v7 + 'W';
}
while ( v5 > 0 );

......

do
{
  v6 = *v9;
  *v9 = *v10;
  *v10 = v6;
  --v9;
  ++v10;
}
while ( v10 < v9 );
```

发现是加完然后头尾交换. 后面差不多就是一直重复上述操作.

然后最后它执行了一下我们最开始 dump 出来的 /tmp/vm, 应该是用来 check 之类的, 分析一下提出数据. 用最后一组数稍微测一下, 可以得到 sum % 32 == 28.
最终 exp:

```python
def ROL(a, b):
  return ((a << (b % 32)) | (a >> (32 - b % 32)))

def big2little(a):
  s = []
  for i in range(4):
    s.append(a & 0xff)
    a >>= 8
  return s[::-1]

enc = [(7, 0x23C34040), (3, 0x14380438), (5, 0x6098398), (7, 0xE0E2C3), (9, 0x30585858), (29, 0x81960C), (27, 0xE8AB41C)]
keys = list(b'elcvxntymrzoahwujigpqfdsbkEZ')
flag = ""

for s, v in enc:
  v = ROL(v, 28 + s)
  w = big2little(v)
  for t in range(4):
    flag += chr(w[t] ^ keys[0])
    keys = keys[1:]

flag = flag[:-2]
keys = 'elcvxntymrzoahwujigpqfdsbk'
for i in range(26):
  print(flag[keys.find(chr(i + 97))], end = '')
```

## Ouroboros

函数逻辑比较简单, 就是整理游戏规则比较麻烦, 下图是所有填充的情况:
![](https://s2.loli.net/2022/03/10/MbH6aRqk9cGBPAn.jpg)

然后看判断:

```cpp
for ( i = 0; i < 7; ++i )
{
  *v12 = *&byte_41B000[4 * i];
  v10 = map[6 * v12[2] + v12[1]];
  v11 = 0;
  if ( v12[0] )                               // v12[0] == 1
  {
    if ( (v10 & 0x20) != 0 )
    {
      for ( j = v12[2] - 1; j > -1; --j )
      {
        ++v11;
        if ( map[6 * j + v12[1]] != 32 )
          break;
      }
      for ( k = v12[2] + 1; k < 6; ++k )
      {
        ++v11;
        if ( map[6 * k + v12[1]] != 32 )
          break;
      }
    }
    else
    {
      if ( (v10 & 0x10) == 0 )
        return 0;
      for ( l = v12[1] - 1; l > -1; --l )
      {
        ++v11;
        if ( map[6 * v12[2] + l] != 16 )
          break;
      }
      for ( m = v12[1] + 1; m < 6; ++m )
      {
        ++v11;
        if ( map[6 * v12[2] + m] != 16 )
          break;
      }
    }
  }
  else                                        // v12[0] == 0
  {
    if ( (v10 & 1) != 0 )                     // x -> 0
    {
      for ( n = v12[1] - 1; n > -1; --n )
      {
        ++v11;
        if ( map[6 * v12[2] + n] != 16 )
          break;
      }
    }
    else
    {
      if ( (v10 & 2) == 0 )                   // x -> +∞
        return 0;
      for ( ii = v12[1] + 1; ii < 6; ++ii )
      {
        ++v11;
        if ( map[6 * v12[2] + ii] != 16 )
          break;
      }
    }
    if ( (v10 & 4) != 0 )                     // y -> 0
    {
      for ( jj = v12[2] - 1; jj > -1; --jj )
      {
        ++v11;
        if ( map[6 * jj + v12[1]] != 32 )
          break;
      }
    }
    else
    {
      if ( (v10 & 8) == 0 )                   // y -> +∞
        return 0;
      for ( kk = v12[2] + 1; kk < 6; ++kk )
      {
        ++v11;
        if ( map[6 * kk + v12[1]] != 32 )
          break;
      }
    }
  }
  if ( v11 != v12[3] )
    return 0;
}
```

发现和题目名一样, 要求连起来, 然后提取一下点, 手动连一下就行.
