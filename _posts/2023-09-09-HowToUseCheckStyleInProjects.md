---
layout:     post
title:      Java编码规范检验工具
subtitle:   Coding
date:       2023-09-09
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - 编码规范
---

- [1、关于代码编码质量](#1关于代码编码质量)
- [2、如何小成本有效管理企业内的编码规范](#2如何小成本有效管理企业内的编码规范)
  - [2.1 阿里编码规约IDE插件](#21-阿里编码规约ide插件)
  - [2.2 CheckStyle IDE插件](#22-checkstyle-ide插件)
- [3、如何在代码提交中检验规范](#3如何在代码提交中检验规范)
  - [3.1 阿里编码规约配置git precommit check](#31-阿里编码规约配置git-precommit-check)
  - [3.2 CheckStyle配置git precommit check](#32-checkstyle配置git-precommit-check)
  - [3.3 实践](#33-实践)

## 1、关于代码编码质量

关于企业内部如何管控代码规范、保证应用服务质量是一个基础性的问题，只要是有技术性要求的项目，都应该遵守一套既定的保障应用代码质量的规范。

常规的小成本而有见效的方式就是使用编辑器或者代码规范检测的方式，控制代码的提交。提交的代码再依赖组件，做人工的code review来把关。

常见的规范扫描工具：**阿里编码规约扫描**、**checkStyle**，今天主要描述这两个工具如何在日常编码过程中使用，以及如何与git配合来规范代码的提交。

## 2、如何小成本有效管理企业内的编码规范

### 2.1 阿里编码规约IDE插件

阿里编码规约扫描是一个成熟的组件，我这边使用的是IDEA
1、可以直接在插件市场中进行下载：Alibaba Java Coding Guidelines
2、如果步骤一存在困难的，可以前往github下载源码进行打包(`https://github.com/alibaba/p3c`)，最终打包成包含依赖的jar包，我这边构建了一个 `p3c-pmd-2.1.1-jar-with-dependencies.jar` ，打包后在IDEA中从本地地址导入插件

以上两种方式已经可以成功安装插件，只需要【Restart IDE】就可以生效。

重启后，在某一个类文件中右击或者在顶部菜单栏【Tools -> 阿里编码规约】下就可以发现它，点击即可触发扫描。

如果IDE性能有限，建议关闭实时检测。

### 2.2 CheckStyle IDE插件

1、与阿里编码规约类似，也是在插件市场中直接下载，重启IDE后生效
2、如果步骤一存在困难，可以下载官方打包的release版本，然后导入 --> `https://github.com/checkstyle/checkstyle/releases?page=1`。

注意checkStyle与JAVA版本的对应关系，否则启动检测你将发现提示`has been compiled by a more recent version of the Java Runtime (class file version 55.0), this version of the Java Runtime only recognizes class file versions up to 52.0`。

**JRE And JDK：Runtime of Checkstyle is limited only by minimal version or JRE.**

|Checkstyle version|JRE version|
|--|--|
|10.x|11 and above|
|7.x, 8.x, 9.x|8 and above|
|6.x|6 and above|
|5.x|5 and above|

官方文档链接：https://checkstyle.sourceforge.io/

## 3、如何在代码提交中检验规范

通用流程是一致的，两者就是在指令以及输出内容上有所差异，一般pre-commit的配置如下：
1. 在项目根目录 .git/hooks 下 找到 pre-commit.example文件，简单查阅，文档内也是会给到一定的提示。
2. `cp pre-commit.example pre-commit` 生成正式生效的文件 **pre-commit**
3. 下面主要是在这个内部书写检测逻辑，大体的一般是：
    a. 从git的提交数据中获取到修改的文件或目录，遍历文件或目录
    b. 针对每一项执行检查，具体规范具体指定，检查出修改项
    c. exitcode返回非0即不会提交代码，exitcode为0表示通过会提交代码

以下主要展示两者的demo

### 3.1 阿里编码规约配置git precommit check

指令： `java -cp ./p3c-pmd-2.1.1-jar-with-dependencies.jar net.sourceforge.pmd.PMD -d ./Task.java -R rulesets/java/ali-comment.xml,rulesets/java/ali-concurrent.xml,rulesets/java/ali-constant.xml,rulesets/java/ali-exception.xml,rulesets/java/ali-flowcontrol.xml,rulesets/java/ali-naming.xml,rulesets/java/ali-oop.xml,rulesets/java/ali-orm.xml,rulesets/java/ali-other.xml,rulesets/java/ali-set.xml -f text`

-cp 指定使用的JAR包以及启动类
-d  指定需要检测的文件或目录，多个文件使用逗号相连
-R  指定检测所需的配置文件，这些配置文件是alibaba-pmp-p3c项目下默认自带的，也可以根据语法自定义
-f  指定输出的格式类型，支持的输出形式有很多

```text
Available report formats and their configuration properties are:
   codeclimate: Code Climate integration.
   csv: Comma-separated values tabular format.
        problem - Include Problem column   default: true
        package - Include Package column   default: true
        file - Include File column   default: true
        priority - Include Priority column   default: true
        line - Include Line column   default: true
        desc - Include Description column   default: true
        ruleSet - Include Rule set column   default: true
        rule - Include Rule column   default: true
   emacs: GNU Emacs integration.
   empty: Empty, nothing.
   html: HTML format
        linePrefix - Prefix for line number anchor in the source file.
        linkPrefix - Path to HTML source.
   ideaj: IntelliJ IDEA integration.
        classAndMethodName - Class and Method name, pass '.method' when processing a directory.   default: 
        sourcePath - Source path.   default: 
        fileName - File name.   default: 
   summaryhtml: Summary HTML format.
        linePrefix - Prefix for line number anchor in the source file.
        linkPrefix - Path to HTML source.
   text: Text format.
   textcolor: Text format, with color support (requires ANSI console support, e.g. xterm, rxvt, etc.).
        color - Enables colors with anything other than 'false' or '0'.   default: yes
   textpad: TextPad integration.
   vbhtml: Vladimir Bossicard HTML format.
   xml: XML format.
        encoding - XML encoding format, defaults to UTF-8.   default: UTF-8
   xslt: XML with a XSL Transformation applied.
        encoding - XML encoding format, defaults to UTF-8.   default: UTF-8
        xsltFilename - The XSLT file name.
   yahtml: Yet Another HTML format.
        outputDir - Output directory.
```

样例：

```xml
#!/bin/sh
# claire

# From java package
# alicheck version: 2.2.1
# jdk version: 1.8

function print(){
echo "aliCheck>> $*"
}

print "Javacode stylecheck starting, please wait..."
wd=`pwd`
print "Workdir: $wd"

check_jar_path="$wd/.git/jars/p3c-pmd-2.1.1-jar-with-dependencies.jar"
check_xml_path="rulesets/java/ali-comment.xml,rulesets/java/ali-concurrent.xml,rulesets/java/ali-constant.xml,rulesets/java/ali-exception.xml,rulesets/java/ali-flowcontrol.xml,rulesets/java/ali-naming.xml,rulesets/java/ali-oop.xml,rulesets/java/ali-orm.xml,rulesets/java/ali-other.xml,rulesets/java/ali-set.xml"
check_result_file="$wd/temp"

# 清空temp文件
rm -rf $check_result_file

is_err=0
is_warn=0

path=''
for file in `git status --porcelain | sed s/^...// | grep '\.java$' | grep -v 'test'`;
do
path+="$wd/$file "
done
if [ "x${path}" != "x" ];then
  print "Check file: $path"
  re=`java -cp $check_jar_path net.sourceforge.pmd.PMD -d $path -R $check_xml_path -f text >> $check_result_file`
  err=`cat temp | grep "va:"`
  err_count=`cat temp | grep "va:" | wc -l`
  if [[ $err = *"va:"* ]];then
    print "detect error lines count: ${err_count}"
    print "${err}"
    is_err=1
  fi
fi
print "Javacode stylecheck finished. Thank you for your commit!"

rm -rf $check_result_file

if [ $is_err -ne 0 ] || [ $is_warn -ne 0 ]
then
print "Please return and fix stylecheck warnings before code commit！"
exit 1
fi

exit 0
```

### 3.2 CheckStyle配置git precommit check

指令：`java -jar ./checkstyle-8.45-all.jar  -c ./checkStyle.xml /Task.java -f text -o outputTempFile`

-jar 指定checkstyle jar包位置
-c   指定检查的配置文件 后面紧跟要检查的文件或目录
-f   指定输出的格式类型 有XML, SARIF, PLAIN
-o   指定输出检查结果的位置，默认不指定是输出到终端

样例：

```xml
#!/bin/sh
# claire

# From java package
# checkStyle version: 8.45
# jdk version: 1.8

function print(){
echo "checkStyle>> $*"
}

print "Javacode stylecheck starting, please wait..."
wd=`pwd`
print "Workdir: $wd"

check_jar_path="$wd/.git/jars/checkstyle-8.45-all.jar"
check_xml_path="$wd/.git/files/checkStyle.xml"
check_result_file="$wd/temp"

# 清空temp文件
rm -rf $check_result_file

is_err=0
is_warn=0

path=''
for file in `git status --porcelain | sed s/^...// | grep '\.java$' | grep -v 'test'`;
do
path+="$wd/$file "
done
if [ "x${path}" != "x" ];then
  print "Check file: $path"
  re=`java -jar $check_jar_path -c $check_xml_path $path -f plain -o $check_result_file`
  err=`cat temp | grep "ERROR"`
  err_count=`cat temp | grep "ERROR" | wc -l`
  warn=`cat temp | grep "WARN"`
  warn_count=`cat temp | grep "WARN" | wc -l`
  info=`cat temp`
  if [[ $err = *"ERROR"* ]];then
    print "detect error lines count: ${err_count}"
    print "${err}"
    is_err=1
  fi
  if [[ $warn = *"WARN"* ]];then
    print "detect warning lines count: ${warn_count}"
    print "${warn}"
    is_warn=1
  fi
fi
print "Javacode stylecheck finished. Thank you for your commit!"

rm -rf $check_result_file

if [ $is_err -ne 0 ] || [ $is_warn -ne 0 ]
then
print "Please return and fix stylecheck warnings before code commit！"
exit 1
fi

exit 0
```

### 3.3 实践

以上任意一款git pre commit 配置完成后

- git add .
- git commit -m "test"

提交后就会开启代码检测，如果存在拦截的逻辑，就无法继续commit

aliCheck:

```bash
git commit -m "test1"
aliCheck>> Javacode stylecheck starting, please wait...
aliCheck>> Workdir: xxxx
aliCheck>> Check file: xxxx/Task.java 
aliCheck>> detect error lines count:        4
aliCheck>> xxx/Task.java:64:       方法名【UUUSE2345789】不符合lowerCamelCase命名风格
aliCheck>> Javacode stylecheck finished. Thank you for your commit!
aliCheck>> Please return and fix stylecheck warnings before code commit！
```

checkStyle:

```bash
 git commit -m "test2"
checkStyle>> Javacode stylecheck starting, please wait...
checkStyle>> Workdir: XXX
checkStyle>> Check file: xxx/Task.java 
Checkstyle ends with 28 errors.
checkStyle>> detect error lines count:       28
checkStyle>> [ERROR] xxx/Task.java:3: Not allow chinese character ! [RegexpSingleline]
......
[ERROR] xxx/Task.java:71:29: '+' is not preceded with whitespace. [WhitespaceAround]
checkStyle>> Javacode stylecheck finished. Thank you for your commit!
checkStyle>> Please return and fix stylecheck warnings before code commit！
```