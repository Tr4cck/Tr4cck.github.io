---
title: 「NCTF2021」Rev 部分题解 by 0xbl4cb12d
mathjax: true
toc: true
date: 2021-11-29 23:25:13
tags: Rev
categories: TECHNOLOGY
---


**URL**: https://nctf.h4ck.fun/
**Teammate**: deebato / corle / wanan / track
**Start Time**: 11.27 9:00
**End Time**: 11.28 21:00

## Hello せかい | DONE

拖进IDA看到明文flag

## Shadowbringer | DONE

c++实现的两次base64加密
两次的表互为逆序串，且padding为由=变为!

```c
#include <stdio.h>
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <string.h>

char table[] = "#$%&'()*+,-.s0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[h]^_`ab";
char padding_char = '!';

int findchr(char *array, char ch) {
        for(int i = 0; i < strlen(array); i++) {
                if(array[i] == ch) {
                        return i;
                }
        }
        return 0;
}

void b64encode(char *src, char *result) {
        int fill_byte = 0, result_length = 0, index = 0, data_length = strlen(src);
        // bool full = true;
        fill_byte = (3-data_length)%3;
        for(int k = 0; k < fill_byte; k++) {
                src[data_length + k] = '\0';
        }
        int j = 0;
        for(int i = 0; i < data_length; i += 3) {
                index = src[i] >> 2;
                result[j++] = table[index];
                index = ((src[i] & 3) << 4) + (src[i+1] >> 4);
                result[j++] =  table[index];
                index = ((src[i+1]&15) << 2) + (src[i+2] >> 6);
                result[j++] = table[index];
                index = (src[i+2]&0x3f);
                result[j++] = table[index];
        }
        result_length = strlen(result);
        for(int k = 0; k < fill_byte; k++) {
                result[result_length - 1 - k] = padding_char;
        }
}

void b64decode(char *src, char *result) {
        int base_len = strlen(src);
        int j = 0;

        for(int i = 0; i < base_len; i++) {
                if (src[i] == padding_char)
                        src[i] = 'A';
        }
        for(int i = 0; i <base_len; i += 4) {
                result[j++] = (findchr(table, src[i]) << 2) + (findchr(table, src[i+1]) >> 4);
                result[j++] = (findchr(table, src[i+1]) << 4) + (findchr(table, src[i+2]) >> 2);
                result[j++] = ((findchr(table, src[i + 2]) & 3) << 6) + (findchr(table, src[i+3]));
        }
}

