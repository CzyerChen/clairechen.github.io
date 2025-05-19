---
layout:     post
title:      Tesseract-轻松实现图片转文字

subtitle:   Tesseract OCR 
date:       2025-02-18
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Tesseract
    - OCR
---

Tesseract 是一款开源的 光学字符识别（OCR）引擎，由 HP 实验室于 1985 年开发，2005 年由 Google 接手维护并开源。它能够从图像中提取文字，支持 100+ 种语言，是当前最流行的 OCR 工具之一。

特点：

- 支持 Windows、macOS、Linux 等操作系统。
- 提供命令行工具和 API，方便集成到其他程序（如 Python、Java、C++ 等）
- 内置主流语言（英语、中文、日语等），支持通过训练数据扩展小众语言。
- 支持 垂直文本 和 复杂排版（如多列文档）
- 从 Tesseract 4.0 开始，引入基于 LSTM（长短期记忆网络） 的 OCR 引擎，显著提升了识别精度
- 允许用户训练自定义模型，优化特定场景（如票据、车牌、手写体等）的识别效果

使用场景就可以非常广泛，可以做PDF扫描件的数据提取、发票数据数字化、号牌识别、名片识别等，相比类似有25K star EasyOCR来说，准确性相对高一些(EasyOCR的在线试用真的很不满意)。

