---
title: 「starCTF2021」Rev | wherekey 本地 socket 通信
date: 2021-06-13 08:17:36
tags: Rev
categories: TECHNOLOGY
mathjax: true
toc: true
---


## Step One

- ELF64位程序，无壳，无符号表
- 运行提示输入key，随便输点东西，直接退出

## Step Two

拖进IDA找start函数，第一个参数就是main函数

```c
// positive sp value has been detected, the output may be wrong!
void __fastcall __noreturn start(__int64 a1, __int64 a2, int a3)
{
  __int64 v3; // rax
  int v4; // esi
  __int64 v5; // [rsp-8h] [rbp-8h] BYREF
  void *retaddr; // [rsp+0h] [rbp+0h] BYREF

  v4 = v5;
  v5 = v3;
  sub_402B60(
    (unsigned int)main,
    v4,
    (unsigned int)&retaddr,
    (unsigned int)sub_4037C0,
    (unsigned int)sub_403860,
    a3,
    (__int64)&v5);
  __halt();
}
```

对main函数中的函数名和变量进行重命名，通过它们调用的一些系统调用可以做到这一点，我没处理完，不过大概长这样

```c
// local variable allocation has failed, the output may be wrong!
int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
{
  int v3; // edx
  __int64 v4; // rsi
  __int64 v5; // rax
  __int64 v6; // rdx
  __int64 v7; // rcx
  u32 *v8; // r8
  u32 v9; // er9
  __int64 v10; // rax
  __int64 v11; // rdx
  __int64 v12; // rcx
  u32 *v13; // r8
  u32 v14; // er9
  fd_set v15; // [rsp+10h] [rbp-90h] BYREF
  unsigned __int64 v16; // [rsp+98h] [rbp-8h]

  v16 = __readfsqword(0x28u);
  socket = sub_40245B();
  sockfd = sub_4024EB();
  dword_4C8570 = sub_40257B(*(__int64 *)&argc, (__int64)argv, v3);
  printf((__int64)"Please enter the key:");
  while ( 1 )
  {
    memset(&v15, 0, sizeof(v15));
    v15.fds_bits[dword_4C8570 / 64] |= 1LL << (dword_4C8570 % 64);
    v15.fds_bits[sockfd / 64] |= 1LL << (sockfd % 64);
    sub_44E080(1024, &v15, 0LL, 0LL, 0LL);
    if ( (v15.fds_bits[sockfd / 64] & (1LL << (sockfd % 64))) != 0 )
      sub_40223C();                             // here
    v4 = v15.fds_bits[dword_4C8570 / 64];
    if ( (v4 & (1LL << (dword_4C8570 % 64))) != 0 )
    {
      ((void (__fastcall *)(__int64))((char *)&sub_401D44 + 1))((__int64)&unk_4C5120);
      sub_411D40(v5, v4, v6, v7, v8, v9);
      sub_402072();
      ((void (__fastcall *)(__int64))((char *)&sub_401D44 + 1))((__int64)&unk_4C5130);
      sub_411D40(v10, v4, v11, v12, v13, v14);
    }
  }
}
```

容易找到`sub_40223C`，再同样进行重命名如下

```c
// local variable allocation has failed, the output may be wrong!
unsigned __int64 sub_40223C()
{
  __int64 v0; // rsi
  __int64 v1; // rdi
  __int64 v2; // rdx
  int v3; // ecx
  int v4; // er8
  int v5; // er9
  unsigned __int64 result; // rax
  socklen_t fromlen; // [rsp+8h] [rbp-18h] OVERLAPPED BYREF
  signed int isRecv_4; // [rsp+Ch] [rbp-14h]
  void *buf; // [rsp+10h] [rbp-10h]
  unsigned __int64 v10; // [rsp+18h] [rbp-8h]

  v10 = __readfsqword(0x28u);
  isRecv_4 = 0;
  fromlen = 16;
  buf = (void *)malloc(10LL);
  sub_401120();
  v0 = (__int64)buf;
  v1 = (unsigned int)sockfd;
  isRecv_4 = recvfrom(sockfd, buf, 5uLL, 0, &from, &fromlen);
  if ( isRecv_4 > 0 )
  {
    v1 = (__int64)buf;
    sub_4022DE((char *)buf);                    // here
  }
  result = __readfsqword(0x28u) ^ v10;
  if ( result )
    sub_450C80(v1, v0, v2, v3, v4, v5);
  return result;
}
```

到这里，结合hint，猜测是用户通过tty输入内容，并通过socket发送至本地，再判断对错

容易发现`sub_4022DE`是接收到输入后的下一步操作