int main() {
        char de_words[100] = {'\0'};
        char en_words[100] = "6G074JP+s)WV:Z+T<&(Q18`Ks)WV:Y4hs9[h:YCS?&0`!";
        b64decode(en_words, de_words);
        printf("encode:%s\n", en_words);
        printf("decode:%s\n", de_words);
        return 0;
}
```

## dog | ALMOST

整个程序的运行流程如下：

- 首先调用局部线程存储的回调函数 `TlsCallback_0` ，第一个 if 判断该函数的被调用次数，从此处可以看出需要被调用 3 次后才能进 if ，进去之后的逻辑为改变了一个函数指针的指向，并且对指向的地址处进行了代码自修改的解密，具体类似 TEA 算法的解密过程。*在代码段有一段未定义的 TEA 加密函数，合理猜测是出题人在上题的时候忘了删掉这个函数*。
- 接着进入 `start` 函数，利用 `initterm` 函数进行初始化。调试不难发现，作为参数的某个函数指针上存放着反调试代码，patch 掉之后可以正常完成初始化过程。然后进入 `main` 函数
- 主函数部分完成用户输入之后，新开了一个线程调用 `StartAddress` 进行变量赋值，经过调试不难看出，在新开线程的同时，我们之前看到的 `TlsCallback_0` 又被调用了一次，但是这只是第二次。第三次出现在关闭线程之前，算上之前的，这已经是第三次了，此时我们可以通过 if 的次数判断，并使用一些反反调试的基本手法（例如修改寄存器和标志位的值从而通过类似 `IsDebuggerPresent` 函数，还有 `CheckRemoteDebugger` 函数的判断），SMC 过后控制流回到 main 函数，进入真正的判断函数，不妨 dump下来分析：

```c
void __cdecl realJudge(const char *input)
{
  signed int v1; // [esp+0h] [ebp-98h]
  unsigned int strlen_input; // [esp+10h] [ebp-88h]
  signed int v3; // [esp+1Ch] [ebp-7Ch]
  int v4; // [esp+2Ch] [ebp-6Ch]
  int v5; // [esp+2Ch] [ebp-6Ch]
  char v6; // [esp+32h] [ebp-66h]
  signed int Size; // [esp+34h] [ebp-64h]
  unsigned int v8; // [esp+38h] [ebp-60h]
  int k; // [esp+38h] [ebp-60h]
  unsigned __int8 *v10; // [esp+3Ch] [ebp-5Ch]
  int i; // [esp+40h] [ebp-58h]
  signed int j; // [esp+40h] [ebp-58h]
  signed int l; // [esp+40h] [ebp-58h]
  signed int m; // [esp+40h] [ebp-58h]
  signed int n; // [esp+40h] [ebp-58h]
  char v16[62]; // [esp+44h] [ebp-54h]
  int v17; // [esp+82h] [ebp-16h]
  int v18; // [esp+86h] [ebp-12h]
  int v19; // [esp+8Ah] [ebp-Eh]
  int v20; // [esp+8Eh] [ebp-Ah]
  __int16 v21; // [esp+92h] [ebp-6h]

  strlen_input = strlen(input);                 // should be 42
  Size = 146 * strlen_input / 0x64 + 1;
  v3 = 0;
  v10 = (unsigned __int8 *)malloc(Size);
  v16[0] = 82;
  v16[1] = 195;
  v16[2] = 26;
  v16[3] = 224;
  v16[4] = 22;
  v16[5] = 93;
  v16[6] = 94;
  v16[7] = -30;
  v16[8] = 103;
  v16[9] = 31;
  v16[10] = 31;
  v16[11] = 6;
  v16[12] = 6;
  v16[13] = 31;
  v16[14] = 23;
  v16[15] = 6;
  v16[16] = 15;
  v16[17] = -7;
  v16[18] = 6;
  v16[19] = 103;
  v16[20] = 88;
  v16[21] = -78;
  v16[22] = -30;
  v16[23] = -116;
  v16[24] = 15;
  v16[25] = 42;
  v16[26] = 6;
  v16[27] = -119;
  v16[28] = -49;
  v16[29] = 42;
  v16[30] = 6;
  v16[31] = 31;
  v16[32] = -104;
  v16[33] = 26;
  v16[34] = 62;
  v16[35] = 23;
  v16[36] = 103;
  v16[37] = 31;
  v16[38] = -9;
  v16[39] = 58;
  v16[40] = 68;
  v16[41] = -61;
  v16[42] = 22;
  v16[43] = 51;
  v16[44] = 105;
  v16[45] = 26;
  v16[46] = 117;
  v16[47] = 22;
  v16[48] = 62;
  v16[49] = 23;
  v16[50] = -43;
  v16[51] = 105;
  v16[52] = 122;
  v16[53] = 27;
  v16[54] = 68;
  v16[55] = 68;
  v16[56] = 62;
  v16[57] = 103;
  v16[58] = -9;
  v16[59] = -119;
  v16[60] = 103;
  v16[61] = -61;
  v17 = 0;
  v18 = 0;
  v19 = 0;
  v20 = 0;
  v21 = 0;
  memset(v10, 0, Size);
  v8 = 0;
  for ( i = 0; i < 256; ++i )                   // swap
  {
    v6 = byte_405018[i];
    byte_405018[i] = byte_405018[(i + *((unsigned __int8 *)&dword_405168 + i % 4)) % 256];
    byte_405018[(i + *((unsigned __int8 *)&dword_405168 + i % 4)) % 256] = v6;
  }
  while ( v8 < strlen(input) )
  {
    v4 = input[v8];
    for ( j = 146 * strlen_input / 0x64; ; --j ) // a re-traversal
    {
      v5 = v4 + (v10[j] << 8);
      v10[j] = v5 % 47;
      v4 = v5 / 47;
      if ( j < v3 )
        v3 = j;
      if ( !v4 && j <= v3 )
        break;
    }
    ++v8;
  }
  for ( k = 0; !v10[k]; ++k )
    ;
  for ( l = 0; l < Size; ++l )
    v10[l] = byte_405118[v10[k++]];
  while ( l < Size )
    v10[l++] = 0;
  v1 = strlen((const char *)v10);
  for ( m = 0; m < v1; ++m )
    v10[m] ^= byte_405018[v10[m]];
  for ( n = 0; n < v1; ++n )
  {
    if ( v10[n] != (unsigned __int8)v16[n] )
    {
      printf("Wrong!\n");
      exit(0);
    }
  }
  printf("Right!\n");
}
```

相关数据可以在调试时 dump 下来。比赛时的思路是直接爆破，但是至少也得把两次表替换之前的 v10 给逆出来，但是当时脑子比较乱，没有充分考虑多解的情况，加上 IDA 出现了神必问题，导致我一直怀疑我的数据是不对的。

```python
xor_table = [
    0x21, 0x43, 0x65, 0x87, 0x09, 0x21, 0x43, 0x65, 0xA2, 0x9B,
    0xF4, 0xDF, 0xAC, 0x7C, 0xA1, 0xC6, 0x16, 0xD0, 0x0F, 0xDD,
    0xDC, 0x73, 0xC5, 0x6B, 0xD1, 0x96, 0x47, 0xC2, 0x26, 0x67,
    0x4E, 0x41, 0x82, 0x20, 0x56, 0x9A, 0x6E, 0x33, 0x92, 0x88,
    0x29, 0xB5, 0xB4, 0x71, 0xA9, 0xCE, 0xC3, 0x34, 0x50, 0x59,
    0xBF, 0x2D, 0x57, 0x22, 0xA6, 0x30, 0x04, 0xB2, 0xCD, 0x36,
    0xD5, 0x68, 0x4D, 0x5B, 0x45, 0x9E, 0x85, 0xCF, 0x9D, 0xCC,
    0x61, 0x78, 0x32, 0x76, 0x31, 0xE3, 0x80, 0xAD, 0x39, 0x4F,
    0xFA, 0x72, 0x83, 0x4C, 0x86, 0x60, 0xB7, 0xD7, 0x63, 0x0C,
    0x44, 0x35, 0xB3, 0x7B, 0x19, 0xD4, 0x69, 0x08, 0x0B, 0x1F,
    0x3D, 0x11, 0x79, 0xD3, 0xEE, 0x93, 0x42, 0xDE, 0x23, 0x3B,
    0x5D, 0x8D, 0xA5, 0x77, 0x5F, 0x58, 0xDB, 0x97, 0xF6, 0x7A,
    0x18, 0x52, 0x15, 0x74, 0x25, 0x62, 0x2C, 0x05, 0xE8, 0x0D,
    0x98, 0x2A, 0x43, 0xE2, 0xEF, 0x48, 0x87, 0x49, 0x1C, 0xCA,
    0x2B, 0xA7, 0x8A, 0x09, 0x81, 0xE7, 0x53, 0xAA, 0xFF, 0x6F,
    0x8E, 0x91, 0xF1, 0xF0, 0xA4, 0x46, 0x3A, 0x7D, 0x54, 0xEB,
    0x2F, 0xC1, 0xC0, 0x0E, 0xBD, 0xE1, 0x6C, 0x64, 0xBE, 0xE4,
    0x02, 0x3C, 0x5A, 0xA8, 0x9F, 0x37, 0xAF, 0xA0, 0x13, 0xED,
    0x1B, 0xEC, 0x8B, 0x3E, 0x7E, 0x27, 0x99, 0x75, 0xAB, 0xFE,
    0xD9, 0x3F, 0xF3, 0xEA, 0x70, 0xF7, 0x95, 0xBA, 0x1D, 0x40,
    0xB0, 0xF9, 0xE5, 0xF8, 0x06, 0xBC, 0xB6, 0x03, 0xC9, 0x10,
    0x9C, 0x2E, 0x89, 0x5C, 0x7F, 0xB1, 0x1A, 0xD6, 0x90, 0xAE,
    0xDA, 0xE6, 0x5E, 0xB9, 0x84, 0xE9, 0x55, 0xBB, 0xC7, 0x0A,
    0xE0, 0x66, 0xF2, 0xD8, 0xCB, 0x00, 0x12, 0xB8, 0x17, 0x94,
    0x6A, 0x4A, 0x01, 0x24, 0x14, 0x51, 0x07, 0x65, 0x21, 0xC8,
    0x38, 0xFD, 0x8F, 0xC4, 0xF5, 0xFC
]
change_table = [
    0xA7, 0x1C, 0x7E, 0xAF, 0xD9, 0xC2, 0xC0, 0xBE, 0x1F, 0x45,
    0x9A, 0x85, 0x26, 0xE3, 0x87, 0xC3, 0x21, 0xE0, 0x95, 0x10,
    0x71, 0x70, 0x02, 0x75, 0x35, 0xA5, 0x1D, 0x0D, 0x2F, 0xEE,
    0x25, 0x7B, 0xB5, 0x82, 0x66, 0x8D, 0xDB, 0x53, 0x3A, 0x29,
    0xD4, 0x43, 0x99, 0x97, 0x9D, 0xE8, 0x49, 0x00
]

