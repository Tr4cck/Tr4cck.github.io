---
title: 「TetCTF2021」Rev 部分Writeup
date: 2022-01-13 23:31:09
tags:
- Rev
categories:
- TECHNOLOGY
---

质量很高，感谢越南黑客

## magicbox
### Background
题目使用**或非指令**构建了一个单指令虚拟机，因为一个或非就能构成全功能集合，可以实现与或非，进而实现单比特全加器，进而实现多比特全加器，进而实现各种数学运算；条件分支也很好实现。虚拟机字长2字节，内存模型为线性65536字节，即刚好可以在一条指令内直接寻址完所有的地址空间。
核心逻辑如下：
```c
int sub_401000()
{
  unsigned __int16 *mem; // ebx
  int result; // eax
  int PC; // edx
  int v3; // ecx
  int v4; // esi
  int v5; // edi

  mem = (unsigned __int16 *)lpAddress;
  for ( result = 1; *mem != 0xFFFF; result = 1 )
  {
    if ( mem[4] == 1 )
    {
      mem[4] = 0;
      printf("%c", mem[3]);
      mem = (unsigned __int16 *)lpAddress;
    }
    if ( mem[6] == 1 )
    {
      mem[6] = 0;
      scanf("%c", mem + 5);
      mem = (unsigned __int16 *)lpAddress;
    }
    PC = *mem;
    v3 = mem[PC + 1];
    v4 = mem[PC];
    v5 = mem[PC + 2];
    *mem = PC + 3;
    LOWORD(v3) = ~(mem[v4] | mem[v3]);
    mem[v5] = v3;
    mem[1] = __ROL2__(v3, 1);                   // SHIFT_REG
  }
  return result;
}
```
### Emulator and logger
当然内存呢，题目已经帮我们布置好了，但是这样的单指令虚拟机，很难通过这样看就得出结论。如果有一定逆向虚拟机经验的话，其实有一个大家比较公认的一般思路，就是去**仿写虚拟机，然后源码插桩打 log**。比如这道题来看的话，大概就是写成这样：
```c
#include <stdio.h>
#include <stdint.h>

#define __ROL2__(a, b) ((a << b) | (a >> (16 - b))) & 0xffff

uint16_t mem[] = {...}; // 省略

// mem[7] fetch constant
// mem[3] output
// mem[4] print interface
// mem[5] input
// mem[6] scanf interface
// mem[0] PC
// mem[0x1a23] ~ mem[0x1a3c] input buffer len = 26
// mem[0x1a0e] ~             constant buffer

int main() {
    uint16_t *p_mem = mem;
    uint16_t PC, a, b, f, r;
    while (*p_mem != 0xffff) {
        if (p_mem[4] == 1) {
            p_mem[4] = 0;
            printf("%c", p_mem[3]);
        }
        if (p_mem[6] == 1) {
            p_mem[6] = 0;
            scanf("%c", p_mem + 5);
        }
        PC = *p_mem;
        a = p_mem[PC + 1];
        b = p_mem[PC];
        r = p_mem[PC + 2];
        printf("%#x:\t\tmem[%#x] = ~(mem[%#x] | mem[%#x])", PC, r, b, a);
        *p_mem = PC + 3;
        f = ~(p_mem[b] | p_mem[a]);
        printf("\t\tRES: %#x", f);
        p_mem[r] = f;
        p_mem[1] = __ROL2__(f, 1); // SHIFT_REG
        printf("\t\tSHIFT_REG: %#x\n", p_mem[1]);
    }
    return 0;
}
```
那么编译并运行之后，我们会得到一堆像这样的 console log：
```
0x12c:		mem[0x7] = ~(mem[0x1a40] | mem[0x1a40])		RES: 0xffaf		SHIFT_REG: 0xff5f
0x12f:		mem[0x3] = ~(mem[0x7] | mem[0x7])		RES: 0x50		SHIFT_REG: 0xa0
0x132:		mem[0x7] = ~(mem[0x138] | mem[0x138])		RES: 0xfec5		SHIFT_REG: 0xfd8b
0x135:		mem[0] = ~(mem[0x7] | mem[0x7])		RES: 0x13a		SHIFT_REG: 0x274
0x13a:		mem[0x7] = ~(mem[0x139] | mem[0x139])		RES: 0xfffe		SHIFT_REG: 0xfffd
0x13d:		mem[0x4] = ~(mem[0x7] | mem[0x7])		RES: 0x1		SHIFT_REG: 0x2
P0x140:		mem[0x7] = ~(mem[0x1a3f] | mem[0x1a3f])		RES: 0xff9e		SHIFT_REG: 0xff3d
```
那么我们其实就可以开始分析，比如 `mem[7]`、`mem[3]` 之类，的作用了，分析结果写在上面的源码注释里。
读了一点 log 我们就可以发现，这个虚拟机是一次性把输入全读进去，然后开始验证，验证用的是 `mem[0x1ee]` 这个标志位，并且即使中间部分验证失败了，也不会立马退出，而是继续把所有部分都验证完，这很好的避免了基于指令数的侧信道攻击。
### Idea 1
到这里我们很容易就想到一种思路：
```
0x3f7:		mem[0x7] = ~(mem[0x3fd] | mem[0x3fd])		RES: 0xfbff		SHIFT_REG: 0xf7ff
0x3fa:		mem[0] = ~(mem[0x7] | mem[0x7])		RES: 0x400		SHIFT_REG: 0x800
0x400:		mem[0x3fe] = ~(mem[0x1a23] | mem[0x1a23])		RES: 0xff9e		SHIFT_REG: 0xff3d
0x403:		mem[0x3ff] = ~(mem[0x1a0e] | mem[0x1a0e])		RES: 0xffa8		SHIFT_REG: 0xff51
0x406:		mem[0x7] = ~(mem[0x40c] | mem[0x40c])		RES: 0xfbf0		SHIFT_REG: 0xf7e1
0x409:		mem[0] = ~(mem[0x7] | mem[0x7])		RES: 0x40f		SHIFT_REG: 0x81e
0x40f:		mem[0x40d] = ~(mem[0x1a23] | mem[0x1a23])		RES: 0xff9e		SHIFT_REG: 0xff3d
0x412:		mem[0x40e] = ~(mem[0x3ff] | mem[0x3ff])		RES: 0x57		SHIFT_REG: 0xae
0x415:		mem[0x7] = ~(mem[0x40d] | mem[0x40e])		RES: 0x20		SHIFT_REG: 0x40
0x418:		mem[0x7] = ~(mem[0x7] | mem[0x7])		RES: 0xffdf		SHIFT_REG: 0xffbf
0x41b:		mem[0x3ff] = ~(mem[0x7] | mem[0x7])		RES: 0x20		SHIFT_REG: 0x40
0x41e:		mem[0x7] = ~(mem[0x424] | mem[0x424])		RES: 0xfbd8		SHIFT_REG: 0xf7b1
0x421:		mem[0] = ~(mem[0x7] | mem[0x7])		RES: 0x427		SHIFT_REG: 0x84e
0x427:		mem[0x425] = ~(mem[0x1a0e] | mem[0x1a0e])		RES: 0xffa8		SHIFT_REG: 0xff51
0x42a:		mem[0x426] = ~(mem[0x3fe] | mem[0x3fe])		RES: 0x61		SHIFT_REG: 0xc2
0x42d:		mem[0x7] = ~(mem[0x425] | mem[0x426])		RES: 0x16		SHIFT_REG: 0x2c
0x430:		mem[0x7] = ~(mem[0x7] | mem[0x7])		RES: 0xffe9		SHIFT_REG: 0xffd3
0x433:		mem[0x3fe] = ~(mem[0x7] | mem[0x7])		RES: 0x16		SHIFT_REG: 0x2c
0x436:		mem[0x7] = ~(mem[0x3fe] | mem[0x3ff])		RES: 0xffc9		SHIFT_REG: 0xff93
0x439:		mem[0x1a0d] = ~(mem[0x7] | mem[0x7])		RES: 0x36		SHIFT_REG: 0x6c
0x43c:		mem[0x7] = ~(mem[0x1a0d] | mem[0x1ee])		RES: 0xffc9		SHIFT_REG: 0xff93
0x43f:		mem[0x1ee] = ~(mem[0x7] | mem[0x7])		RES: 0x36		SHIFT_REG: 0x6c
# ~(~0x57 | input[0]) | ~(~input[0] | 0x57) | mem[0x1ee]
```
像这样，硬着头皮分析，然后就会发现其实这么一大段的或非逻辑，其实就是一个简单的异或和或的组合。这么做肯定是能做出来的，而且你必然相信你能做出来，就是堆时间问题。但是这么做题太无聊了，所以我们选择想一个算法，让计算机帮我们解决这个繁琐的问题。于是就有了思路二。