- [如何安装](#如何安装)
  - [CentOS7](#centos7)
  - [MacOS](#macos)
- [验证与命令行测试](#验证与命令行测试)
- [Python \& Java 调用](#python--java-调用)
  - [Java](#java)
    - [期间遇到的问题有： MacOS + IDEA 程序找不到 tesseract](#期间遇到的问题有-macos--idea-程序找不到-tesseract)
  - [Python](#python)

## 如何安装

以下主要是CentOS7 、MacOS 的安装方式，其他[更多Linux版本的安装详见官方](https://github.com/tesseract-ocr/tesseract)，源码包还是编译包安装的形式也可自选。

主要是安装tesseract和tesseract-lang两部分，在ubuntu可采用我们熟悉的apt的方式

### CentOS7

CentOS上，就采用yum的方式安装

1. 添加仓库地址

```bash
$ yum-config-manager --add-repo https://download.opensuse.org/repositories/home:/Alexander_Pozdnyakov/CentOS_7/
已加载插件：fastestmirror
adding repo from: https://download.opensuse.org/repositories/home:/Alexander_Pozdnyakov/CentOS_7/

[download.opensuse.org_repositories_home_Alexander_Pozdnyakov_CentOS_7_]
name=added from: https://download.opensuse.org/repositories/home:/Alexander_Pozdnyakov/CentOS_7/
baseurl=https://download.opensuse.org/repositories/home:/Alexander_Pozdnyakov/CentOS_7/
enabled=1
```

2. 刷新仓库链接

```bash
$ sudo rpm --import https://build.opensuse.org/projects/home:Alexander_Pozdnyakov/signing_keys/download?kind=gpg
$ yum update
已加载插件：fastestmirror
Determining fastest mirrors
epel/x86_64/metalink                                                                                                                                                                 | 5.1 kB  00:00:00     
 * base: mirrors.aliyun.com
 * epel: ftp-stud.hs-esslingen.de
 * extras: mirrors.aliyun.com
 * remi-php74: ftp.riken.jp
 * remi-safe: ftp.riken.jp
 * updates: mirrors.aliyun.com
base                                                                                                                                                                                 | 3.6 kB  00:00:00     
download.opensuse.org_repositories_home_Alexander_Pozdnyakov_CentOS_7_                                                                                                               | 1.3 kB  00:00:00     
extras                                                                                                                                                                               | 2.9 kB  00:00:00     
irontec                                                                                                                                                                              | 2.5 kB  00:00:00     
mysql-connectors-communi
......

```

3. 获取安装包

```bash
$ yum install tesseract
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * epel: ftp-stud.hs-esslingen.de
 * extras: mirrors.aliyun.com
 * remi-php74: ftp.riken.jp
 * remi-safe: ftp.riken.jp
 * updates: mirrors.aliyun.com
正在解决依赖关系
--> 正在检查事务
---> 软件包 tesseract.x86_64.0.4.1.3+git4271-3.1 将被 安装
--> 正在处理依赖关系 tesseract-langpack-osd >= 3.99，它被软件包 tesseract-4.1.3+git4271-3.1.x86_64 需要
--> 正在处理依赖关系 tesseract-langpack-eng >= 3.99，它被软件包 tesseract-4.1.3+git4271-3.1.x86_64 需要
--> 正在处理依赖关系 liblept.so.5()(64bit)，它被软件包 tesseract-4.1.3+git4271-3.1.x86_64 需要
--> 正在检查事务
---> 软件包 leptonica.x86_64.0.1.76.0-2.5 将被 安装
--> 正在处理依赖关系 libwebp.so.4()(64bit)，它被软件包 leptonica-1.76.0-2.5.x86_64 需要
---> 软件包 tesseract-langpack-eng.noarch.0.4.00~git30-5.5 将被 安装
---> 软件包 tesseract-langpack-osd.noarch.0.4.00~git30-5.5 将被 安装
--> 正在检查事务
---> 软件包 libwebp.x86_64.0.0.3.0-11.el7 将被 安装
--> 解决依赖关系完成

依赖关系解决

============================================================================================================================================================================================================
 Package                                   架构                      版本                                   源                                                                                         大小
============================================================================================================================================================================================================
正在安装:
 tesseract                                 x86_64                    4.1.3+git4271-3.1                      download.opensuse.org_repositories_home_Alexander_Pozdnyakov_CentOS_7_                     10 M
为依赖而安装:
 leptonica                                 x86_64                    1.76.0-2.5                             download.opensuse.org_repositories_home_Alexander_Pozdnyakov_CentOS_7_                    1.0 M
 libwebp                                   x86_64                    0.3.0-11.el7                           updates                                                                                   170 k
 tesseract-langpack-eng                    noarch                    4.00~git30-5.5                         download.opensuse.org_repositories_home_Alexander_Pozdnyakov_CentOS_7_                    1.6 M
 tesseract-langpack-osd                    noarch                    4.00~git30-5.5                         download.opensuse.org_repositories_home_Alexander_Pozdnyakov_CentOS_7_                    3.4 M

事务概要
============================================================================================================================================================================================================
安装  1 软件包 (+4 依赖软件包)

总下载量：17 M
安装大小：55 M
Is this ok [y/d/N]: y
Downloading packages:
(1/5): libwebp-0.3.0-11.el7.x86_64.rpm                                                                                                                                               | 170 kB  00:00:00     
(2/5): leptonica-1.76.0-2.5.x86_64.rpm                                                                                                                                               | 1.0 MB  00:01:05     
(3/5): tesseract-langpack-eng-4.00~git30-5.5.noarch.rpm                                                                                                                              | 1.6 MB  00:01:45     
(4/5): tesseract-4.1.3+git4271-3.1.x86_64.rpm                                                                                                                                        |  10 MB  00:04:30     
(5/5): tesseract-langpack-osd-4.00~git30-5.5.noarch.rpm                                                                                                                              | 3.4 MB  00:03:28     
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
总计                                                                                                                                                                         45 kB/s |  17 MB  00:06:20     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : tesseract-langpack-eng-4.00~git30-5.5.noarch                                                                                                                                            1/5 
  正在安装    : libwebp-0.3.0-11.el7.x86_64                                                                                                                                                             2/5 
  正在安装    : leptonica-1.76.0-2.5.x86_64                                                                                                                                                             3/5 
  正在安装    : tesseract-langpack-osd-4.00~git30-5.5.noarch                                                                                                                                            4/5 
  正在安装    : tesseract-4.1.3+git4271-3.1.x86_64                                                                                                                                                      5/5 
  验证中      : tesseract-4.1.3+git4271-3.1.x86_64                                                                                                                                                      1/5 
  验证中      : tesseract-langpack-osd-4.00~git30-5.5.noarch                                                                                                                                            2/5 
  验证中      : libwebp-0.3.0-11.el7.x86_64                                                                                                                                                             3/5 
  验证中      : tesseract-langpack-eng-4.00~git30-5.5.noarch                                                                                                                                            4/5 
  验证中      : leptonica-1.76.0-2.5.x86_64                                                                                                                                                             5/5 

已安装:
  tesseract.x86_64 0:4.1.3+git4271-3.1                                                                                                                                                                      

作为依赖被安装:
  leptonica.x86_64 0:1.76.0-2.5             libwebp.x86_64 0:0.3.0-11.el7             tesseract-langpack-eng.noarch 0:4.00~git30-5.5             tesseract-langpack-osd.noarch 0:4.00~git30-5.5            

完毕！
```

继续安装lang包

```bash
$ yum install tesseract-langpack-deu
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * epel: ftp-stud.hs-esslingen.de
 * extras: mirrors.aliyun.com
 * remi-php74: ftp.riken.jp
 * remi-safe: ftp.riken.jp
 * updates: mirrors.aliyun.com
正在解决依赖关系
--> 正在检查事务
---> 软件包 tesseract-langpack-deu.noarch.0.4.00~git30-5.5 将被 安装
--> 解决依赖关系完成

依赖关系解决

============================================================================================================================================================================================================
 Package                                    架构                       版本                                源                                                                                          大小
============================================================================================================================================================================================================
正在安装:
 tesseract-langpack-deu                     noarch                     4.00~git30-5.5                      download.opensuse.org_repositories_home_Alexander_Pozdnyakov_CentOS_7_                     763 k

事务概要
============================================================================================================================================================================================================
安装  1 软件包

总下载量：763 k
安装大小：1.5 M
Is this ok [y/d/N]: y
Downloading packages:
tesseract-langpack-deu-4.00~git30-5.5.noarch.rpm                                                                                                                                     | 763 kB  00:00:03     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : tesseract-langpack-deu-4.00~git30-5.5.noarch                                                                                                                                            1/1 
  验证中      : tesseract-langpack-deu-4.00~git30-5.5.noarch                                                                                                                                            1/1 

已安装:
  tesseract-langpack-deu.noarch 0:4.00~git30-5.5                                                                                                                                                            

完毕！
```

### MacOS

在MacOS中更为简单，同样是更新仓库，拉取安装包安装

```bash
$ brew install tesseract
==> Auto-updating Homebrew...
Adjust how often this is run with HOMEBREW_AUTO_UPDATE_SECS or disable with
HOMEBREW_NO_AUTO_UPDATE. Hide these hints with HOMEBREW_NO_ENV_HINTS (see `man brew`).
==> Auto-updated Homebrew!
Updated 3 taps (homebrew/services, homebrew/core and homebrew/cask).
==> New Formulae
adapterremoval    bagels            feluda            gowall            identme           koji              mac               pdfly             sby               threatcl          yices2
aqua              bazel@7           ggh               havener           jsrepo            lazyjj            md2pdf            ratarmount        soft-serve        vfkit             yor
arelo             code2prompt       git-mob           hk                jupytext          libpostal         mummer            reuse             sql-formatter     wfa2-lib          zimfw
bacon-ls          dockerfilegraph   go@1.23           hl                keeper-commander  libpostal-rest    nping             rustywind         symfony-cli       yamlfix
==> New Casks
autogram                          font-big-shoulders                font-comic-relief                 losslessswitcher                  pinwheel                          ui-tars
badgeify                          font-big-shoulders-inline         granola                           luanti                            precize                           vezer
browser-actions                   font-big-shoulders-stencil        kunkun                            mitti                             structuredlogviewer

You have 15 outdated formulae installed.

```

继续安装

```bash
$ brew install tesseract-lang
==> Downloading https://formulae.brew.sh/api/formula.jws.json
==> Downloading https://formulae.brew.sh/api/cask.jws.json
==> Fetching tesseract-lang
==> Downloading https://mirrors.aliyun.com/homebrew/homebrew-bottles/tesseract-lang-4.1.0.all.bottle.2.tar.gz
##################################################################################################################################################################################################### 100.0%
==> Pouring tesseract-lang-4.1.0.all.bottle.2.tar.gz
🍺  /usr/local/Cellar/tesseract-lang/4.1.0: 165 files, 654.0MB
==> Running `brew cleanup tesseract-lang`...
Disable this behaviour by setting HOMEBREW_NO_INSTALL_CLEANUP.
Hide these hints with HOMEBREW_NO_ENV_HINTS (see `man brew`).
```

## 验证与命令行测试

使用命令确认下是否安装成功

```bash
$ tesseract -v
tesseract 5.5.0
 leptonica-1.85.0
  libgif 5.2.2 : libjpeg 8d (libjpeg-turbo 3.0.4) : libpng 1.6.46 : libtiff 4.7.0 : zlib 1.2.11 : libwebp 1.5.0 : libopenjp2 2.5.3
 Found AVX2
 Found AVX
 Found FMA
 Found SSE4.1
 Found libarchive 3.7.7 zlib/1.2.11 liblzma/5.6.3 bz2lib/1.0.8 liblz4/1.10.0 libzstd/1.5.6
 Found libcurl/8.7.1 SecureTransport (LibreSSL/3.3.6) zlib/1.2.12 nghttp2/1.63.0
```

使用命令行测试基础功能，是否能够实现图片转文字，可以自备几张英文字、中文字、中英文结合的图片，尽量不要网页那样过于复杂。

```bash
# tesseract demo.png  stdout -l eng
prod_lua.lua

# tesseract demo2.png  stdout -l eng
org.apache.zookeeper.server.quorum.QuorumPeerMain
org.elasticsearch. hootstrap.Elasticsearch

# tesseract demo3.png  stdout -l chi_sim
管 理 平 台 更 新 说 明 :
) 手 机 地 区 过 源 支 持

2) 增 加 相 关 字 段 , 丰 富 统 计 内 容
```

## Python & Java 调用

### Java

Tess4J是一款开源的、易于使用的Java API封装，提供了对Tesseract OCR引擎的便捷调用。它使得开发者能够在Java应用程序中轻松实现光学字符识别功能。Tess4J的开发使得Tesseract能够更加贴近Java社区，降低学习曲线，并且为Java开发者提供了一个强大的文字识别工具。

java需要调用主要是服务可以访问到 tesseract 命令、java服务中明确work dir(需要包含tesseract)、maven 引入tess4j

实际调用的方式十分简答

```java
 public static void main(String[] args) throws TesseractException {
        ITesseract iTesseract = new Tesseract();
        iTesseract.setLanguage("chi_sim");
        iTesseract.setDatapath("/files/spring-ai-demo/src/main/resources/tessdata");
        File img = new File("/Users/admin/Downloads/demo.png");
        String ocrResult = iTesseract.doOCR(img);
        System.out.println("识别结果: \n" + ocrResult);
    }
```

- language: ocr识别的库，命令行 -l 可选eng/chi_sim/以及trainedata中包含的语种
- datapath: 指向语言traineddata的目录，需要去搜寻匹配的语言
- doOCR: 执行识别

#### 期间遇到的问题有： MacOS + IDEA 程序找不到 tesseract 

```log
Suppressed: java.lang.UnsatisfiedLinkError: dlopen(/Library/Frameworks/tesseract.framework/tesseract, 0x0009): tried: '/Library/Frameworks/tesseract.framework/tesseract' (no such file), '/System/Volumes/Preboot/Cryptexes/OS/Library/Frameworks/tesseract.framework/tesseract' (no such file)
```

通过启动指定work directory解决，Run -> Edit Configurration -> Environment: Work Directory

在系统中查询到 tesseract 的路径

```bash
$ whereis tesseract
tesseract: /usr/local/bin/tesseract
```

将 `/usr/local/bin/` 填入 `Work Directory`

启动后即可执行 OCR。

### Python

在AI领域，Python基本是排名第一的，Python有丰富的lib包可以支持这样的功能

确认本地pip和python环境，我这边使用pip3和python3

1、引入依赖：

```bash
$ pip3 install pillow
Requirement already satisfied: pillow in /usr/local/lib/python3.11/site-packages (11.1.0)

[notice] A new release of pip is available: 24.3.1 -> 25.0.1
[notice] To update, run: python3.11 -m pip install --upgrade pip

$ pip3 install pytesseract
Collecting pytesseract
  Downloading pytesseract-0.3.13-py3-none-any.whl.metadata (11 kB)
Requirement already satisfied: packaging>=21.3 in /usr/local/lib/python3.11/site-packages (from pytesseract) (23.2)
Requirement already satisfied: Pillow>=8.0.0 in /usr/local/lib/python3.11/site-packages (from pytesseract) (11.1.0)
Downloading pytesseract-0.3.13-py3-none-any.whl (14 kB)
Installing collected packages: pytesseract
Successfully installed pytesseract-0.3.13

[notice] A new release of pip is available: 24.3.1 -> 25.0.1
[notice] To update, run: python3.11 -m pip install --upgrade pip
```

2、编写脚本：

```py
# USAGE
# python ocr.py --image images/example_01.png 
# python ocr.py --image images/example_02.png  --preprocess blur

# import the necessary packages
from PIL import Image
import pytesseract
import argparse
import cv2
import os

# construct the argument parse and parse the arguments
ap = argparse.ArgumentParser()
ap.add_argument("-i", "--image", required=True,
	help="path to input image to be OCR'd")
ap.add_argument("-p", "--preprocess", type=str, default="thresh",
	help="type of preprocessing to be done")
args = vars(ap.parse_args())

# load the example image and convert it to grayscale
image = cv2.imread(args["image"])
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)


# check to see if we should apply thresholding to preprocess the
# image
if args["preprocess"] == "thresh":
	gray = cv2.threshold(gray, 0, 255,
		cv2.THRESH_BINARY | cv2.THRESH_OTSU)[1]

# make a check to see if median blurring should be done to remove
# noise
elif args["preprocess"] == "blur":
	gray = cv2.medianBlur(gray, 3)

# write the grayscale image to disk as a temporary file so we can
# apply OCR to it
filename = "{}.png".format(os.getpid())
cv2.imwrite(filename, gray)

# load the image as a PIL/Pillow image, apply OCR, and then delete
# the temporary file
text = pytesseract.image_to_string(Image.open(filename),lang='chi_sim+eng')
os.remove(filename)
print(text)
```

3、执行脚本

```bash
$ python3 ocr.py --image demo.png 
i 小白入门DeepSeek必备的50个高阶提示词.zip
```
