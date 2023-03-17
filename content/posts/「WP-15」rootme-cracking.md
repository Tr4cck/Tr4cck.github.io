---
title: 「Rootme-Cracking」做题平台练习记录
date: 2022-01-13 23:34:37
tags:
- Rev
categories:
- TECHNOLOGY
---

## ch1

```cpp
int __cdecl main(int argc, const char **argv, const char **envp)
{
  char *s1; // [esp+1Ch] [ebp-Ch]
  char *s1a; // [esp+1Ch] [ebp-Ch]

  puts("############################################################");
  puts("##        Bienvennue dans ce challenge de cracking        ##");
  puts("############################################################\n");
  printf("Veuillez entrer le mot de passe : ");
  s1a = (char *)getString(s1);
  if ( !strcmp(s1a, "123456789") )
    printf("Bien joue, vous pouvez valider l'epreuve avec le pass : %s!\n", "123456789");
  else
    puts("Dommage, essaye encore une fois.");
  return 0;
}
```

## ch2

printf 没有给出参数, 看汇编即可发现 password

## ch3

找字符串发现 main 函数, 发现明文比较

## ch4

```python
key = [0x18, 0xD6, 0x15, 0xCA, 0xFA, 0x77]
magic = [0x50, 0xB3, 0x67, 0xAF, 0xA5, 0x0E, 0x77, 0xA3, 0x4A, 0xA2, 0x9B, 0x01, 0x7D, 0x89, 0x61, 0xA5, 0xA5, 0x02, 0x76, 0xB2, 0x70, 0xB8, 0x89, 0x03, 0x79, 0xB8, 0x71, 0x95, 0x9B, 0x28, 0x74, 0xBF, 0x61, 0xBE, 0x96, 0x12, 0x47, 0x95, 0x3E, 0xE1, 0xA5, 0x04, 0x6C, 0xA3, 0x73, 0xAC, 0x89]
print(bytes([magic[i] ^ key[i%len(key)] for i in range(len(magic))]))
```

## ch5

```c#
private void Button1_Click(object sender, EventArgs e)
{
      if (Operators.CompareString(this.TextBox1.Text, "DotNetOP", false) == 0)
      {
        int num1 = (int) Interaction.MsgBox((object) "Bravo! Vous pouvez valider avec ce mot de passe\r\nWell done! You can validate with this password");
      }
      else
      {
        int num2 = (int) Interaction.MsgBox((object) "Mauvais mot de passe\r\nBad password");
      }
}
```

## ch6

```c
puts("crack-me for Root-me by s4r");
  puts("Enter password please");
  fgets(v5, 64, stdin);
  *((_BYTE *)&i + strlen(v5) + 3) = 0;
  if ( strlen(v5) != 19 )
    goto LABEL_21;
  for ( i = 8; i < 17; ++i )
  {
    if ( *((_BYTE *)&i + i + 4) != 'i' )
      goto LABEL_21;
  }
  if ( v11 == 's'
    && v10 == 'p'
    && v9 == 'm'
    && v5[2] == 'n'
    && v8 == 'n'
    && v5[0] == 'c'
    && v5[1] == 'a'
    && v5[3] == 't'
    && v6 == 'r'
    && v7 == v6 + 3 )
  {
    sub_4007C0();
  }
```

结合汇编大概是一个先填充, 然后按位比较的过程

## ch7

golang 基础

```python
xx = bytes([0x3b, 0x02, 0x23, 0x1b, 0x1b, 0x0c, 0x1c, 0x08, 0x28, 0x1b, 0x21, 0x04, 0x1c, 0x0b])
key = b'rootme'
print(bytes([xx[i] ^ key[i%len(key)] for i in range(len(xx))]))
# b'ImLovingGoLand'
```

## ch8

是一个输出 flag 的题应该可以通过 gdb 直接调出来. 下面是输出逻辑:

```c
void __noreturn blowfish()
{
  unsigned int i; // [esp+Ch] [ebp-4Ch]
  char *v1; // [esp+14h] [ebp-44h]
  char v2[35]; // [esp+18h] [ebp-40h] BYREF
  int v3[5]; // [esp+3Bh] [ebp-1Dh] BYREF
  char v4[9]; // [esp+4Fh] [ebp-9h] BYREF

  *(_DWORD *)&v4[5] = __readgsdword(0x14u);
  strcpy(v2, "liberté!");
  for ( i = 0; i < 0x14; i += 4 )
    *(_DWORD *)&v2[i + 12] = 0;
  v1 = &v2[i + 12];
  *(_WORD *)v1 = 0;
  v1[2] = 0;
  qmemcpy(v3, "_0cGjc5m_.5T3Ç8", 15);
  v3[4] = '\xC30JC';
  strcpy(v4, "\x809H9");
  printf("'+) Authentification réussie...\n U'r root! \n\n sh 3.0 # password: %s\n\r", v2);
  exit(0);
}
```

