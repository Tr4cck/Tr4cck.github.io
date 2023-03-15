---
title: 「ReversingKR」缓慢刷题
date: 2021-07-18 15:41:47
tags: Rev
categories: TECHNOLOGY
mathjax: true
toc: true
---


## 前言

BBS上的资源都没有好好利用（懊恼。从头翻BBS，看到Skye师傅17年发的一个做题网站`ReversingKR`

## Easy-CrackMe

VC写的32位exe程序，我们可以通过字符串定位到关键函数

```c
int __cdecl sub_401080(HWND hDlg)
{
  CHAR String[100]; // [esp+4h] [ebp-64h] BYREF 存储输入

  String[0] = 0; // 以下四行表示一些初始化的内容
  memset(&String[1], 0, 0x60u);
  *(_WORD *)&String[97] = 0;
  String[99] = 0;
  GetDlgItemTextA(hDlg, 1000, String, 100); // 获取输入的函数
  if ( String[1] != 'a' || strncmp(&String[2], Str2, 2u) || strcmp(&String[4], aR3versing) || String[0] != 'E' ) // 输入直接与明文比较，没有任何加密，直接跟进去就看到了
    return MessageBoxA(hDlg, aIncorrectPassw, Caption, 0x10u);
  MessageBoxA(hDlg, Text, Caption, 0x40u);
  return EndDialog(hDlg, 0);
}
```



## Easy-Keygen

> 我才发现这种解密脚本应该叫`Keygen`...

