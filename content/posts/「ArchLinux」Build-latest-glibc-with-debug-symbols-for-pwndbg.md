---
title: 「ArchLinux」Build the latest glibc with debug symbols for pwndbg
date: 2022-09-14 13:51:16
tags:
- Linux
categories:
- TECHNOLOGY
---

## Background

在 archlinux 中 [pwndbg](https://github.com/pwndbg/pwndbg) 无法正常使用 [pwngdb](https://github.com/scwuaptx/Pwngdb) 的 heap 系列命令，因为缺少 glibc 的调试符号。但问题是在 arch 的构建系统中默认会 strip 掉二进制文件的调试符号，以减小程序体积题高性能等（想过跑路 opensuse 但还是没有这样做）。为了获得更好的调试体验，就来研究一下如何优雅地解决这个问题。~~为什么 5 年过去了这个 [issue](https://github.com/pwndbg/pwndbg/issues/340) 还没有被很好地解决~~

## Build glibc

尝试自己编译一份带调试符号的 glibc 试试：

```bash
#!/bin/bash

# Install Dependencies
sudo pacman -S git svn gd lib32-gcc-libs patch make bison fakeroot devtools

# Checkout glibc source
svn checkout --depth=empty svn://svn.archlinux.org/packages
cd packages
svn update glibc
cd glibc/repos/core-x86_64

# Add current locale to locale.gen.txt
grep -v "#" /etc/locale.gen >> locale.gen.txt

# Enable debug build in PKGBUILD
sed -i 's#!strip#debug#' PKGBUILD

# Build glibc and glibc-debug packages
makepkg -j4 --skipchecksums --config /usr/share/devtools/makepkg-x86_64.conf

# Install glibc-debug
sudo pacman -U *.pkg.tar.zst
```

这边注意可能的几个问题：

- pacman -U 安装的时候可能会报错文件已存在，全部删掉即可

- 一定要把 glibc 也一并安装上，就算版本和你本地的一致，不能只装 glibc-debug, 这样会导致 CRC mismatch 的问题

- 所有步骤完成后要把 `/etc/makepkg.conf` 中所有 OPTIONS 中的 `strip -> !strip` 以及 `!debug -> debug`

- 如果机器上有 pwndbg 需要重新 `sudo pacman -S pwndbg` 安装一份挤掉老版，记得要添加 `source /usr/share/pwndbg/gdbinit.py` 到 `~/.gdbinit`



可以愉快地断在 glibc 的源码中了：

![](https://s2.loli.net/2022/09/14/e3DKmiO9F8l7kXL.png)

当然缺点很明显，每次更新 glibc 都需要自己重新 build, 后续会写个自动化来一键完成。



## Use other plugins

当然了，gef 还是可以在没有 debug symbols 的情况下很好地查看 heap chunks, 这里给出一个切换 gdb 插件的 zsh 脚本：

```shell
#!/usr/bin/zsh

# gdbs : gdb-switcher
function gdbs() {
      echo -e "\n[*] Which debugger ?"
      echo -e "\n1 : Legacy GDB"
      echo "2 : peda"
      echo "3 : gef"
      echo "4 : pwndbg"
          echo "5 : dashboard"
      echo "6 : radare2"

      radare="no"

      read choice
      case $choice in
          1) echo "[+] gdb-switch => gdb"
             cp ~/.gdbinit-gdb ~/.gdbinit
             ;;
          2) echo "[+] gdb-switch => peda"
                 cp ~/.gdbinit-peda ~/.gdbinit
             ;;
          3) echo "[+] gdb-switch => gef"
                 cp ~/.gdbinit-gef ~/.gdbinit
             ;;
          4) echo "[+] gdb-switch => pwndbg"
                 cp ~/.gdbinit-pwndbg ~/.gdbinit
             ;;
          5) echo "[+] gdb-switch => dashboard"
                 cp ~/.gdbinit-dashboard ~/.gdbinit
             ;;
          6) echo "[+] gdb-switch => radare2"
                  radare="run"
             ;;
      esac

      if [ "$#" -eq 1 ]; then
         if [ "$radare" = "run" ]; then
            echo "[+] debugger execution : radare2"
            r2 $1
         else
            echo "[+] debugger execution"
            gdb -q $1
         fi
      fi
}
```

放在 `~/.oh-my-zsh/<some dir>/` 下，并且在 `~/.zshrc` 中添加 `autoload -Uz gdbs` 即可在启动 shell 时自动加载，记得改变一下路径。



## TODO

- [x] auto build glibc

- [ ] maybe other efficient solutions
