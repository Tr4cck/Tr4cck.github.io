---
title: 「xhljCTF202」Rev | flow 利用 SROP 隐藏控制流
date: 2022-01-13 23:34:16
categories:
- TECHNOLOGY
tags:
- Pwnable
- Rev
---

看帖看到, 遗憾找不到附件, 只能虚空复现一下.

<!--more -->

## SROP

**首先来点 pwn 的基础知识**.

> SROP 是一种高级的 ROP 技术, 核心原理是 signal 被触发时, sigreturn 系统调用会间接的被触发.

signal 主要用于在类 unix 系统中实现进程间通信. 来张图展示一下基本流程:

![](https://ctf-wiki.org/pwn/linux/user-mode/stackoverflow/x86/advanced-rop/figure/ProcessOfSignalHandlering.png)

- 首先是内核向某个进程发送 signal, 该进程会被暂时挂起, 进入内核态
- 内核会将被挂起进程的上下文存在栈上, 主要是所有的寄存器、signal信息以及指向 sigreturn 的系统调用指针
- 然后回到用户态, 执行对应的 signal handler
- signal handler 完成之后执行 sigreturn 系统调用, 为进程恢复上下文信息

攻击面在上下文信息被保存在栈上, 我们是有机会对它进行读写操作的, 并且内核在恢复上下文的时候, 也并不会对栈上的信息做检验, 所以 sigframe 可以被伪造.

## PIN

插桩框架. 除了可以使用 python 封装好的 pintool2.py | pintool.py 调用 API, 我们也可以自己写 pintool 实现我们自己想要的功能.

参考:

[CTF-All-in-One](https://firmianay.gitbooks.io/ctf-all-in-one/content/doc/5.2.1_pin.html)

[官网](https://software.intel.com/sites/landingpage/pintool/docs/98484/Pin/html/index.html)
