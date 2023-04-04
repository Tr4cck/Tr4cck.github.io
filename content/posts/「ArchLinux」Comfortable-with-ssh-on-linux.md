---
title: 「ArchLinux」Comfortable with ssh on linux
date: 2022-10-04 22:34:05
draft: true
tags:
- Linux
categories:
- TECHNOLOGY
---


<!-- more -->

## Overview

我的 ssh 处于一种诡异的情况: 即使把 `pubkey` 放到远程服务器上, 仍会在 ssh 请求连接时向你询问 `gen key pair` 时的密钥. 并且我的 `ssh-agent` 始终处于

## Reference

https://github.com/capocasa/systemd-user-pam-ssh

https://wiki.archlinux.org/title/SSH_keys
