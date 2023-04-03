---
title: 「CISCN2021」Rev 部分题解
date: 2021-05-16 22:35:43
tags:
- Rev
categories:
- TECHNOLOGY
---


<!-- more -->

Team: Mini-7

Members: cyxq \ avalon \ wanan \ track

## glass

### Android Part

使用 jadx-gui 反编译附件中的 apk, 定位到 `MainActivity`, 得到:

```java
package com.ciscn.glass;

import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {
    Button but;
    EditText txt;

    public native boolean checkFlag(String str);

    static {
        System.loadLibrary("native-lib");
    }

    /* access modifiers changed from: protected */
    @Override // androidx.core.app.ComponentActivity, androidx.appcompat.app.AppCompatActivity, androidx.fragment.app.FragmentActivity
    public void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        setContentView(C0273R.layout.activity_main);
        this.but = (Button) findViewById(C0273R.C0275id.button);
        this.txt = (EditText) findViewById(C0273R.C0275id.editText);
        this.but.setOnClickListener(new View.OnClickListener() {
            /* class com.ciscn.glass.MainActivity.View$OnClickListenerC02721 */

            public void onClick(View view) {
                MainActivity mainActivity = MainActivity.this;
                if (mainActivity.checkFlag(mainActivity.txt.getText().toString())) {
                    Toast.makeText(MainActivity.this, "right!", 0).show();
                } else {
                    Toast.makeText(MainActivity.this, "wrong!", 0).show();
                }
            }
        });
    }
}
```

容易发现明显的 JNI 特征, 故将 apk 解包出来, 得到 so 文件.

### JNI

拖进 IDA, 定位到 `Java_com_ciscn_glass_MainActivity_checkFlag`, 看看 `sub_FFC`:

```c
int __fastcall sub_FFC(char *_box, char *_key, int _key_len)
{
  int i; // r6
  int v7; // r1
  int v8; // r0
  int v9; // r1
  int v10; // r3
  _BYTE v12[260]; // [sp+0h] [bp-120h] BYREF

  memset(v12, 0, 0x100u);
  for ( i = 0; i != 256; ++i )
  {
    _box[i] = i;
    sub_126C(i, _key_len);
    v12[i] = _key[v7];
  }
  v8 = 0;
  v9 = 0;
  while ( v8 != 256 )
  {
    v10 = (unsigned __int8)_box[v8];
    v9 = (v9 + v10 + (unsigned __int8)v12[v8]) % 256;
    _box[v8++] = _box[v9];
    _box[v9] = v10;
  }
  return _stack_chk_guard;
}
```

发现明显的 RC4 特征, 但有所不同.

### Angr

采用一种特殊的办法: 将 so 文件复制下来稍加修改:

```c
#include "defs.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void *__fastcall sub_10D4(char *_input, int _input_len, char *_key, int _key_len)
{
  int i;
  char *v5;
  char v6;
  char v7;
  char v8;
  int j;
  int k;

  for ( i = 0; i < _input_len; i += 3 )
  {
    v5 = &_input[i];
    v6 = _input[i + 2];
    v7 = _input[i + 1];
    v8 = _input[i] ^ v6;
    _input[i] = v8;
    v5[2] = v6 ^ v7;
    v5[1] = v7 ^ v8;
  }
  for ( j = 0; j < _input_len; j += _key_len )
  {
    for ( k = 0; (_key_len & ~(_key_len >> 31)) != k && j + k < _input_len; ++k )
      _input[k] ^= _key[k];
    _input += _key_len;
  }
}

void *__fastcall sub_1088(char *_box, char *_input, int _key_len)
{
  int v3;
  int v4;
  int v5;

  v3 = 0;
  v4 = 0;
  while ( _key_len )
  {
    --_key_len;
    v4 = (v4 + 1) % 256;
    v5 = (unsigned __int8)_box[v4];
    v3 = (v3 + v5) % 256;
    _box[v4] = _box[v3];
    _box[v3] = v5;
    *_input++ ^= _box[(unsigned __int8)(v5 + _box[v4])];
  }
}

void __fastcall sub_FFC(char *_box, char *_key, int _key_len)
{
  int i;
  int v8;
  int v9;
  int v10;
  _BYTE v12[260];

  memset(v12, 0, 0x100u);
  for ( i = 0; i != 256; ++i )
  {
    _box[i] = i;
    v12[i] = _key[i % _key_len];
  }
  v8 = 0;
  v9 = 0;
  while ( v8 != 256 )
  {
    v10 = (unsigned __int8)_box[v8];
    v9 = (v9 + v10 + (unsigned __int8)v12[v8]) % 256;
    _box[v8++] = _box[v9];
    _box[v9] = v10;
  }
}

int main() {
    char key[256];
    char box[260];
    char enc[] = {0xA3,0x1A,0xE3,0x69,0x2F,0xBB,0x1A,0x84,0x65,0xC2,0xAD,0xAD,0x9E,0x96,0x05,0x02,0x1F,0x8E,0x36,0x4F,0xE1,0xEB,0xAF,0xF0,0xEA,0xC4,0xA8,0x2D,0x42,0xC7,0x6E,0x3F,0xB0,0xD3,0xCC,0x78,0xF9,0x98,0x3F};
    char input[100] = {0};
    scanf("%s", input);
    if ( strlen(input) != 39 ){
        printf("Wrong length");
        return 0;
    }
    memset(box, 0, 256u);
    memcpy(key, "12345678", sizeof(key));
    int key_len = strlen(key);
    sub_FFC(box, key, key_len);
    sub_1088(box, input, 39);
    sub_10D4(input, 39, key, key_len);
    if(strncmp(input, enc, 0x27) == 0) {
        printf("Congratulations!");
        return 0;
    }
    else {
        printf("Ruaaa~");
        return 0;
    }
}
```

