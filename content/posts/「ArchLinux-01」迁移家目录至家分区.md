---
title: 「ArchLinux」迁移家目录至家分区
mathjax: true
toc: true
date: 2021-10-18 19:57:11
tags: Linux
categories: TECHNOLOGY
---


## 工作环境

```
                   -`                    track@Track-Arch
                  .o+`                   ----------------
                 `ooo/                   OS: Arch Linux x86_64
                `+oooo:                  Host: 82AX Lenovo Legion Y7000P2020H
               `+oooooo:                 Kernel: 5.14.12-arch1-1
               -+oooooo+:                Uptime: 31 mins
             `/:-:++oooo+:               Packages: 1681 (pacman)
            `/++++/+++++++:              Shell: fish 3.3.1
           `/++++++++++++++:             Resolution: 1920x1080
          `/+++ooooooooooooo/`           DE: Plasma 5.23.0
         ./ooosssso++osssssso+`          WM: KWin
        .oossssso-````/ossssss+`         WM Theme: Breeze 微风
       -osssssso.      :ssssssso.        Theme: Layan [Plasma], Breeze [GTK2/3]
      :osssssss/        osssso+++.       Icons: Fluent [Plasma], Fluent [GTK2/3]
     /ossssssss/        +ssssooo/-       Terminal: st
   `/ossssso+/:-        -:/+osssso+-     Terminal Font: JetBrainsMono Nerd Font
  `+sso+:-`                 `.-/+oso:    CPU: Intel i7-10875H (16) @ 5.100GHz
 `++:.                           `-/+/   GPU: NVIDIA GeForce RTX 2060 Mobile
 .`                                 `/   GPU: Intel CometLake-H GT2 [UHD Graphics]
                                         Memory: 4282MiB / 15868MiB
                                         GPU Driver: i915
                                         CPU Usage: 6%
                                         Disk (/): 44G / 457G (11%)
                                         Battery0: 100% [Full]
```

## 为什么要使用家分区代替家目录

众所周知 `ArchLinux` 是一个不太稳定的系统，如果你没有很好的备份你的操作系统的话，可能在某次 `pacman -Syyu` 之后你的系统就挂掉了。那么家分区的创建可以很好的避免重装系统带来的痛苦，因为你的所有软件的配置文件等等主要文件基本都放在了家目录下，如果有一个家分区，那么我们就不需要完全重建分区标并格式化，只需把家分区挂载到家目录，就可以有一个几乎无差别的体验。

## 如何迁移

### 迁移之前

- [ ] 当然你最好有一块儿新的硬盘，这会给你带来很大的便利
- [ ] 确保你知道关于 `Linux` 的一些基本知识，包括但不限于挂载、分区表、格式化之类的

### 开始迁移

首先使用 `sudo fdisk -l` 查看你要用来充当家分区的那个设备：

![](https://i.loli.net/2021/10/18/oXnyCasSFpEmqvi.png)

比如说我的是 `/dev/nvme1n1` 这个设备。接着使用 `sudo fdisk /dev/nvme1n1` 进入 `fdisk`，然后先 `g`，接着 `n` ，一路回车表示使用默认值，最后 `w` 即可写入。

不要忘记使用 `sudo fdisk -l` 确保正确，接着你可能需要重启一下你的电脑，否则无法进行格式化，如果你的系统还在使用这个设备的话。

格式化使用 `sudo mkfs.ext4 /dev/nvme1n1p1`。

**注意：**格式化的是分区而不是设备！

之后就是挂载了，我们使用 `sudo mount /dev/nvme1n1p1 /mnt` 将设备挂载到 `/mnt` 目录，可以选择删除 `/mnt/lost+found`。

然后将家目录整个复制过来，`sudo cp -rp /home/* /mnt`，并 `sudo mv /home /home.orig` 同时创建新的家目录 `/home` 。

接着 `cd /` 避免挂载失败，然后更改挂载点 ：

 ```bash
 sudo umount /dev/nvme1n1p1
 sudo mount /dev/nvme1n1p1 /home
 ```

随后测试一下 `df /dev/nvme1n1p1` 这个命令帮助我们查看设备的使用情况和挂载情况。

### 开机挂载

上面基本已经算完成了，但是还有最关键的一步：就是更改 `/etc/fstab` ，实际上这个文件记录了开机时有哪些分区以及挂载信息，`sudo vim /etc/fstab` 写入下面这一行：

```bash
/dev/nvme1n1p1    /home    ext4    defaults    0    0
```

## 迁移完成

然后重启你的系统就好了！如果确定无误，可以删除 `/home.orig`。

## 题外话

在我重启之后卡在了sddm，一度让我怀疑是迁移哪里出了问题。但是我经过查看 `sudo journalctl -b -1`，一番折腾，发现在卡住的页面居然可以打开终端！原来是 `KDE` 炸了，不得不去 `/var/cache` 里找历史版本回滚上，然后重启才恢复正常。这让我考虑更换一个窗口管理器，例如 `i3` 或者 `dwm` ，因为一个桌面环境确实增加了一些不稳定因素（虽然可能窗口管理器也可能有这种问题），不管怎么样还是想试试，之后应该也会相应的更一篇关于 `st` 以及 `dwm` 等等的配置之类的。