main函数处理完之后如下

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  signed int v3; // ebp
  int i; // esi
  char v6[4]; // [esp+Ch] [ebp-130h]
  char input[100]; // [esp+10h] [ebp-12Ch] BYREF
  char Buffer[100]; // [esp+74h] [ebp-C8h] BYREF
  __int16 v9; // [esp+139h] [ebp-3h]
  char v10; // [esp+13Bh] [ebp-1h]

  input[0] = 0;
  Buffer[0] = 0;
  memset(&input[1], 0, 0x60u);
  *(_WORD *)&input[97] = 0;
  input[99] = 0;
  memset(&Buffer[1], 0, 0xC4u);
  v9 = 0;
  v10 = 0;
  v6[0] = 16;
  v6[1] = 32;
  v6[2] = 48;
  IO_PUTS(aInputName);
  scanf("%s", input); // name
  v3 = 0;
  for ( i = 0; v3 < (int)strlen(input); ++i )
  {
    if ( i >= 3 ) // 这地方相当于取模
      i = 0;
    sprintf(Buffer, "%s%02X", Buffer, input[v3++] ^ v6[i]); // 我们可以选择动调查看Buffer的值，就会比较清晰了
  }
  memset(input, 0, sizeof(input));
  IO_PUTS(aInputSerial);
  scanf("%s", input); // serial，它用同一个变量存储了name和serial，即input
  if ( !strcmp(input, Buffer) ) // 可以看到，serial和name是相关的，通过name生成serial
    IO_PUTS(aCorrect);
  else
    IO_PUTS(aWrong);
  return 0;
}
```

调试器选择IDA

![keygen.png](https://i.loli.net/2021/07/18/xTEJWKoyFa4gfuw.png)

调的时候观察Buffer，可以发现Buffer存的是

```c
hex(ord(input[i]) ^ v6[i])][2:]
```

keygen如下

```python
base = [16, 32, 48]
num = []
serial = '5B134977135E7D13' # serial已知的
for i in range(len(serial)//2):
    num.append(base[i%3])

# print(num)
index = 0
for i in range(0, 16, 2):
    each_hex = int(serial[i]+serial[i+1], 16)
    print(chr(each_hex^num[index]), end="")
    index += 1
    
# K3yg3nm3
```



## Easy-Unpack

> 题目要求OEP就是flag

![1](https://i.loli.net/2021/07/18/FjnMg4DBKRICfuO.png)

OD看入口特征，先push再call。我们先单步执行一下

![2](https://i.loli.net/2021/07/18/oJBCwj2LuU6fiPl.png)

发现ESP变了，我们把ESP的值那个位置设一个硬件断点，我们可以在`调试——硬件断点`窗口看到硬件断点。接下来我们F9将程序高速运行起来，接着单步执行，这个单步的过程会比较长，我们需要耐心一点

跟到这里

![3](https://i.loli.net/2021/07/18/BTAN43gfWncGeY9.png)

我们删除分析，脱壳，发现这里就是OEP了

## Music Player

### First-patch

题目README告诉我们这个播放器只能播放一分钟，我们要做的就是让它播放超过一分钟。直接想到的思路就是修改跳转，所以我们需要先找到跳转的位置，进一步的，我们就需要找到失败的特征。先随便找个`.mp3`文件，播放到底，发现弹窗了，弹窗内容有个`1? ???`，所以我们就在OD里面搜索字符串，希望能定位到一个位置

![4](https://i.loli.net/2021/07/18/wCzbi3u5I1g8MGs.png)

跟过去看一眼，下面不远处就有`MsgBox`函数，推测就是这里了，但我们还需要找到这样一个结构：比较的指令紧跟一个跳转，并且跳转能被patch成直接跳过弹窗

![5](https://i.loli.net/2021/07/18/d7nDw9Mh5IlvJPR.png)

往上翻一翻，果然找到了

![6](https://i.loli.net/2021/07/18/LCTNxPqWFRGySJ9.png)

直接暴力patch成jmp，跳过跳转。但这题还是比较坑的，因为只patch这一处的话，等到一分钟以后会弹出另一个runtime error的弹窗。所以我们找另一处，这一处涉及到一个调试的技巧。

### Another-patch

我们在上一处patch的jmp那里断一个断点，然后让程序运行到这个位置，一开始发现只能播放一秒就停下了，但如果我们继续往下跟的话，会发现这整个jmp处在一个巨大的循环里面。所以我们只需要一直F9，程序会始终断在断点处，每次播放一秒。

**PS：**至于这里为什么要用一个循环来实现还在跟，等到深入分析一下再来更新，如果有师傅看到也可以在下面留言。

我们让播放器播放到59秒，然后往下单步执行，发现一个比较远的跳转，下面不远处有MsgBox，于是Patch。然后播放到一分钟发现标题处变成了flag

## Replace

这题运行起来挺像MFC的，有窗口控件，用`ResourcceHacker`也能搜到控件ID为`1002`。根据这个ID的十六进制`3EAh`定位到关键代码处，或者根据字符串`Correct!`来定位会比较直观。先不着急调试，先看看大概的跳转流程：

![7](https://i.loli.net/2021/07/18/ogQlVFHLEeAfwZN.png)

`loc_401071`刚好跳过了正确的位置使得函数retn，正确的位置似乎被独立出来了。我们x一下`loc_401071`，发现有一个跳转：

![8](https://i.loli.net/2021/07/18/s5jDKCJNrymbozB.png)

然后上OD，定位到同样的位置，并且在获取输入的位置下一个断点。

![9.png](https://i.loli.net/2021/07/18/Og7WLVvDMuyIa3A.png)

由于一般返回值会存在eax里面，我们获取输入之后不断观察eax值的变化，先输入0观察变化。下面一行汇编把eax赋值给一个变量，紧跟着call了一个函数。过了这个函数之后eax加1，然后往后跟，可以猜测这个函数还有一个功能：改变那个变量的值

![10](https://i.loli.net/2021/07/18/6WzdXyaOqVPwfJv.png)

到这里发现eax的值为`0x601605CB`。然后我们换一个输入，比如10，会发现跟到这里之后eax变成`0x601605D5`，是不是发现了什么？

```
0x601605CB - 0 == 0x601605CB
0x601605D5 - 10 == 0x601605CB
```

你可以继续试，但都是这样的规律，这个暂时没用，往下跟，发现call了`0x004066F`的位置，里面的内容比较有意思

![11](https://i.loli.net/2021/07/18/5WQNK1w6bJZTh29.png)

它把eax的值指向的内容搞成了nop。结合IDA的分析，我们很容易想到构造eax的值，使得它等于正确代码块上面的那一处jmp的地址即可。于是输入为

`0x100401071-0x601605CB`

至于这里为什么要是把地址的最高位写成1，涉及到补码的问题，在4字节的地址上，这个符号位会被截断（还在验证，希望师傅能指出错误

## Ransomeware

![12.png](https://i.loli.net/2021/07/18/3sPTRMnFLdWjpCe.png)

IDA加载就发现了这个问题了，我们进汇编窗口发现一大坨花指令，也难怪加载不出来了，其中整个`sub_401000`都是花指令，这个函数整个就是一个空函数，然后同样的花指令在`main`函数也有一坨，去掉以后看到`main`函数了

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  unsigned int input_len; // kr00_4
  FILE *v5; // [esp+1Ch] [ebp-14h]
  unsigned int file_len; // [esp+20h] [ebp-10h]
  int index; // [esp+28h] [ebp-8h]
  unsigned int i; // [esp+28h] [ebp-8h]
  unsigned int j; // [esp+28h] [ebp-8h]
  FILE *Stream; // [esp+2Ch] [ebp-4h]

  NtCurrentTeb()->ProcessEnvironmentBlock->BeingDebugged = 68;
  CreateThread(0, 0, StartAddress, 0, 0, 0);
  printf(
    "唱绰 唱慧仇捞促!\n"
    "唱绰 概快 唱悔扁 锭巩俊 呈狼 颇老阑 鞠龋拳沁促!\n"
    "呈狼 颇老阑 汗备窍绊 酵促搁 5玫撅 崔矾甫 涝陛窍绊 罐篮 虐蔼栏肺 颇老阑 汗备秦扼!\n"
    "\n");
  printf("Key : ");
  empty();
  scanf("%s", input);
  input_len = strlen(input);
  empty();
  index = 0;
  Stream = fopen("file", "rb");
  empty();
  if ( !Stream )                                // 打开失败
  {
    empty();
    printf("\n\n\n颇老阑 茫阑荐 绝促!\n");
    empty();
    exit(0);
  }
  fseek(Stream, 0, 2);
  empty();
  file_len = ftell(Stream);
  empty();
  rewind(Stream);
  empty();
  while ( !feof(Stream) )
  {
    empty();
    space_bak[index] = fgetc(Stream);
    empty();
    ++index;
    empty();
  }
  empty();
  for ( i = 0; i < file_len; ++i )
  {
    space_bak[i] ^= input[i % input_len];
    empty();
    space_bak[i] = ~space_bak[i];
    empty();
  }
  fclose(Stream);

  empty();
  v5 = fopen("file", "wb");
  empty();
  empty();
  for ( j = 0; j < file_len; ++j )
  {
    fputc(space_bak[j], v5);
    empty();
  }
  printf(
    "\n"
    "颇老阑 汗备沁促!\n"
    "唱绰 各矫 唱悔瘤父 距加篮 瘤虐绰 荤唱捞促!\n"
    "蝶扼辑 呈啊 唱俊霸 捣阑 玲绊, 棵官弗 虐蔼阑 罐疽促搁 颇老篮 沥惑拳 登绢 乐阑 巴捞促!\n"
    "窍瘤父 父距 肋给等 虐甫 持菌促搁 唱绰 酒林酒林 唱悔扁 锭巩俊 呈狼 颇老篮 肚 噶啊龙 巴捞促!");
  empty();
  return getch();
}
```

我们结合提示，可以猜测这个file文件就是一个被加密过后的exe。**PS：**这一步个人觉得是最考验脑洞的地方了。联系到PE文件结构，一般在004Eh的位置会有一句固定的字符串`This program cannot be run in DOS mode`。这题也没别的信息了，大概率就是利用这个点，然后我们可以拿到异或的key，keygen如下：

```python
# 004e~0073
pe_dos = b'This program cannot be run in DOS mode'
file_r = [199, 242, 226, 255, 175, 227, 236, 233, 251, 229, 251, 225, 172, 240, 251, 229, 226, 224, 231, 190, 228, 249, 183, 232, 249, 226, 179, 243, 229, 172, 203, 220, 205, 166, 241, 248, 254, 233]

key = ''
for i in range(len(pe_dos)):
    key += chr(pe_dos[i]^file_r[i]^0xff)

print(key)
# letsplaychess letsplaychess letsplayches
```

然后还原文件：

```python
enc_data = open('./file', 'rb').read()
origin_file = open('./origin_file.exe', 'wb')
key = b'letsplaychess'

origin_data = bytes([(enc_data[i] ^ 0xff)^key[i % len(key)] for i in range(len(enc_data))])
origin_file.write(origin_data)
```

再拖一下IDA，搜字符串就有明文的flag了

## Easy-ELF

比较简单也比较常规，先不写了。

## CSHOP

C#逆向，我们有两种方法

### 静态逆向 ×

用Rider载入可以定位到关键的代码，但是flag是乱序的，那么长总不可能直接试出来吧，也太没技术含量了（，但算法应该也能逆出来，但它加了混淆，先挂个坑——去混淆的工具+逆算法

### 改资源 √

我们要用到一个SpyLite的工具搜窗口句柄

![13.png](https://i.loli.net/2021/07/18/XAbiphuwIB9qOeQ.png)

打开子窗口列表，容易看到`BUTTON`

![14](https://i.loli.net/2021/07/18/oeaVHRq9CNQ2FLz.png)

把BUTTON最大化之后点一下，再把最大化关掉，就可以看到flag了

![15](https://i.loli.net/2021/07/18/hpez1gy3kFmqUWX.png)

## HateIntel

关键函数：

```c
int sub_2224()
{
  char input[80]; // [sp+4h] [bp-5Ch] BYREF
  int con; // [sp+54h] [bp-Ch]
  int input_len; // [sp+58h] [bp-8h]
  int i; // [sp+5Ch] [bp-4h]

  con = 4;
  printf("Input key : ");
  scanf("%s", input);
  input_len = strlen(input);
  enc(input, con);
  for ( i = 0; i < input_len; ++i )
  {
    if ( input[i] != aim[i] )
    {
      puts("Wrong Key! ");
      return 0;
    }
  }
  puts("Correct Key! ");
  return 0;
}
```

拐进`enc`里面：

```c
// _con = 4
void __fastcall enc(char *_input, int _con)
{
  int i; // [sp+8h] [bp-Ch]
  int j; // [sp+Ch] [bp-8h]

  for ( i = 0; i < _con; ++i ) // 循环四次，并且每一位的加密是相对独立的，所以我们可以理解为直接对一位加密四次，再转到下一位
  {
    for ( j = 0; (int)strlen(_input) > j; ++j )
      _input[j] = sub_2494(_input[j], 1);
  }
}
```

`sub_2494`函数：

```c
// a = 1
char __fastcall sub_2494(unsigned __int8 ch, int a)
{
  int tmp; // [sp+8h] [bp-8h]
  int i; // [sp+Ch] [bp-4h]

  tmp = ch;
  for ( i = 0; i < a; ++i ) // 只有一轮
  {
    tmp *= 2;
    if ( (tmp & 0x100) != 0 )
      tmp |= 1u;
  }
  return tmp;
}
```

keygen爆破脚本如下：

```python
aim = [ 68, 246, 245, 87, 245, 198, 150, 182, 86, 245, 20, 37, 212, 245, 150, 230, 55, 71, 39, 87, 54, 71, 150, 3, 230, 243,
163, 146]
flag = ''
for index in range(len(aim)):
    for i in range(256):
        tmp = i
        for l in range(4):
            tmp *= 2
            if (tmp & 0x100) != 0:
                tmp |= 1
        if tmp%256 == aim[index]:
            flag += chr(i)

print(flag)
# Do_u_like_ARM_instructi0n?:)
```

## Position

因为缺少 dll，linux 下用 wine 也跑不起来，翻函数表加字符串，找到关键函数，发现就是一个比较。keygen 如下：

```python
from itertools import product

judge = lambda result, target, index : result[index] == target[index]
serial = [int(i) for i in '7687677776']
tail = ord('p')

for i, j in product(range(ord('a'), ord('z') + 1, 1), repeat=2):
    name_0_mess = [(i & 1) + 5, ((i & 8) != 0) + 5, ((i & 2) != 0) + 5, ((i & 4) != 0) + 5, ((i & 0x10) != 0) + 5]
    name_1_mess = [((j & 4) != 0) + 1, ((j & 8) != 0) + 1, ((j & 0x10) != 0) + 1, (j & 1) + 1, ((j & 2) != 0) + 1]
    result = [x + y for x, y in zip(name_0_mess, name_1_mess)]
    is_ok = True
    for idx in range(5):
        if not judge(result, serial, idx):
            is_ok = False

    if is_ok:
        print(chr(i) + chr(j))

print()
for i in range(ord('a'), ord('z') + 1, 1):
    name_2_mess = [(i & 1) + 5, ((i & 8) != 0) + 5, ((i & 2) != 0) + 5, ((i & 4) != 0) + 5, ((i & 0x10) != 0) + 5]
    name_3_mess = [((tail & 4) != 0) + 1, ((tail & 8) != 0) + 1, ((tail & 0x10) != 0) + 1, (tail & 1) + 1, ((tail & 2) != 0) + 1]
    result = [x + y for x, y in zip(name_2_mess, name_3_mess)]
    is_ok = True
    for idx in range(5):
        if not judge(result, serial[5:], idx):
            is_ok = False

    if is_ok:
        print(chr(i) + chr(tail))
```

发现前两个是有多解，最后两个只有 mp 这一种情况，一个个试一下就行