### Idea 2
其实就是枚举每个字符，但是肯定不可能只是这样，我们需要用一个小小的回溯思想，把再加点限制条件。其实就是把 `mem[0x1ee]` 这个位置什么时候不是0了，我们就把那个字符排除掉，但是因为刚才说了，题目会把输入一股脑全输入进去，然后检验，所以即使你这一个字符是对的，等它跑完也不是0了，那么你就不能立马确定这个字符是不是对的，从而会影响到时间复杂度。解决方案是这样：我们再设置一个独立于虚拟机运行的变量 `guard`，让它去记录我们这个字符跑到了哪一个地址，那么一旦我们枚举到了正确的字符，即使后面 `mem[0x1ee]` 不是0，但由于它对了，所以 `guard` 肯定会大于刚才的 `guard`，此时我们就认为这个字符正确，然后跳出虚拟机，继续爆破下一个字符。脚本实现如下：
```python
from mem import opcode
import string

flag_len = 26
flag = 'WE1' # 先逆出的3个字符，虽说没啥用，但还是写上吧
isok = False

while True:
    print(flag)
    guard = -1
    for ch in string.printable:
        if isok:
            isok = False
            break

        tryflag = (flag + ch).ljust(26)
        p_mem = opcode[:]
        # vmrun
        counter = 0
        while p_mem[0] != 0xffff:
            counter += 1
            if p_mem[0x1ee] != 0:
                if guard != -1 and guard < counter:
                    flag += ch
                    isok = True
                    break
                else:
                    guard = counter
                    break

            if p_mem[6] == 1:
                p_mem[6] = 0
                p_mem[5] = ord(tryflag[0])
                tryflag = tryflag[1:]

            pc = p_mem[0]
            a = p_mem[pc + 1]
            b = p_mem[pc]
            r = p_mem[pc + 2]

            p_mem[0] = pc + 3
            f = ~(p_mem[a] | p_mem[b]) & 0xffff
            p_mem[r] = f
            p_mem[1] = (f >> 15) & 1 | ((f & 0x7fff) << 1)

        if p_mem[0x1ee] == 0:
            flag += ch
            isok = True
            break
    
    if len(flag) == flag_len:
        break

print(flag)
```
跑一下看看，emmmm好像有点问题，它卡在了一个地方：
![](https://s2.loli.net/2022/04/08/7ZaXtCPI98Nl246.png)
不知道为什么，直接去手动逆一下那一位是 `@`。最终keygen：
```python
from mem import opcode
import string

...

padding = '@'

while True:
    if len(flag) == 15:
        flag += padding

    ...

print(flag)
```

## hidden


## crackmepls
咕咕咕