magic = [
    0x52, 0xC3, 0x1A, 0xE0, 0x16, 0x5D, 0x5E, 0xE2, 0x67, 0x1F,
    0x1F, 0x06, 0x06, 0x1F, 0x17, 0x06, 0x0F, 0xF9, 0x06, 0x67,
    0x58, 0xB2, 0xE2, 0x8C, 0x0F, 0x2A, 0x06, 0x89, 0xCF, 0x2A,
    0x06, 0x1F, 0x98, 0x1A, 0x3E, 0x17, 0x67, 0x1F, 0xF7, 0x3A,
    0x44, 0xC3, 0x16, 0x33, 0x69, 0x1A, 0x75, 0x16, 0x3E, 0x17,
    0xD5, 0x69, 0x7A, 0x1B, 0x44, 0x44, 0x3E, 0x67, 0xF7, 0x89,
    0x67
]

for item in magic:
    for i in range(256):
        if xor_table[i] ^ i == item:
            if i in change_table:
                print(change_table.index(i), end=" ")
    print("")
```

利用这个可以找到所有可能的 v10。然后上面一部分是一个 47进制的转换（*怎么说呢，这种猜测的部分已经在比赛中出现了多次，从我这几次踩坑看下来，下次再遇到这种意义不明的运算，可以自己写一个模拟一下，然后从结果的数据特征猜测*）

```python
from Crypto.Util.number import *
index = [
    2, 0, 45, 44, 30, 40, 8, 23, 11, 37, 34, 43, 43, 37, 24, 19, 4, 29, 19, 22, 13, 5, 	   23, 41, 4, 35, 20, 9, 14, 35, 43, 37, 3, 33, 10, 24, 22, 37, 38, 1, 25, 0, 30, 6, 	 42, 45, 36, 30, 10, 24, 21, 42, 26, 28, 25, 25, 10, 7, 38, 9, 11
]

