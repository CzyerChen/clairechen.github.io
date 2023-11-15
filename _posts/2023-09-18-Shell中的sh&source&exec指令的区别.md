---
layout:     post
title:      Shell | sh & source & exec 的区别
subtitle:   Shell指令
date:       2023-09-18
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Shell
---

## sh

在当前父进程shell中新建一个子进程shell

使用子进程执行sh脚本

可以通过echo $$ 打印当前线程，执行结束后销毁子进程，回到父进程

## source

在当前进程shell中执行

执行完成sh脚本后，仍然在当前shell，执行目录跳到tmp

## exec

在当前进程的command进程中，执行完成sh脚本后，停留在command进程中

需要手动ctrl+c 退出才可回到当前进程
