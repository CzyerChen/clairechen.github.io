---
layout:     post
title:      Codespaces
subtitle:   Github
date:       2023-09-13
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kubernetes
---

- [如何丝滑地白嫖一个本地开发环境？](#如何丝滑地白嫖一个本地开发环境)
- [怎么新建一个代码空间？](#怎么新建一个代码空间)
	- [1：通过Github网页新建](#1通过github网页新建)
	- [2：通过VSCode插件新建](#2通过vscode插件新建)
- [为代码创建相应的开发测试环境](#为代码创建相应的开发测试环境)

## 如何丝滑地白嫖一个本地开发环境？

使用Codespaces为开发者解决这样的痛点：

- 为项目设置和维护一个或一组开发工作站。
- 在“第一次提交”发生之前浪费的时间。
- 开发工作站之间的配置/工具/设置不一致。
- 版本控制工具/扩展、调试器和依赖项。
- 基于个人或团队的设置和自定义。
- 安全和漏洞。
- 硬件规格要求。

## 怎么新建一个代码空间？

### 1：通过Github网页新建

![在这里插入图片描述](https://img-blog.csdnimg.cn/e43a939db6194a8cb54a41b3e256c5f0.png)

- 首先New Codespaces
- 通过四个选择开启一个空间
  - Repository - 选择一个自有仓库新建
    To be cloned into your codespace
  - Branch - 选择仓库内某一个分支新建
    This branch will be checked out on creation
  - Region - 选择一个所在的地区
    Your codespace will run in the selected region
  - Machine type - 选择取用的资源(2core/8G/32G,4core/16G/32G)
    Resources for your codespace
- 点击Create Codespace即可创建

创建后稍等一段时间，就可以连接所选资源的远程服务器

### 2：通过VSCode插件新建

- 通过扩展搜索Github Codespaces插件，选择安装
![在这里插入图片描述](https://img-blog.csdnimg.cn/98224f182b9944368d21c92ef77dd636.png)
- 安装成功后，左侧有一个远程资源管理器
![在这里插入图片描述](https://img-blog.csdnimg.cn/70563b488ed24ab6b7230a8231051564.png)
- 如没有新建过空间，会有一个New的按钮。如已创建，右上角会有多余一个加号
  - 点击后进入第一步，选择项目 select a repo to create your codespaces
  - 选择项目后进入第二步，选择分支 select the branch you'd like to use for the codespaces
  - 随后，地区和空间资源选择后即可创建成功

## 为代码创建相应的开发测试环境

根据Codespaces的设计初衷，就是希望边coding边testing，不需要本地搭建繁重又不一的测试环境，开发环境即服务。

那么Codespaces当前面板，你同时可以看到代码，同时也可以看到远程服务器，操作远程可以像本地开发一样地方便，那么远程的环境如何配置的和本地一样呢？

- 通过唤起指令，输入”Add Dev Containers Configuration Files...“，进入一步步地操作引导流程
- 首先选择需要的组件，比如我们本地会安装JDK、maven、node、docker、kubectl、minikube等，都可以在下拉列表中找到，以上基本也就是常规开发所需的本地环境了
- 勾选后，会要求对每一个版本进行选择，比如JDK选17，maven选3.6.3，node选16.14.0，docker选latest，minikube选latest 一，切版本都落实后，就是稍作等待，根据所选择组件，会在空间容器中构建，你需要做的就是等待...

一般完成后会跳至终端terminal，就是我们常见的服务器上面的命令行界面，这个时候你可以通过 java -version这些验证本地环境。

一切都是经过初期考验的，所以只要不是自己选的版本跑自己的服务兼容性上本来就有问题，你的服务打包后，就可以顺利地在Codespaces中跑起来，一切就和本地测试一样地丝滑。
