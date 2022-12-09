---
layout:     post
title:      K8S部署Jenkins
subtitle:   k8s、Jenkins
date:       2022-08-20
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - K8S
    - Jenkins
---


> MacOS
> 
> Docker version：20.10.17(初次尝试的时候版本是18.09.2，最终还是遇到的适配的问题，后来升级了版本）
> 
> Jenkins： 11.4(拉取的是lts版本，当前具体版本号为11.4)


## 获取镜像

为了在deploy的时候太长的时间花费在拉取镜像（甚至容易失败的情况），可以提前准备好镜像资源

这边是本地搭建测试，选取了最新版，选择具体版本只要在后面加上指定的版本号即可

```bash
# 查看到jenkins的镜像
$ docker search jenkins
NAME                               DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
jenkins                            DEPRECATED; use "jenkins/jenkins:lts" instead   5534                [OK]                
jenkins/jenkins                    The leading open source automation server       3183                                    
jenkins/jnlp-slave                 a Jenkins agent which can connect to Jenkins…   151                                     [OK]
jenkins/inbound-agent                                                              75                                      
bitnami/jenkins                    Bitnami Docker Image for Jenkins                54                                      [OK]
jenkins/slave                      base image for a Jenkins Agent, which includ…   48                                      [OK]
jenkins/agent                                                                      44                                      
jenkins/ssh-slave                  A Jenkins slave using SSH to establish conne…   38                                      [OK]

# 拉取Jenkins的最新版镜像
$ docker pull jenkins/jenkins:lts
```

查看镜像，确定下载的确切版本

```bash
docker images
```

