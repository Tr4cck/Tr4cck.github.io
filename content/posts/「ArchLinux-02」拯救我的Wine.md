---
title: 「ArchLinux」拯救我的 Wine
date: 2022-01-04 22:51:44
tags:
- Linux
categories:
- TECHNOLOGY
---


<!-- more -->

## wine-6.23-1

> 参考：https://wiki.winehq.org/Wine_Developer%27s_Guide/Debugging_Wine

使用 `winedbg ida.exe` 启动调试器，单步调试，发现调试到下面的位置再尝试单步步进，但是 IDA 会直接启动，这个不知道被调用函数内部的调用：

`0x000000017000c968 ntdll+0xc968: calll  *0x000000007ffe1000`

启动之后可以正常启动

## wine-7.0rc4-1

可以说汇编一模一样，没有任何区别，但是同样调试到上述位置处时，尝试步进，可以打开 IDA 的欢迎界面：

![](https://s2.loli.net/2022/01/04/sdBPNE2cOe7n6HG.png)

但是尝试点击 Go 启动时，winedbg 直接触发了异常：

```
Unhandled exception: 0x40000015 in 64-bit code (0x000000007b0123ae).
016c:fixme:dbghelp:dump_unwind_info couldn't read memory for UNWIND_INFO at af740ab8
Register dump:
 rip:000000007b0123ae rsp:0000000012e2ec10 rbp:000000000310a2f0 eflags:00000202 (   - --  I   - - - )
 rax:0000000012e2ec50 rbx:00000000033af690 rcx:0000000012e2ec30 rdx:0000000000000001
 rsi:0000000012e2ed28 rdi:0000000012e2ec58  r8:0000000000000001  r9:0000000012e2ed20 r10:0000000000000000
 r11:0000000000000001 r12:00000000033ac2f0 r13:0000000000000000 r14:0000000000000000 r15:0000000000000000
Stack dump:
0x0000000012e2ec10:  0000000012e2ec30 0000000000000000
0x0000000012e2ec20:  0000000000000000 0000000000000024
0x0000000012e2ec30:  0000000140000015 0000000000000000
0x0000000012e2ec40:  000000007b0123ae 0000000000000001
0x0000000012e2ec50:  00000000033af690 00000003af710280
0x0000000012e2ec60:  0000000000000038 000000000310a328
0x0000000012e2ec70:  000000000310a2f0 00000003af710280
0x0000000012e2ec80:  000000000000001e 0000000000000000
0x0000000012e2ec90:  0000000000000000 00000003af6a94cd
0x0000000012e2eca0:  00000000fffffffe 00000000033af690
0x0000000012e2ecb0:  0000000000000000 000000000310a328
0x0000000012e2ecc0:  000000000310a2f0 00000000033af690
Backtrace:
=>0 0x000000007b0123ae AccessCheckByTypeAndAuditAlarmW+0x113ae() in kernelbase (0x000000000310a2f0)
  1 0x00000000118e8181 in libzmq-v142-mt-4_3_4-4e355e3e (+0x18181) (0x000000000310a2f0)
  2 0x00000000118e75a2 in libzmq-v142-mt-4_3_4-4e355e3e (+0x175a2) (0x000000000310a2f0)
  3 0x0000000011912998 zmq_atomic_counter_inc+0x23138() in libzmq-v142-mt-4_3_4-4e355e3e (0x0000000000000000)
0x000000007b0123ae kernelbase+0x123ae: nop
```

可以看到触发了 0x40000015 异常，但是我对于 windows 异常代码以及问题并没有太多了解。我尝试搜索了一下，并没有很好的解决方案。但是通过调用栈，可以确定异常的发生在于AccessCheckByTypeAndAuditAlarmW 函数内部，这是 windowsapi，貌似用于访问权限检查。但我无法确定在 wine-6.23-1 中进入该函数中的行为和上下文是什么。于是寄辣！

## solution

嘻嘻, 不如开摆等更新.
