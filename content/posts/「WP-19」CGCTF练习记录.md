---
title: 「CGCTF」部分练习记录
date: 2022-03-18 19:47:33
tags:
- Rev
categories:
- TECHNOLOGY
---


<!-- more -->

## WxyVM1

`main`:

```cpp
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  char isok; // [rsp+Bh] [rbp-5h]
  int i; // [rsp+Ch] [rbp-4h]

  puts("[WxyVM 0.0.1]");
  puts("input your flag:");
  scanf("%s", &input);
  isok = 1;
  runVM();
  if ( strlen(&input) != 24 )
    isok = 0;
  for ( i = 0; i <= 23; ++i )
  {
    if ( *(&input + i) != dword_601060[i] )
      isok = 0;
  }
  if ( isok )
    puts("correct");
  else
    puts("wrong");
  return 0LL;
}
```

`runVM`:

```cpp
void __fastcall runVM()
{
  int i; // [rsp+0h] [rbp-10h]
  char v1; // [rsp+8h] [rbp-8h]
  int v2; // [rsp+Ch] [rbp-4h]

  for ( i = 0; i <= 14999; i += 3 )
  {
    v1 = byte_6010C0[i + 2];
    v2 = byte_6010C0[i + 1];
    switch ( byte_6010C0[i] )
    {
      case 1:
        *(&input + v2) += v1;
        break;
      case 2:
        *(&input + v2) -= v1;
        break;
      case 3:
        *(&input + v2) ^= v1;
        break;
      case 4:
        *(&input + v2) *= v1;
        break;
      case 5:
        *(&input + v2) ^= *(&input + byte_6010C0[i + 2]);
        break;
      default:
        continue;
    }
  }
}
```

`exp`:

```py
with open('opcode-list.dat', 'rb') as fin:
    data = fin.read()

pseudocode = ''
for i in range(14997, -1, -3):
    v2, v1 = data[i + 1], data[i + 2]
    opcode = data[i]
    match opcode:
        case 1:
            pseudocode += f'input[{v2}] -= {v1}\n'
        case 2:
            pseudocode += f'input[{v2}] += {v1}\n'
        case 3:
            pseudocode += f'input[{v2}] ^= {v1}\n'
        case 4:
            pseudocode += f'input[{v2}] //= {v1}\n'
        case 5:
            pseudocode += f'input[{v2}] ^= input[{data[v1]}]\n'

with open('pseudocode.dat', 'wb') as fout:
    fout.write(pseudocode.encode())
```

## WxyVM2

差不多, 不同的是中间插入了一些花指令, 导致无用操作, 全部拿出来处理一下就行:

```py
lines = open('code.dat', 'r').read().split('\n')[::-1]
c_header = '''
#include <stdio.h>
int main() {
  unsigned int byte_694100[25] = {0xFFFFFFC0, 0xFFFFFF85, 0xFFFFFFF9, 0x0000006C, 0xFFFFFFE2, 0x00000014, 0xFFFFFFBB, 0xFFFFFFE4, 0x0000000D, 0x00000059, 0x0000001C, 0x00000023, 0xFFFFFF88, 0x0000006E, 0xFFFFFF9B, 0xFFFFFFCA, 0xFFFFFFBA, 0x0000005C, 0x00000037, 0xFFFFFFFF, 0x00000048, 0xFFFFFFD8, 0x0000001F, 0xFFFFFFAB, 0xFFFFFFA5};
'''
with open('exp.c', 'w') as fout:
    fout.write(c_header)
    for line in lines:
        if 'dword' not in line:
            if '+=' in line:
                fout.write('  ' + line.replace('+=', '-='))
            elif '-=' in line:
                fout.write('  ' + line.replace('-=', '+='))
            elif '--' in line:
                fout.write('  ' + line.replace('--', '++'))
            elif '++' in line:
                fout.write('  ' + line.replace('++', '--'))
            else:
                fout.write('  ' + line)
            fout.write('\n')

    c_tail = '''
  for (int i = 0; i < 25; i++)
    printf("%c", byte_694100[i] & 0xff);

  return 0;
}
'''
    fout.write(c_tail)
```
