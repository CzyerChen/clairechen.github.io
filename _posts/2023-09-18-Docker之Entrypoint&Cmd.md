---
layout:     post
title:      Docker| ENTRYPOINT & CMD 的区别
subtitle:   Docker指令
date:       2023-09-18
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Docker
---

## Docker配置文件中，ENTRYPOINT & CMD 有什么区别和联系

ENTRYPOINT 指令和 CMD指令 均可以表示容器启动后要执行的命令

ENTRYPOINT 是镜像固有的，外部不可修改，类似硬编码，适合定义初始化指令

CMD 是镜像发布到容器时可修改的，内部编码作为一个默认值，适合定义执行类指令

两者在Dockerfile中可以单独存在，也可以组合存在

```bash
ENTRYPOINT [command]
CMD [command]
```

其中command可以为 shell指令形式和exec指令形式

- shell指令 根据原理，会在当前父shell中创建一个子shell进程，在子shell进程中执行sh脚本，执行完毕回到父shell
  - 直接使用shell指令，由于会生成一个子进程shell来处理指令，如果指令不能正常结束回到父shell，则外部ctrl+c也无法终止子进程
- exec指令，根据原理，会在当前shell的command进程中，执行sh脚本，执行完毕停留在command进程中
  - exec指令 command进程可以通过 ctrl+c 退出进程，回到父进程

ENTRYPOINT & CMD 的指令后面

- 如果是shell指令，直接跟指令
- 如果是exec指令，使用 ['executable','param1','param2'] 方式进行定义

[参考文档](https://zhuanlan.zhihu.com/p/30555962)