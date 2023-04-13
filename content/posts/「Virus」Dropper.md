---
title: "「Virus」Dropper"
date: 2023-04-06T13:23:36+08:00
draft: true
---


<!-- more -->

## Overview

一台 50 块收的电脑：

![iIRi5k.jpeg](https://i.328888.xyz/2023/04/06/iIRi5k.jpeg)

## 一阶段

![iIRnXy.png](https://i.328888.xyz/2023/04/06/iIRnXy.png)

好家伙，开幕创建恶臭文件夹 114514，看样子不是什么好东西。进 `sub_14000151E` 一探究竟，这里可以看到我的注释，即该函数做了一些 [UAC]^(User Account Control) 的操作，具体如下：

![iIR03Z.png](https://i.328888.xyz/2023/04/06/iIR03Z.png)

置 0 了一些注册表项，分别是：

- `ConsentPromptBehaviorAdmin`：置 0 后将不会在请求管理员权限时弹出询问窗口
- `EnableLUA`：置 0 后将不会在程序试图安装或改变系统状态时弹出提示
- `PromptOnSecureDesktop`：置 0 会禁用安全桌面提示

好家伙，全是高危操作啊，可以认为其为 KILL-CHAIN 七阶段中 Exploitation 的初始部分，即构成前置条件。

接下来最引人注目的莫过于我重命名之后的 wget 函数，即利用 [HFS]^(HTTP File Server) C2 技术从远程服务器上下载 `liulian.exe`，用于接下来释放 shellcode，浏览器访问 IP 地址可以看到：

![iIcz7C.png](https://i.328888.xyz/2023/04/06/iIcz7C.png)

全是好东西啊我去！从接下来的代码看到，程序使用 `ShellExecuteA` 函数的 `open` 操作执行了 `liulian.exe`，应该会在这里释放 shellcode。那么先中断一下对主程序的分析，去看一下载荷的释放过程。

## 二阶段

初步可以断定是一个 C++ 的 Dropper 主函数就是这样了：

![iImVEU.png](https://i.328888.xyz/2023/04/06/iImVEU.png)

和一个开源的 [httpdropper](https://github.com/0xjbb/httpdropper) 项目非常相似，先去下载一下 shellcode：

[![iImceV.png](https://i.328888.xyz/2023/04/06/iImceV.png)](https://imgloc.com/i/iImceV)

大概率被加密了，即刻打开虚拟机调试：

发现会在内存中释放一个 C# 后门，也是开源项目，不赘述了。