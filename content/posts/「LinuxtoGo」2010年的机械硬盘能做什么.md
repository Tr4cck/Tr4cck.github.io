---
title: "「LinuxtoGo」2010年的机械硬盘能做什么"
date: 2023-03-19T02:46:44+08:00
tags:
- Linux
- CS
categories:
- TECHNOLOGY
---


<!-- more -->

## Overview

如题.

![ix4AXa.jpeg](https://i.328888.xyz/2023/04/14/ix4AXa.jpeg)

![ix44Cw.jpeg](https://i.328888.xyz/2023/04/14/ix44Cw.jpeg)

如何在没有环境强依赖性的编程上机试验中, 仅仅靠学校机房里的老爷机顺利地生存下来? 你可能在找 `linux to go`: 在 U 盘中安装的 linux 操作系统.

## Format

作为一个忠实的 `Arch Linux` 用户, 我选择了 `Arch Linux` 作为我的 `linux to go` 系统. 首先用 https://archlinux.org/download/ 提供的各种你喜欢的方式获取最新的系统镜像, 然后使用 `dd` 命令将其写入 U 盘中.

```shell
❯ file archlinux-2023.04.01-x86_64.iso 
archlinux-2023.04.01-x86_64.iso: data

❯ dd if=archlinux-2020.03.01-x86_64.iso of=/dev/sdb bs=4M status=progress

❯ sync # flush cache
```

## Installation

从 U 盘启动系统, 并**仔细**阅读 https://wiki.archlinux.org/index.php/Installation_guide 中的步骤安装系统. 其中, 你需要注意的是:

- 在 `/etc/fstab` 中, 我们需要使用 UUID 而不是 label 来指定分区, 以保证在不同机器上的可移植性.

- 在执行 `grub-install` 时, 你需要指定 `--removable` 参数, 以便在 U 盘中安装 `grub`. (血一样的教训, 不这样做的话在重新拔插 U 盘后会无法识别启动盘)
  
> 其实在 GNU GRUB Manual 中明确指出了这一点:
> 
> For removable installs you have to use --removable and specify both --boot-directory and --efi-directory:
> 
> `# grub-install --efi-directory=/mnt/usb --boot-directory=/mnt/usb/boot --removable`

## FIN

[![iguG6J.png](https://i.328888.xyz/2023/04/13/iguG6J.png)](https://imgloc.com/i/iguG6J)

[![iguFjt.png](https://i.328888.xyz/2023/04/13/iguFjt.png)](https://imgloc.com/i/iguFjt)