result = 0
index = index[::-1]
for i in range(len(index)):
    result += index[i] * pow(47, i)

print(long_to_bytes(result))
```

## shark | DONE

根据反调试相关代码位置的初始化，自己可以生成一份数据。拿到数据之后实际上就可以不用管数据了，直接patch反调试的位置然后只关注流程

```c
#include <stdio.h>
#include <stdint.h>

int main() {
    int32_t num = 0xEDB88320;
    int32_t v5 = 0;
    uint32_t v6 = 0;
    int16_t v7 = 0;
    int32_t wtf_table[256] = {0};

    do {
        v6 = (uint8_t)v5;
        v7 = 0;
        do {
            if ((v6 & 1) != 0)
                v6 = num ^ (v6 >> 1);
            else
                v6 = v6 >> 1;
            v7++;
        }
        while (v7 < 8);
        wtf_table[v5] = v6;
        v5++;
    }
    while (v5 != 256);
    for (int i = 0; i < 256; i++) {
        printf("%#x, ", wtf_table[i]);
    }
    return 0;
}
```

拿到之后根据程序流程获得汇编，由于patch了啥只与地址相关，可以不用管输入具体是什么。模拟加密之后，可以猜测是每两个字节取输入，进行20轮相同的加密，考虑直接爆破。
在模拟过程中注意：

- 逻辑右移的实现方法
- 取低8 bits

```python
asm = [
    'mov     ds:dword_404E50, 0FFFFFFFFh',
    'mov     ecx, ds:dword_404E40',
    'mov     dl, [ecx+4049F8h]', # input[0]
    'mov     byte ptr ds:unk_404E4C, dl',
    'movzx   eax, ds:byte_404E4C',
    'xor     eax, ds:dword_404E50',
    'mov     ds:byte_404E4C, al', # get the low 8 bits of eax
    'movzx   ecx, ds:byte_404E4C',
    'and     ecx, 0FFh', # (input[0] ^ 0xffffffff) & 0xff & 0xff
    'mov     ds:byte_404E4C, cl',
    'mov     edx, ds:dword_404E50',
    'shr     edx, 8',
    'mov     ds:dword_404E50, edx',
    'movzx   eax, ds:byte_404E4C',
    'mov     ecx, ds:dword_404E50',
    'xor     ecx, ds:wft_table[eax*4]',
    'mov     ds:dword_404E50, ecx',
    'mov     edx, ds:dword_404E40',
    'mov     al, [edx+4049F9h]', # input[1]
    'mov     ds:byte_404E4C, al',
    'movzx   ecx, ds:byte_404E4C',
    'xor     ecx, ds:dword_404E50',
    'mov     ds:byte_404E4C, cl', # get the low 8 bits of ecx
    'mov     edx, ds:dword_404E50',
    'shr     edx, 8',
    'mov     ds:dword_404E50, edx',
    'movzx   eax, ds:byte_404E4C',
    'mov     ecx, ds:dword_404E50',
    'xor     ecx, ds:wft_table[eax*4]'
    'mov     ds:dword_404E50, ecx'
    'mov     edx, ds:dword_404E50'
    'xor     edx, 0FFFFFFFFh',
    'mov     ds:dword_404E50, edx'
]

