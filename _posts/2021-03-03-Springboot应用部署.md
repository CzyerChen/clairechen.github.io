---
layout:     post
title:      SpringBoot jar包优雅的部署方式
subtitle:   SpringBoot jar Systemd
date:       2021-02-23
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringBoot
    - jar
    - Systemd
---

- centos版本：CentOS Linux release 7.6.1810 (Core)

## 一、SpringBoot jar包的部署方式

1. nohup 后台进程形式
2. Linux脚本 启动形式
3. systemd 优雅系统服务形式，systemd是System V init系统的继承者，现在被许多现代Linux发行版使用

今天主要展开的是第三种：systemd 优雅系统服务形式

## 二、Systemd形式，优雅部署SpringBoot项目

### 1. 整理文件夹+jar包+外部配置文件

文件夹的目录结构可以有多种：

1. jar包 + config文件位于统一目录
2. jar包 + /config 目录下的config文件
3. 自定义文件夹目录结构形式（/app下存放jar包，/config目录下存放配置）

上面第一和第二种，主要是运用了默认的外部配置文件的加载顺序，可以省去指定配置文件的指定步骤

运用第三种模式，一方面是能够文件夹的形式更清晰整洁，另一方面也能尝试一下外部指定配置文件位置的功能

其实一般采用第一第二种形式就可以了

最终的文件夹目录结构如下：

```text
.
├── app
│   ├── web.jar
├── command
│   └── web.service
└── config
    ├── application-dev.yml
    └── application.yml

3 directories, 4 files
```

### 2. 构建应用启动脚本

这里由于指定配置文件、指定运行环境，所以需要传入两个变量：

```bash
-Dspring.config.location,-Dspring.profiles.active
```

其余按照应用情况进行参数设置

```bash
/usr/local/jdk/bin/java  -Xmx2048m -Xms2048m -Xmn1024m -Xss4m -XX:ParallelGCThreads=20 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -Dspring.config.location=/home/web/config/application.yml,/home/web/config/application-dev.yml -Dspring.profiles.active=dev  -jar /home/web/app/web.jar
```

### 3. 配置systemd脚本

vi web.service

```bash

[Unit]
Description=Web Service
After=syslog.target

[Service]
ExecStart=/usr/local/jdk/bin/java  -Xmx2048m -Xms2048m -Xmn1024m -Xss4m -XX:ParallelGCThreads=20 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -Dspring.config.location=/home/web/config/application.yml,/home/web/config/application-dev.yml -Dspring.profiles.active=dev  -jar /home/web/app/dmp-web.jar
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target

```
[以上配置相对简单，详情参看systemd脚本配置官方文档]( https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/System_Administrators_Guide/sect-Managing_Services_with_systemd-Services.html)
[systemd 总结文档](https://cloud.tencent.com/developer/article/1516125)

chmod +x ...

- 确保web.jar包有可执行权限 
- 确保web.service脚本有可执行权限

### 4. 配置软连接

ln -s  source target

centos7 systemd的目录： /usr/lib/systemd/system

```bash
ln -s /home/web/command/web.service /usr/lib/systemd/system/web.service
```

### 5. 校验服务情况，并启动

```bash
#注册为系统服务，自启动
sudo systemctl enable web.service

#其他命令
#查看状态
sudo systemctl status web.service
#启动服务
sudo systemctl start web.service
#暂停服务
sudo systemctl stop web.service
#重启服务
sudo systemctl restart web.service

#旧版本的命令：
#查看状态
sudo service web.service status
#启动
sudo service web.service start
#暂停
sudo service web.service stop
#重启
sudo service web.service restart

#如果修改了systemd文件需要重新加载
sudo systemctl daemon-reload

#查看该服务的console log
journalctl -f -u  web.service
```

如果无异常报错，配置就完成了

## 三、system部署的优点

- 可执行的jar可以其他 Unix 系统程序一样运行
- 便捷地在生成应用环境中安装和管理SpringBoot应用程序