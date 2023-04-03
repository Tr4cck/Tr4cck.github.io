---
title: 「HITCONCTF2021」Rev复现
date: 2022-04-10 12:45:16
tags:
- Rev
categories:
- TECHNOLOGY
---


<!-- more -->

## cclemon

先给出附件:

```
0: const 26 ; <module 'src'>
5: module 15 833

11: const 27 ; 68694329
16: store 0 0
19: const 28 ; w
24: define 0 0 0 0 59
33: load 1 0
36: const 29 ; 1259409
41: mul
42: const 30 ; 321625345
47: add
48: const 31 ; 4294967296
53: mod
54: dup
55: store 1 0
58: return
59: store 0 1 -> func w

62: const 32 ; n
67: const 33 ; __init__
72: define 0 0 1 2 157
81: array 0
86: self
87: const 34 ; a
92: setattr
93: const 35 ; 0
98: store 0 1
101: load 0 1
104: load 0 0
107: lt
108: jz 151
113: self
114: const 34 ; a
119: getattr
120: const 36 ; append
125: getattr -> func append
126: load 1 1
129: call 0
131: call 1
133: pop
134: load 0 1
137: const 37 ; 1
142: add
143: store 0 1
146: jmp 101
151: const 38 ; nil
156: return
157: const 33 ; __init__

162: const 39 ; x
167: const 40 ; y
172: const 41 ; r
177: define 0 0 2 2 277
186: load 0 0
189: load 0 1
192: gt
193: jz 214
198: self
199: const 41 ; r
204: getattr
205: load 0 0
208: load 0 1
211: tailcall 2
213: return

214: load 0 0
217: load 0 1
220: lt
221: jz 271
226: self
227: const 42 ; s
232: getattr
233: load 0 1
236: load 0 0
239: call 2
241: pop
242: load 0 0
245: const 37 ; 1
250: add
251: store 0 0
254: load 0 1
257: const 37 ; 1
262: sub
263: store 0 1
266: jmp 214
271: const 38 ; nil
276: return
277: const 41 ; r

282: const 39 ; x
287: const 40 ; y
292: const 42 ; s
297: define 0 0 2 3 362
306: self
307: const 34 ; a
312: getattr
313: load 0 0
316: getitem
317: store 0 2
320: self
321: const 34 ; a
326: getattr
327: load 0 1
330: getitem
331: self
332: const 34 ; a
337: getattr
338: load 0 0
341: setitem
342: load 0 2
345: self
346: const 34 ; a
351: getattr
352: load 0 1
355: setitem
356: const 38 ; nil
361: return
362: const 42 ; s

367: const 39 ; x
372: const 40 ; y
377: const 43 ; val
382: const 44 ; o
387: define 0 0 3 4 494
396: load 0 0
399: load 0 1
402: gt
403: jz 427
408: self
409: const 44 ; o
414: getattr
415: load 0 2
418: load 0 0
421: load 0 1
424: tailcall 3
426: return

427: load 0 0
430: store 0 3
433: load 0 3
436: load 0 1
439: le
440: jz 488
445: self
446: const 34 ; a
451: getattr
452: load 0 3
455: getitem
456: load 0 2
459: bxor
460: self
461: const 34 ; a
466: getattr
467: load 0 3
470: setitem
471: load 0 3
474: const 37 ; 1
479: add
480: store 0 3
483: jmp 433
488: const 38 ; nil
493: return

494: const 44 ; o
499: const 45 ; A
504: class 8 0
507: store 0 2 -> class A

510: const 46 ; 200000
515: store 0 7
518: load 0 2
521: load 0 7
524: call 1
526: store 0 8
529: const 35 ; 0
534: store 0 9
537: load 0 9
540: load 0 7
543: const 47 ; 5
548: mul
549: lt
550: jz 727
555: load 0 1
558: call 0
560: const 48 ; 3
565: mod
566: store 0 10
569: load 0 1
572: call 0
574: load 0 7
577: mod
578: store 0 11
581: load 0 1
584: call 0
586: load 0 7
589: mod
590: store 0 12
593: load 0 10
596: const 35 ; 0
601: eq

602: jz 630
607: load 0 8
610: const 41 ; r
615: getattr
616: load 0 12
619: load 0 11
622: call 2
624: pop
625: jmp 710
630: load 0 10
633: const 37 ; 1
638: eq

639: jz 667
644: load 0 8
647: const 42 ; s
652: getattr
653: load 0 12
656: load 0 11
659: call 2
661: pop
662: jmp 710
667: load 0 10
670: const 49 ; 2
675: eq
676: jz 710
681: load 0 1
684: call 0
686: store 0 13
689: load 0 8
692: const 44 ; o
697: getattr
698: load 0 13
701: load 0 12
704: load 0 11
707: call 3
709: pop

710: load 0 9
713: const 37 ; 1
718: add
719: store 0 9
722: jmp 537 -> while end

727: const 35 ; 0
732: store 0 14
735: const 35 ; 0
740: store 0 9
743: load 0 9
746: load 0 7
749: lt
750: jz 802
755: load 0 14 -> v14
758: load 0 8
761: const 34 ; a
766: getattr
767: load 0 9
770: getitem -> v8.a[v9]
771: load 0 9 -> v9
774: const 37 ; 1 -> 1
779: add
780: mul
781: add
782: store 0 14
785: load 0 9
788: const 37 ; 1
793: add
794: store 0 9
797: jmp 743
802: const 23 ; <function 'print'>
807: const 50 ; hitcon{
812: load 0 14
815: const 51 ; __string__
820: getattr
821: call 0
823: add
824: const 52 ; }
829: add
830: call 1
832: pop
```