patch_length = [
    0x0000020A, 0x00000226, 0x00000236, 0x00000216, 0x00000317, 0x00000206, 			0x00000115, 0x00000317, 0x00000FF6, 0x00000216, 0x00000206, 0x00000FF3, 			0x00000206, 0x00000317, 0x00000206, 0x00000347, 0x00000206, 0x00000226, 			0x00000236, 0x00000115, 0x00000317, 0x00000206, 0x00000216, 0x00000206, 			0x00000FF3, 0x00000206, 0x00000317, 0x00000206, 0x00000347, 0x00000206, 			0x00000206, 0x00000FF3, 0x00000206]

wtf_table = [
         0, 0x77073096, 0xee0e612c, 0x990951ba, 0x76dc419, 0x706af48f, 0xe963a535, 0x9e6495a3, 0xedb8832, 0x79dcb8a4, 0xe0d5e91e, 0x97d2d988, 0x9b64c2b, 0x7eb17cbd, 0xe7b82d07, 0x90bf1d91, 0x1db71064, 0x6ab020f2, 0xf3b97148, 0x84be41de, 0x1adad47d, 0x6ddde4eb, 0xf4d4b551, 0x83d385c7, 0x136c9856, 0x646ba8c0, 0xfd62f97a, 0x8a65c9ec, 0x14015c4f, 0x63066cd9, 0xfa0f3d63, 0x8d080df5, 0x3b6e20c8, 0x4c69105e, 0xd56041e4, 0xa2677172, 0x3c03e4d1, 0x4b04d447, 0xd20d85fd, 0xa50ab56b, 0x35b5a8fa, 0x42b2986c, 0xdbbbc9d6, 0xacbcf940, 0x32d86ce3, 0x45df5c75, 0xdcd60dcf, 0xabd13d59, 0x26d930ac, 0x51de003a, 0xc8d75180, 0xbfd06116, 0x21b4f4b5, 0x56b3c423, 0xcfba9599, 0xb8bda50f, 0x2802b89e, 0x5f058808, 0xc60cd9b2, 0xb10be924, 0x2f6f7c87, 0x58684c11, 0xc1611dab, 0xb6662d3d, 0x76dc4190, 0x1db7106, 0x98d220bc, 0xefd5102a, 0x71b18589, 0x6b6b51f, 0x9fbfe4a5, 0xe8b8d433, 0x7807c9a2, 0xf00f934, 0x9609a88e, 0xe10e9818, 0x7f6a0dbb, 0x86d3d2d, 0x91646c97, 0xe6635c01, 0x6b6b51f4, 0x1c6c6162, 0x856530d8, 0xf262004e, 0x6c0695ed, 0x1b01a57b, 0x8208f4c1, 0xf50fc457, 0x65b0d9c6, 0x12b7e950, 0x8bbeb8ea, 0xfcb9887c, 0x62dd1ddf, 0x15da2d49, 0x8cd37cf3, 0xfbd44c65, 0x4db26158, 0x3ab551ce, 0xa3bc0074, 0xd4bb30e2, 0x4adfa541, 0x3dd895d7, 0xa4d1c46d, 0xd3d6f4fb, 0x4369e96a, 0x346ed9fc, 0xad678846, 0xda60b8d0, 0x44042d73, 0x33031de5, 0xaa0a4c5f, 0xdd0d7cc9, 0x5005713c, 0x270241aa, 0xbe0b1010, 0xc90c2086, 0x5768b525, 0x206f85b3, 0xb966d409, 0xce61e49f, 0x5edef90e, 0x29d9c998, 0xb0d09822, 0xc7d7a8b4, 0x59b33d17, 0x2eb40d81, 0xb7bd5c3b, 0xc0ba6cad, 0xedb88320, 0x9abfb3b6, 0x3b6e20c, 0x74b1d29a, 0xead54739, 0x9dd277af, 0x4db2615, 0x73dc1683, 0xe3630b12, 0x94643b84, 0xd6d6a3e, 0x7a6a5aa8, 0xe40ecf0b, 0x9309ff9d, 0xa00ae27, 0x7d079eb1, 0xf00f9344, 0x8708a3d2, 0x1e01f268, 0x6906c2fe, 0xf762575d, 0x806567cb, 0x196c3671, 0x6e6b06e7, 0xfed41b76, 0x89d32be0, 0x10da7a5a, 0x67dd4acc, 0xf9b9df6f, 0x8ebeeff9, 0x17b7be43, 0x60b08ed5, 0xd6d6a3e8, 0xa1d1937e, 0x38d8c2c4, 0x4fdff252, 0xd1bb67f1, 0xa6bc5767, 0x3fb506dd, 0x48b2364b, 0xd80d2bda, 0xaf0a1b4c, 0x36034af6, 0x41047a60, 0xdf60efc3, 0xa867df55, 0x316e8eef, 0x4669be79, 0xcb61b38c, 0xbc66831a, 0x256fd2a0, 0x5268e236, 0xcc0c7795, 0xbb0b4703, 0x220216b9, 0x5505262f, 0xc5ba3bbe, 0xb2bd0b28, 0x2bb45a92, 0x5cb36a04, 0xc2d7ffa7, 0xb5d0cf31, 0x2cd99e8b, 0x5bdeae1d, 0x9b64c2b0, 0xec63f226, 0x756aa39c, 0x26d930a, 0x9c0906a9, 0xeb0e363f, 0x72076785, 0x5005713, 0x95bf4a82, 0xe2b87a14, 0x7bb12bae, 0xcb61b38, 0x92d28e9b, 0xe5d5be0d, 0x7cdcefb7, 0xbdbdf21, 0x86d3d2d4, 0xf1d4e242, 0x68ddb3f8, 0x1fda836e, 0x81be16cd, 0xf6b9265b, 0x6fb077e1, 0x18b74777, 0x88085ae6, 0xff0f6a70, 0x66063bca, 0x11010b5c, 0x8f659eff, 0xf862ae69, 0x616bffd3, 0x166ccf45, 0xa00ae278, 0xd70dd2ee, 0x4e048354, 0x3903b3c2, 0xa7672661, 0xd06016f7, 0x4969474d, 0x3e6e77db, 0xaed16a4a, 0xd9d65adc, 0x40df0b66, 0x37d83bf0, 0xa9bcae53, 0xdebb9ec5, 0x47b2cf7f, 0x30b5ffe9, 0xbdbdf21c, 0xcabac28a, 0x53b39330, 0x24b4a3a6, 0xbad03605, 0xcdd70693, 0x54de5729, 0x23d967bf, 0xb3667a2e, 0xc4614ab8, 0x5d681b02, 0x2a6f2b94, 0xb40bbe37, 0xc30c8ea1, 0x5a05df1b, 0x2d02ef8d]