## ch9

![](https://s2.loli.net/2022/01/14/GbFiuP8CDoSnad2.png)

file 完然后运行一下, 发现就是一个口令验证器:

![](https://s2.loli.net/2022/01/14/PmZNFytAeQCk2wG.png)

一处简单的花指令, patch 之后是一个明文比较

## ch10

```js
var jsEventHandler = function jsEventHandler(event) {
      // Increment nesting count for the event handler.
      ++JSEvents.inEventHandler;
      JSEvents.currentEventHandler = eventHandler;
      // Process any old deferred calls the user has placed.
      JSEvents.runDeferredCalls();
      // Process the actual event, calls back to user C code handler.
      eventHandler.handlerFunc(event);
      // Process any new deferred calls that were placed right now from this event handler.
      JSEvents.runDeferredCalls();
      // Out of event handler - restore nesting count.
      --JSEvents.inEventHandler;
};
```

发现输入之后触发这个地方的 js. 惭愧, 我不是很懂 js 相关技术, 好在题目提示了 wasm 之后去分析wasm. 先`gcc index.c -c -o index` 编译. 通过字符串可以发现有口令检测相关函数. 大概读了一下发现先在初始化了一个 md5 数据, 然后在 check_password 处进行 md5 的比较, 直接查到 md5.

## ch11

对于 arm 还有不少要去了解的, 好在现在的反编译工具已经特别方便, 但这其实也是阻碍进步的一把双刃剑.

题目是六个方程, 刚好能解出来.

## ch12

一个比较简单的 keygen 的题目, 猜测应该是出题人用汇编手写出来的, 先从 `byte_600260` 出读 32 个字节, 再从 `.m.key` 中连续着读 32 个字节, 通过下面的 `check` 进行比对:

```c
__int64 __fastcall sub_400146(__int64 a1, __int64 a2)
{
  __int64 v2; // rax
  __int64 i; // rcx

  v2 = strlen(a1);
  if ( v2 == 1 )
    return 4919LL;
  for ( i = 0LL; i != v2 - 1; ++i )
  {
    if ( *(_BYTE *)(a1 + i) - (_BYTE)i + 0x14 != *(_BYTE *)(a2 + i) )
      return 4919LL;
  }
  return 0LL;
```

来个 keygen:

```python
import hashlib
name = b'root-me.org'
passwd = bytes([name[i] - i + 0x14 for i in range(len(name))])
print(hashlib.sha256(passwd).hexdigest())
```

## ch13

rider 里面看一下, 下面是生成字符串：

```c#
public static string mqsldfksdfgljk(string data, string key)
{
  int length1 = data.Length;
  int length2 = key.Length;
  char[] chArray = new char[checked (length1 - 1 + 1)];
  int num = checked (length1 - 1);
  int index = 0;
  while (index <= num)
  {
    chArray[index] = Strings.ChrW(Strings.Asc(data[index]) ^ Strings.Asc(key[index % length2]));
    checked { ++index; }
  }
  return new string(chArray);
}
```

下面是调用, 输入内容与上面生成的字符串比较:

```c#
private void Button1_Click(object sender, EventArgs e)
{
  string data = Encoding.UTF8.GetString(Convert.FromBase64String("B29mVBJ7cDdtRAYAHh0GKw1yX1g4IV9QOA=="));
  string key = "I_Gu3$$_Y0u_Ju5t_Fl4gg3d_!!!";
  // ISSUE: reference to a compiler-generated method
  MySettingsProperty.mqsldfksdfgljk(data, key);
  // ISSUE: reference to a compiler-generated method
  if (Operators.CompareString(this.TextBox1.Text, MySettingsProperty.mqsldfksdfgljk(data, key), false) == 0)
  {
    int num1 = (int) Interaction.MsgBox((object) "GG !!! Vous pouvez valider le challenge avec ce mot de passe.");
  }
  else
  {
    int num2 = (int) Interaction.MsgBox((object) "Nop");
  }
}
```

太久不写 C# 了, 所以就用 python 模拟一个:

```python
import base64
key = b'I_Gu3$$_Y0u_Ju5t_Fl4gg3d_!!!'
data = base64.b64decode('B29mVBJ7cDdtRAYAHh0GKw1yX1g4IV9QOA==')
print(bytes([data[i] ^ key[i % len(key)] for i in range(len(data))]))
```

## ch14

rider 里面看一下, 只有一个 Program 类, 猜测是个纯逆算法的题.



## ch15

file 发现为 python3.1 的 bytecode, 直接 pycdc 可以看到源码:

```python
# Source Generated with Decompyle++
# File: ch19.pyc (Python 3.1)

Warning: Stack history is not empty!
if __name__ == '__main__':
    print('Welcome to the RootMe python crackme')
    PASS = input('Enter the Flag: ')
    KEY = 'I know, you love decrypting Byte Code !'
    I = 5
    SOLUCE = [
        57,
        73,
        79,
        16,
        18,
        26,
        74,
        50,
        13,
        38,
        13,
        79,
        86,
        86,
        87]
    KEYOUT = []
    for X in PASS:
        KEYOUT.append((ord(X) + I ^ ord(KEY[I])) % 255)
        I = (I + 1) % len(KEY)

    if SOLUCE == KEYOUT:
        print('You Win')
    else:
        print('Try Again !')
```

简单逆一下就行:

```python
SOLUCE = [57, 73, 79, 16, 18, 26, 74, 50, 13, 38, 13, 79, 86, 86, 87]
KEY = b'I know, you love decrypting Byte Code !'
I = 5
flag = []
for x in SOLUCE:
    flag.append((x  ^ KEY[I]) - I)
    I = (I + 1) % len(KEY)

print(bytes(flag))
```

## ch16

```python
from malduck import *

init_seed = 0x563b2b37
key = [ror(init, 1, 32) for i in range(25)]
print(key)
```

## ch17 | LUA

```
Reverse/Root-Me/ch17 
➜ file ch45.out 
ch45.out: Lua bytecode, version 5.1
```

## ch18 | Mac

## ch19

实现了一个矩阵一样的数据结构, 里面依次往里填充 `ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_` 然后进行比较.

## ch20

开局一个 ptrace 检测反调, password 在运行时以命令行的形式传入. 输入的每一个字符约束为 `ch & 8 == 0`, 随便模拟一下生成目标字符串:

```c
#include <stdio.h>

void sub_804851C(unsigned char *a1, unsigned char *s)
{
  unsigned char *v2; // esi
  unsigned char *v4; // ecx
  unsigned char v5; // al
  int v6; // ebx
  int v7; // edx
  unsigned char v8; // al
  unsigned char *v9; // edx
  unsigned char v10; // al
  unsigned char v11; // al
  unsigned char v12; // al
  unsigned char v13; // al
  unsigned char v14; // al

  v2 = a1;
  *a1 ^= 0xABu;
  v4 = a1 + 1;
  v5 = a1[1];
  if ( v5 )
  {
    v6 = 1;
    v7 = 1;
    do
    {
      v8 = v5 - v6;
      *v4 = v8;
      v9 = &a1[v7 - 1];
      v10 = *v9 ^ v8;
      *v4 = v10;
      v11 = *v9 + v10;
      *v4 = v11;
      v12 = *v9 ^ v11;
      *v4 = v12;
      v13 = a1[8] ^ v12;
      *v4 = v13;
      if ( !v13 )
        *v4 = 1;
      v7 = ++v6;
      v4 = &a1[v6];
      v5 = a1[v6];
    }
    while ( v5 );
  }
  v14 = *a1;
  if ( *a1 )
  {
    do
    {
      sprintf((char *)s, "%.2x", v14);
      s += 2;
      v14 = *++v2;
    }
    while ( *v2 );
  }
}

int main() {
  unsigned char go[10000];
  char s[] = "THEPASSWORDISEASYTOCRACK";
  sub_804851C(s, go);
  printf("%s\n", go);
  return 0;
}
```

## ch21

可以改跳转, 也可以通过溢出覆盖函数指针的值让它指向 asm 函数.

## ch22

边调边改寄存器, 即对比结果, 然后在内存中把程序事先有的正确 password 拿出来, 往里面一输:

![](https://s3.bmp.ovh/imgs/2022/01/1cd496388a34ce38.png)

## ch23

## ch24