搜索一下, 可以找到[项目地址](https://github.com/lemon-lang/lemon.git), 文档里关于 bytecode 啥也没有, 让我很异或, 但是这语言就像 c 混合 python 一样, 所以学起来还挺快.

只能手撕字节码🌶️, 毕竟有了解过 python bytecode 的话看这个也挺好猜的. 它是一种类似多层双链表的内存管理结构, 然后 GC 也是基于此的. 如果有兴趣可以看看源码, 挺好理解的, 除了有些地方可能有点离谱.

还原如下，有点长:

```
// bytecode report
// load a b: `<-` push from `buffer + b` to `sp + a`
// store a b: `->` pop from `sp + a` to `buffer + b`
// const: load a const value to `the buffer`

var seed = 68694329;
def w() {
    seed = (seed * 1259409 + 321625345) % 4294967296;
    return seed;
}

class A {
    def __init__(var n) {
        self.a = [];
        var i = 0;
        while (i < n) {
            self.a.append(w());
            i = i + 1;
        }
    }

    def s(var x, var y) { // swap
        var tmp = self.a[x];
        self.a[x] = self.a[y];
        self.a[y] = tmp;
    }

    def r(var x, var y) {
        if (x > y) {
            return self.r(y, x);
        }
        while (x < y) {
            self.s(x, y);
            x += 1;
            y -= 1;
        }
    }

    def o(var x, var y, var val) {
        if (x > y) {
            return self.o(y, x, val);
        }
        var tmp = x;
        while (tmp <= y) {
            self.a[tmp] ^= val;
            tmp += 1;
        }
    }
}

var v7 = 200000;
var v8 = A(v7);
var v9 = 0;
while (v9 < 5 * v7) {
    var v10 = w() % 3;
    var v11 = w() % v7;
    var v12 = w() % v7;
    if (v10 == 0) {
        v8.r(v11, v12);
    } else if (v10 == 1) {
        v8.s(v11, v12);
    } else {
        v13 = w();
        v8.o(v11, v12, v13);
    }
    v9 += 1;
}
v14 = 0;
v9 = 0;
while (v9 < v7) {
    v14 += v8.a[v9] * (v9 + 1);
    v9 += 1;
}

print("hitcon{" + v14.__string() + "}"); // like `str` in python
```

跑一遍就有 flag 咯, 但是当然是要翻译成 C 来跑啦, 这里就不给了, 毕竟长的可能和上面还原的东西差不多 (x

## mercy

### Background

原题改自 DEFCON-25-Final 的 cLEMENCy, 具体的开发者心路历程可以看看这篇[博客](https://blog.legitbs.net/2017/10/clemency-showing-mercy.html), 看了之后特别震撼, 两三年时间开发一个新体系结构, 并且赛前给出文档和工具链. 然后当年的选手视角来看这道题的话, 比较有代表性的是[这篇](https://blog.trailofbits.com/2017/07/30/an-extra-bit-of-analysis-for-clemency/), 看看这段话:

> Ryan, Sophia, and I wrote and used a Binary Ninja processor module for cLEMENCy during the event. This helped our team analyze challenges with Binary Ninja’s graph view and dataflow analyses faster than if we’d relied on the limited disassembler and debugger provided by the organizers. We are releasing this processor module today in the interest of helping others who want to try out the challenges on their own.

或许这就是神仙吧, 赛中写工具的外国猛男 (x

### Analysis

那么现在来看看这个架构吧. 它最有特点的就是**中端序**和 1 byte = 9 bits. 那么怎么理解这两个特点, 或者说具体影响了哪些地方呢?