magic = [0xC0F6605E, 0x00B16E0A, 0x3319A2D2, 0x57CAB7B7, 0x9A646D9C, 0xBDD82726, 0xD838FB91, 0x8DE10BB3, 0x176B0DAD, 0x685FDEEF,
         0x2C1FF7B1, 0x6C444296, 0xA15CFE90, 0x20CD8721, 0x62967CE8, 0x2C1641FD, 0x572D0F9A, 0xAE52DC2C, 0x50497DCF, 0xFF6ABF4A]

def gen_patch_length_table():
    result = []
    for i in patch_length:
        result.append(i&0xf)
  
    return result


def unsigned32(signed):
    return signed % 0x100000000

def enc_func(a, b):
    result = 0xFFFFFFFF
    temp = (result ^ a) & 0xff
    result =  unsigned32(result) >> 8
    result = result ^ wtf_table[temp]
    temp = (result ^ b) & 0xff
    result = unsigned32(result) >> 8
    result = result ^ wtf_table[temp]
    result ^= 0xFFFFFFFF
    return result

def patch(addr, patch_table_item): # just a model
    v2 = patch_table_item >> 8
    choice = patch_table_item >> 4
    if choice == 0:
        pass
    elif choice == 1:
        pass
    elif choice == 2:
        pass
    elif choice == 3:
        pass
    elif choice == 4:
        pass

if __name__ == '__main__':
    flag = []
    for cnt in range(len(magic)):
        for i in range(256):
            for j in range(256):
                if enc_func(i, j) == magic[cnt]:
                    flag.append(i)
                    flag.append(j)
                    break

    print(bytes(flag))
```

## easy_mobile | OPEN

在 nativelib 中有一个平坦化，感觉是改过的，，可以从一些勉强可读的代码看出去完应该还有类似等价表达式一类的混淆，总之就是 ollvm 的变态整合，比赛时时间不够了，挖个坑之后尝试下手去这个混淆。