```c
unsigned __int64 __fastcall sub_4022DE(char *a1)
{
  char *v2; // rdi
  __int64 v3; // rcx
  __int64 v4; // r8
  __int64 v5; // r9
  __int64 v6; // rdx
  unsigned __int64 result; // rax
  int i; // [rsp+18h] [rbp-18h]
  int v9; // [rsp+1Ch] [rbp-14h]
  char v10[5]; // [rsp+23h] [rbp-Dh] BYREF
  unsigned __int64 v11; // [rsp+28h] [rbp-8h]

  v11 = __readfsqword(0x28u);
  if ( !inp )
    inp = malloc(30LL);
  sub_401090();
  v2 = (char *)inp;
  v9 = (unsigned int)sub_401190();
  v6 = (unsigned int)cnt;
  if ( v9 >= 5 * cnt )
  {
    for ( i = 0; i <= 4; ++i )
    {
      nullsub_3();
      LayerOne((__int64)v10, i);
      a1 = v10;
      v2 = (char *)(5 * cnt + inp);
      v3 = (unsigned int)((int)LayerTwo(v2, v10, 5) % 257);
      v6 = (__int64)res;
      res[5 * cnt + i] = v3;
    }
    ++cnt;
  }
  if ( cnt > 4 )
    return proc_then();
  result = __readfsqword(0x28u) ^ v11;
  if ( result )
    sub_450C80((__int64)v2, (__int64)a1, v6, v3, v4, v5);
  return result;
}
```

**注：这里有一个mod257，待会儿反解的时候是需要考虑到的**

俩加密函数内容如下

```c
__int64 __fastcall LayerOne(__int64 a1, int a2)
{
  int v2; // eax
  int counter; // [rsp+18h] [rbp-4h]

  counter = 0;
  while ( a2 <= 24 )
  {
    v2 = counter++;
    *(_BYTE *)(a1 + v2) = aFlagAreYouSure[a2];
    a2 += 5;
  }
  return nullsub_2();
}
```

> 这个是把fake flag分成五组，取每一组的第一个字符，放在a1中

```c
// a3 = 5
__int64 __fastcall LayerTwo(char *a1, char *a2, int a3)
{
  unsigned int v4; // [rsp+1Ch] [rbp-8h]
  int i; // [rsp+20h] [rbp-4h]

  v4 = 0;
  for ( i = 0; i < a3; ++i )
    v4 += a1[i] * a2[i];
  return v4;
}
```

> 把两个数组的对应位置相乘再相加

结合前面`flag`是一列一列取，而`inp`是一行一行取，还有一个`LayerTwo`是相乘，容易想到这就是一个矩阵乘法。有了这个认知后，跟进`proc_then`进行查看

```c
unsigned __int64 __fastcall proc_then()
{
  __int64 v0; // rbp
  __int64 aim; // rdi
  __int64 v2; // rdx
  __int64 v3; // rcx
  __int64 v4; // r8
  __int64 v5; // r9
  unsigned __int64 result; // rax

  aim = (__int64)&unk_4C5150;
  if ( !(unsigned int)sub_4010F0() )
  {
    aim = 3LL;
    sub_40FF90(3);
  }
  result = __readfsqword(0x28u) ^ *(_QWORD *)(v0 - 8);
  if ( result )
    sub_450C80(aim, (__int64)res, v2, v3, v4, v5);
  return result;
}
```

发现`aim`是乘完的结果，跟进`sub_40FF90`，发现他是`syscall`函数，结合系统调用号3，这里应该是close掉socket连接

~~然后后面就不想分析了~~

## Step Three

写一下解密脚本，需要一点线性代数知识

$$
AB=X\\
\rarr A=XB^{-1}\\
其中A代表输入，因为取的是行\\
B代表fake\quad flag生成的矩阵，因为取的是列\\
而X就是加密后的结果了

$$

使用sage求解（用`numpy`、`matlab`都可以。~~甚至手算也可以~~）

```sage
sage: x = Matrix(GF(257), [[0x38,0x6D,0x4B,0x4B,0xB9],
....:                      [0x8A,0xF9,0x8A,0xBB,0x5C],
....:                      [0x8A,0x9A,0xBA,0x6B,0xD2],
....:                      [0xC6,0xBB,0x05,0x90,0x56],
....:                      [0x93,0xE6,0x12,0xBD,0x4F]])
sage: b = Matrix(GF(257), [[102, 108, 97,  103, 123],
....: ....:                [97, 114, 101, 95,  121],
....: ....:                [111, 117, 95,  115, 117],
....: ....:                [114, 101, 95,  102, 114],
....: ....:                [105, 101, 110, 100, 125]])
sage: flag = x*b.inverse()

sage: for i in flag:
....:     for j in i:
....:         print(chr(j))
....:       
H
a
2
3
_
f
0
n
_
9
n
d
_
G
0
o
d
-
1
u
c
k
-
O
H
```

得到`flag{Ha23_f0n_9nd_G0od_luck_OH}`

## FIN