以上代码和原代码作用相同, 编译出可执行文件后, 使用 angr 爆破

```python
import angr, claripy
proj = angr.Project('./mod_glass.exe', load_options={'auto_load_libs':False})
flag_chars = [claripy.BVS('flag_%d' % i, 8) for i in range(39)]
flag = claripy.Concat(*flag_chars+[claripy.BVV(b'\n')])
state = proj.factory.blank_state(addr=0x4016F9, stdin=flag)
simgr = proj.factory.simgr(state)
simgr.explore(avoid=[0x4018D5, 0x4017DD], find=0x4018C2)
print(simgr.found[0].posix.dumps(0))
b'CISCN{6654d84617f627c88846c172e0f4d46c}\n'
```

## baby_bc

### 获取 ELF 文件

- llvm-dis ./baby.bc
- clang ./baby.ll

### IDA 分析

明显的数独特征, 有一些限制, 手解的, 需要了解 z3 解法与深搜爆破.

## gift

> 2021-08-07 复现完毕, 来更一下.

### First Look

使用 IDA7.6 Free 版本可以看到符号信息, 首先我们关注到 `main_CISCN6666666()`, `main_CISCN66666666()`, `main_CISCN6666666666()`, 由于它们位于 `main_main` 的最开始, 猜测是输出一些欢迎信息和输出提示之类的东西, 所以考虑直接把断点打在第三个函数, 运行一下, 发现我们的猜测果然是对的, 那就先不用分析这三个函数, 直接往下看.

有一个比较大的 `while` 循环, 稍微看看可以发现就是一个 `for (int i = 0; i < 32; i++)` 的过程, 或者更详细一点, 这个循环就是在遍历下面这个东西:

![1.png](https://i.loli.net/2021/08/07/NoPirn6GpjBxTQl.png)

### Reversing

再下面的 `makeslice` 是第一个重点, 考虑到 `golang` 内部的 `slice` 实现:

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

并且紧跟着就有一个未初始化的变量直接用来赋值, 猜测应该是 `makeslice` 的结果, 所以我们自己创建一个结构体, 方便静态分析:

![2.png](https://i.loli.net/2021/08/07/L3GNwAOhucZ4xFf.png)

大概这样, **一定注意每个成员的大小 (满足对齐) 和顺序**. 创建成功之后把相关变量的类型声明改成这个结构体, 发现代码可读性明显变好了. 接着看, 下面又有一个类似的 `while` 循环, 实际上就是等价于:

```c
for (int i = 1; i <= 4; i++) {
    main_wtf(0, i, slice)
}
```

然后下面突然使用 `qword_1C20E8` 来取得一些值, 但是前面没有对它赋值, 猜想应该是 `main_wtf` 里面实现的. 流程大概就是先根据这个值取一个数组里面的一个元素, 并且把该元素与 0x66 异或之后 \*8, 取得另一个数组里面的元素. 关于这个 \*8 实际上看下对应数组, 不难发现, 这个矩阵是形如:

$$
0\quad 0\quad 0\quad 0\quad 0\quad 0\quad 0\quad 0\\
1\quad 0\quad 0\quad 0\quad 0\quad 0\quad 0\quad 0\\
2\quad 0\quad 0\quad 0\quad 0\quad 0\quad 0\quad 0\\
3\quad 0\quad 0\quad 0\quad 0\quad 0\quad 0\quad 0\\
...
$$



所以列数就是第一列的数, 也就是并不改变结果.

### What the fxck

然后我们跟进 `main_wtf` 里看看, 发现递归调用, 并且随着递归深入, 第一个参数依次 +1, 大概美化一下:

```c
// a1 = 0
void __golang main_wtf(__int64 depth, __int64 a2, struct slice a3)
{
  __int64 v3; // rax
  __int64 len; // rcx
  __int64 v5; // rdx
  struct slice v6; // [rsp+10h] [rbp-28h]
  __int64 v7; // [rsp+28h] [rbp-10h]
  void *retaddr; // [rsp+38h] [rbp+0h] BYREF

  while ( (unsigned __int64)&retaddr <= *(_QWORD *)(*(_QWORD *)NtCurrentTeb()->NtTib.ArbitraryUserPointer + 16LL) )
    runtime_morestack_noctxt();
  v3 = depth;
  len = a3.len;
  if ( (unsigned __int64)depth >= a3.len )
    runtime_panicIndex();
  a3.array[depth] = a2;
  if ( depth == a3.len - 1 )
  {
    if ( main_goooo(a3) )
      qword_1C20E8 = qword_1C20E8
                   - 17
                   * (((__int64)(qword_1C20E8
                               + ((unsigned __int128)((qword_1C20E8 + 1) * (__int128)(__int64)0xF0F0F0F0F0F0F0F1LL) >> 64)
                               + 1) >> 4)
                    - ((qword_1C20E8 + 1) >> 63))
                   + 1;
  }
  else
  {
    v5 = 1LL;
    while ( v5 <= 4 )
    {
      v7 = v5;
      v6.array = a3.array;
      v6.len = len;
      v6.cap = a3.cap;
      main_wtf(v3 + 1, v5, v6);
      v5 = v7 + 1;
      v3 = depth;
      len = a3.len;
    }
  }
}
```

逻辑也比较清晰了，看看 `main_goooo`, 美化代码方法类似:

```c
__int8 __golang main_goooo(struct slice a1)
{
  __int64 i; // rax
  unsigned __int64 v2; // rsi
  __int64 v3[5]; // [rsp+10h] [rbp-30h] BYREF

  memset(v3, 0, sizeof(v3));
  for ( i = 0LL; a1.len > i; ++i )
  {
    v2 = a1.array[i];
    if ( v2 >= 5 )
      runtime_panicIndex();
    v3[v2] ^= 1uLL;
  }
  return !v3[1] && !v3[3];
}
```

### Python Simulation

```python
enc = 0
timelist = [
    0x01, 0x03, 0x06, 0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10, 0x11, 0x12, 0x14,
    0x19, 0x1E, 0x28, 0x42, 0x66, 0x0A0, 0x936, 0x3D21, 0x149A7, 0x243AC, 0x0CB5BE, 0x47DC61,
    0x16C0F46, 0x262C432, 0x4ACE299, 0x10FBC92A, 0x329ECDFD, 0x370D7470
]

def main_goooo(array):
    box = 5*[0]
    for i in array:
        box[i] ^= 1
    return box[1] == 0 and box[3] == 0

def main_wtf(depth, jj, array):
    global enc, enc_time
    array[depth] = jj
    if depth == len(array) - 1:
        if (main_goooo(array)):
            enc = enc - 17 * (((enc + (((enc + 1) * 0xF0F0F0F0F0F0F0F1) >> 64) + 1) >> 4) - ((enc + 1) >> 63)) + 1
    else:
        for k in [1, 2, 3, 4]:
            main_wtf(depth + 1, k, array)

def main_main():
    global enc, enc_time
    for aTime in timelist:
        enc = 0
        enc_time = 0
        for j in [1, 2, 3, 4]:
            main_wtf(0, j, aTime*[0])
        print(enc)

if __name__ == '__main__':
    main_main()
```

这个脚本和题目的核心功能一致, 下面找一下优化点, 就是 `main_goooo` 这个东西, 它是不一定执行的, 反复循环判断会随着数据规模增大而大幅消耗时间, 所以我们看看能否确定 enc 的计算次数的通项, 这样就不用花时间在判断上了:

```python
enc = 0
enc_time = 0
timelist = [
    0x01, 0x03, 0x06, 0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10, 0x11, 0x12, 0x14,
    0x19, 0x1E, 0x28, 0x42, 0x66, 0x0A0, 0x936, 0x3D21, 0x149A7, 0x243AC, 0x0CB5BE, 0x47DC61,
    0x16C0F46, 0x262C432, 0x4ACE299, 0x10FBC92A, 0x329ECDFD, 0x370D7470
]

# 不一定执行，有时间损耗，所以我们可以尝试计算一下enc的计算次数
def main_goooo(array):
    box = 5*[0]
    for i in array:
        box[i] ^= 1
    return box[1] == 0 and box[3] == 0

def main_wtf(depth, jj, array):
    global enc, enc_time
    array[depth] = jj
    if depth == len(array) - 1:
        if (main_goooo(array)):
            enc_time += 1
            enc = enc - 17 * (((enc + (((enc + 1) * 0xF0F0F0F0F0F0F0F1) >> 64) + 1) >> 4) - ((enc + 1) >> 63)) + 1
    else:
        for k in [1, 2, 3, 4]:
            main_wtf(depth + 1, k, array)

def main_main():
    global enc, enc_time
    for aTime in timelist:
        enc = 0
        enc_time = 0
        for j in [1, 2, 3, 4]:
            main_wtf(0, j, aTime*[0])
        print(enc_time)

if __name__ == '__main__':
    main_main()
    # 2
    # 20
    # 1056
    # 65792
    # 262656
    # 1049600
    # 2**n + 4**n
```

结果已经写在注释里面了, 然后我们生成几个 enc 的测试数据:

```python
def enc_test():
    for n in range(20):
        enc = 0
        for _ in range(2**n + 4**n):
            enc = enc - 17 * (((enc + (((enc + 1) * 0xF0F0F0F0F0F0F0F1) >> 64) + 1) >> 4) - ((enc + 1) >> 63)) + 1
        print(enc, end=", ")
        # 2, 6, 3, 4, 0, 2, -5, 5, 2, 6, 3, 4, 0, 2, ...
```

发现周期性, 然后就没有然后了:

```python
enc = 0
enc_time = 0
timelist = [
    0x01, 0x03, 0x06, 0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10, 0x11, 0x12, 0x14,
    0x19, 0x1E, 0x28, 0x42, 0x66, 0x0A0, 0x936, 0x3D21, 0x149A7, 0x243AC, 0x0CB5BE, 0x47DC61,
    0x16C0F46, 0x262C432, 0x4ACE299, 0x10FBC92A, 0x329ECDFD, 0x370D7470
]

# 不一定执行，有时间损耗，所以我们可以尝试计算一下enc的计算次数
def main_goooo(array):
    box = 5*[0]
    for i in array:
        box[i] ^= 1
    return box[1] == 0 and box[3] == 0

def main_wtf(depth, jj, array):
    global enc, enc_time
    array[depth] = jj
    if depth == len(array) - 1:
        if (main_goooo(array)):
            enc_time += 1
            enc = enc - 17 * (((enc + (((enc + 1) * 0xF0F0F0F0F0F0F0F1) >> 64) + 1) >> 4) - ((enc + 1) >> 63)) + 1
    else:
        for k in [1, 2, 3, 4]:
            main_wtf(depth + 1, k, array)

def enc_test():
    for n in range(20):
        enc = 0
        for _ in range(2**n + 4**n):
            enc = enc - 17 * (((enc + (((enc + 1) * 0xF0F0F0F0F0F0F0F1) >> 64) + 1) >> 4) - ((enc + 1) >> 63)) + 1
        print(enc, end=", ")
        # 2, 6, 3, 4, 0, 2, -5, 5, 2, 6, 3, 4, 0, 2, ...

def main_main():
    global enc, enc_time
    for aTime in timelist:
        enc = 0
        enc_time = 0
        for j in [1, 2, 3, 4]:
            main_wtf(0, j, aTime*[0])
        print(enc_time)

def get_flag():
    sbox = [84, 94, 82, 4, 85, 5, 83, 95, 80, 7, 84, 86, 81, 2, 3, 0, 87]
    enc_cycle = [2, 6, 3, 4, 0, 2, 12, 5]
    print(''.join(chr(sbox[enc_cycle[(i - 1)%8]] ^ 0x66) for i in timelist))

if __name__ == '__main__':
    get_flag()
    # main_main()
    # 2
    # 20
    # 1056
    # 65792
    # 262656
    # 1049600
    # 2**n + 4**n
```
