---
layout:     post
title:      Kamailio-基于Homer与heplify的SIP信令监控-1

subtitle:   Kamailio Homer heplify
date:       2024-09-12
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kamailio
    - Homer
    - heplify
    - heplify-server
    - HEP
---

接[上篇](https://blog.csdn.net/c_zyer/article/details/142208675)，对Kamailio的一个基础监控有了一定的概念，但是光看数字如果发现问题，要如何回顾解决呢？生产环境不能随时随地抓包来确定链路的正常与否。

这个时候 Sipcapture 公司推出了`Homer`这个开源软件，目前`Github有1.6K的star`。

搭建之后体验了一把，先说说总体的印象：

**缺点：**

- 部署相对复杂，需要几个服务合作，包括heplify/heplify-server/homer-app，如果你只是小打小闹做点监控，可以先不要尝试。
- 开源版本功能没有想想中的强大，有”开源版本“该有的丐版模样。

**优点：**

- 相对整体的结构布局，可以从Kamalio获取数据，也可以从网口获取 HEP（分包协议）数据，通过heplify-server暂存后，发往homer。有后端服务、前端展示分离的服务，对KM监控无侵入性。
- Homer对会话数据进行异步存储后，很多自定义的功能可以顺利地在homer上进行二开，基础的查询、会话调用查看，对接有loki\prometheus\ldap\grafana 等新式监控组件与统一认证。
- 有专业团队维护，持续的更新，活跃的社区，都能保障服务的质量，未来可期。

因为这一整套有专业公司维护，因此这一套还被称为：Sipcapture HEP Stack。

下面将分三个章节来介绍整个Homer的部署安装配置、Kamailio的配置、Homer页面的查询与操作。

跟着我，你将学会：

- [下载并安装](#下载并安装)
  - [踩坑：按照官方步骤来，可是网络条件不允许](#踩坑按照官方步骤来可是网络条件不允许)
    - [获取YUM源](#获取yum源)
    - [下载RPM包](#下载rpm包)
    - [手动解压安装](#手动解压安装)
    - [避坑](#避坑)

[官方指导手册](https://github.com/sipcapture/homer)先放出，如后续此文章版本过期了，请移步官方文档流程。

以下**实验版本**：

- x86_64 CentOS Linux release 7.9.2009
- heplify 1.66.7
- heplify-server 1.59.7
- homer7
- PostgreSQL 11.4

为什么不装Homer10？因为生产还是虚机，没上Docker!!

安装的过程中遇到了好多坑的，下面一起来看看。

## 下载并安装

### 踩坑：按照官方步骤来，可是网络条件不允许

#### 获取YUM源

首先先要安装一些下载的源，CentOS 采用rpm进行安装。

第一步：

```bash
curl -s https://packagecloud.io/install/repositories/qxip/sipcapture/script.rpm.sh | sudo bash
```

看似一行，实则生产环境翻墙不通，直接就是失败了，那么怎么解决？

看看脚本里到底需要哪些源，需要哪些包，是不是可以手动下了传上去？

那首先就是wget把脚本拿下来。

```bash
#!/bin/bash

unknown_os ()
{
  echo "Unfortunately, your operating system distribution and version are not supported by this script."
  echo
  echo "You can override the OS detection by setting os= and dist= prior to running this script."
  echo "You can find a list of supported OSes and distributions on our website: https://packagecloud.io/docs#os_distro_version"
  echo
  echo "For example, to force CentOS 6: os=el dist=6 ./script.sh"
  echo
  echo "Please email support@packagecloud.io and let us know if you run into any issues."
  exit 1
}

curl_check ()
{
  echo "Checking for curl..."
  if command -v curl > /dev/null; then
    echo "Detected curl..."
  else
    echo "Installing curl..."
    yum install -d0 -e0 -y curl
  fi
}


detect_os ()
{
  if [[ ( -z "${os}" ) && ( -z "${dist}" ) ]]; then
    if [ -e /etc/os-release ]; then
      . /etc/os-release
      os=${ID}
      if [ "${os}" = "poky" ]; then
        dist=`echo ${VERSION_ID}`
      elif [ "${os}" = "sles" ]; then
        dist=`echo ${VERSION_ID}`
      elif [ "${os}" = "opensuse" ]; then
        dist=`echo ${VERSION_ID}`
      elif [ "${os}" = "opensuse-leap" ]; then
        os=opensuse
        dist=`echo ${VERSION_ID}`
      elif [ "${os}" = "amzn" ]; then
        dist=`echo ${VERSION_ID}`
      else
        dist=`echo ${VERSION_ID} | awk -F '.' '{ print $1 }'`
      fi

    elif [ `which lsb_release 2>/dev/null` ]; then
      # get major version (e.g. '5' or '6')
      dist=`lsb_release -r | cut -f2 | awk -F '.' '{ print $1 }'`

      # get os (e.g. 'centos', 'redhatenterpriseserver', etc)
      os=`lsb_release -i | cut -f2 | awk '{ print tolower($1) }'`

    elif [ -e /etc/oracle-release ]; then
      dist=`cut -f5 --delimiter=' ' /etc/oracle-release | awk -F '.' '{ print $1 }'`
      os='ol'

    elif [ -e /etc/fedora-release ]; then
      dist=`cut -f3 --delimiter=' ' /etc/fedora-release`
      os='fedora'

    elif [ -e /etc/redhat-release ]; then
      os_hint=`cat /etc/redhat-release  | awk '{ print tolower($1) }'`
      if [ "${os_hint}" = "centos" ]; then
        dist=`cat /etc/redhat-release | awk '{ print $3 }' | awk -F '.' '{ print $1 }'`
        os='centos'
      elif [ "${os_hint}" = "scientific" ]; then
        dist=`cat /etc/redhat-release | awk '{ print $4 }' | awk -F '.' '{ print $1 }'`
        os='scientific'
      else
        dist=`cat /etc/redhat-release  | awk '{ print tolower($7) }' | cut -f1 --delimiter='.'`
        os='redhatenterpriseserver'
      fi

    else
      aws=`grep -q Amazon /etc/issue`
      if [ "$?" = "0" ]; then
        dist='6'
        os='aws'
      else
        unknown_os
      fi
    fi
  fi

  if [[ ( -z "${os}" ) || ( -z "${dist}" ) ]]; then
    unknown_os
  fi

  # Enumerate all old versions that need pygpgme for each os/dist.
  # This is not hard because all old versions are known.
  # Even if we miss some versions, users will report them and eventually we'll have a complete list.
  # This ensures that if pygpgme is required, we will install it.
  # The harder part is detecting when pygpgme is NOT required since we don't know what
  # new version names/numbering are, however, it's alright to miss those because
  # as long as they are not the enumerated old versions, we can assume that they are new.
  # If we wrongly assume that they are new, then repo/package install will fail and users
  # will report back, and this goes away eventually once we have a complete list of old versions
  amzn_dist_requires_pygpgme_array=("1${IFS}2${IFS}2016${IFS}2017${IFS}2018")

  echo "Detected operating system as ${os}/${dist}."

  if [[ "$os" = "ol" || "$os" = "el" || "$os" = "rocky" || "$os" = "almalinux" || "$os" = "centos" || "$os" = "rhel" ]] && [ $(($dist)) \> 7 ]; then
    _skip_pygpgme=1
  elif [ "$os" = "fedora" ] && [ $(($dist)) \> 19 ]; then
    _skip_pygpgme=1
  elif [ "$os" = "amzn" ]; then
    amzn_dist="${dist%%[.-]*}"
    if  [[ ! " ${amzn_dist_requires_pygpgme_array[*]} " =~ [[:space:]]${amzn_dist}[[:space:]] ]]; then
      # whatever you want to do when array doesn't contain value
      _skip_pygpgme=1
    else
      _skip_pygpgme=0
    fi
  else
    _skip_pygpgme=0
  fi
}

finalize_yum_repo ()
{
  if [ "$_skip_pygpgme" = 0 ]; then
    echo "Attempting to install pygpgme for your os/dist: ${os}/${dist}. Only required on older OSes to verify GPG signatures."
    yum install -y pygpgme --disablerepo="${repo_config_name}" 2>/dev/null
    pypgpme_check=`rpm -qa | grep -qw pygpgme`
    if [ "$?" != "0" ]; then
      echo
      echo "Note: "
      echo "The pygpgme package may no longer be required for your OS (${os}/${dist}), and has not been installed."
      echo "If your repo and package installation does not work please reach out to support."
      echo
    fi
  fi

  echo "Installing yum-utils..."
  yum install -y yum-utils --disablerepo="${repo_config_name}"
  yum_utils_check=`rpm -qa | grep -qw yum-utils`
  if [ "$?" != "0" ]; then
    echo
    echo "WARNING: "
    echo "The yum-utils package could not be installed. This means you may not be able to install source RPMs or use other yum features."
    echo
  fi

  echo "Generating yum cache for ${repo_config_name}..."
  yum -q makecache -y --disablerepo='*' --enablerepo="${repo_config_name}"

  echo "Generating yum cache for ${repo_config_name}-source..."
  yum -q makecache -y --disablerepo='*' --enablerepo="${repo_config_name}-source"
}

finalize_zypper_repo ()
{
  zypper --gpg-auto-import-keys refresh $repo_config_name
  zypper --gpg-auto-import-keys refresh $repo_config_name-source
}


main ()
{
  repo_config_name=qxip_sipcapture
  detect_os
  curl_check


  yum_repo_config_url="https://packagecloud.io/install/repositories/qxip/sipcapture/config_file.repo?os=${os}&dist=${dist}&source=script"

  if [ "${os}" = "sles" ] || [ "${os}" = "opensuse" ]; then
    yum_repo_path=/etc/zypp/repos.d/$repo_config_name.repo
  else
    yum_repo_path=/etc/yum.repos.d/$repo_config_name.repo
  fi

  echo "Downloading repository file: ${yum_repo_config_url}"

  curl -sSf "${yum_repo_config_url}" > $yum_repo_path
  curl_exit_code=$?

  if [ "$curl_exit_code" = "22" ]; then
    echo
    echo
    echo -n "Unable to download repo config from: "
    echo "${yum_repo_config_url}"
    echo
    echo "This usually happens if your operating system is not supported by "
    echo "packagecloud.io, or this script's OS detection failed."
    echo
    echo "You can override the OS detection by setting os= and dist= prior to running this script."
    echo "You can find a list of supported OSes and distributions on our website: https://packagecloud.io/docs#os_distro_version"
    echo
    echo "For example, to force CentOS 6: os=el dist=6 ./script.sh"
    echo
    echo "If you are running a supported OS, please email support@packagecloud.io and report this."
    [ -e $yum_repo_path ] && rm $yum_repo_path
    exit 1
  elif [ "$curl_exit_code" = "35" -o "$curl_exit_code" = "60" ]; then
    echo
    echo "curl is unable to connect to packagecloud.io over TLS when running: "
    echo "    curl ${yum_repo_config_url}"
    echo
    echo "This is usually due to one of two things:"
    echo
    echo " 1.) Missing CA root certificates (make sure the ca-certificates package is installed)"
    echo " 2.) An old version of libssl. Try upgrading libssl on your system to a more recent version"
    echo
    echo "Contact support@packagecloud.io with information about your system for help."
    [ -e $yum_repo_path ] && rm $yum_repo_path
    exit 1
  elif [ "$curl_exit_code" -gt "0" ]; then
    echo
    echo "Unable to run: "
    echo "    curl ${yum_repo_config_url}"
    echo
    echo "Double check your curl installation and try again."
    [ -e $yum_repo_path ] && rm $yum_repo_path
    exit 1
  else
    echo "done."
  fi


  if [ "${os}" = "sles" ] || [ "${os}" = "opensuse" ]; then
    finalize_zypper_repo
  else
    finalize_yum_repo
  fi

  echo
  echo "The repository is setup! You can now install packages."
}

main

```

简单一看脚本里做的事情，就是判断我的操作系统，然后生产一个yum仓库config的文件地址，然后配置为资源地址。

这里只是生产一个yum源地址是没什么问题的，那关键的是这个地址是外网，国内不用手段就访问不了。

所以你梳理执行完源配置后，下一步拉取，包括其他所有yum安装的步骤就都开始报错，说这个仓库地址连接不上。

所以遗憾的，需要手动来做后续的步骤，手动下载配置，看后续的包到底是去哪里下载的，手动下载后，放到服务器上。

我根据上面的脚本拼接出我的yum源配置的内容：
`https://packagecloud.io/install/repositories/qxip/sipcapture/config_file.repo?os=centos&dist=7&source=script`

放到浏览器，获取到仓库信息：

```yml
[qxip_sipcapture]
name=qxip_sipcapture
baseurl=https://packagecloud.io/qxip/sipcapture/el/7/$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/qxip/sipcapture/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300

[qxip_sipcapture-source]
name=qxip_sipcapture-source
baseurl=https://packagecloud.io/qxip/sipcapture/el/7/SRPMS
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/qxip/sipcapture/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
```

看到了我想要的真实下载安装包的网址 `https://packagecloud.io/qxip/sipcapture/`，通过合法手段登录平台后你将看到。

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-Homer-downloadpage.png)

能看到我们非常熟悉的RPM包，这个就好办了。

#### 下载RPM包

手动下载，传送到服务器，然后采用rpm手动安装，这一条流程就比较顺利了。

确认准备好的材料。

```bash
$ ls -l
total 48296
-rw-r--r-- 1 root root  5343776 Aug 20 16:22 heplify-1.66.6-1.x86_64.rpm
-rw-r--r-- 1 root root  7629945 Aug 20 16:31 heplify-server-1.59.7-1.x86_64.rpm
-rw-r--r-- 1 root root 22776724 Aug 20 16:31 homer-app-1.5.3-1.x86_64.rpm
```

#### 手动解压安装

手动魔鬼安装，顺序还是遵循官方的指导。

我这边heplify-server是借用现有的一个PG数据库，那么如果你没有的话，这个步骤请自行百度安装好，客户端也安装上(yum install postgresql -y)。

中间环节会提示要装相关的依赖包，比如：luajit-2.0.5、luajit-devel、postgresql-libs、postgresql-9.2.24。

rpm安装的过程中，可能需要一些依赖的版本是没有，它会提示让你更新仓库中的列表，你按照指示去更新即可。

报错示例：

```bash
error: Failed dependencies:
        luajit is needed by heplify-server-0:1.59.7-1.x86_64
# 表示缺 luajit, 那你就可以通过 yum install luajit-devel 之类的方式去安装      
```

环境一切顺利的话，rpm安装会非常快速

```bash
$ rpm -ivh heplify-1.66.6-1.x86_64.rpm
.....
$ rpm -ivh heplify-server-1.59.7-1.x86_64.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:heplify-server-0:1.59.7-1        ################################# [100%]
$ rpm -ivh homer-app-1.5.3-1.x86_64.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:homer-app-0:1.5.3-1              ################################# [100%]   
....

```

#### 避坑

那你如果提前就知道你的网络环境会有很多下包的限制，或者是纯内网的情况的话，可以提前将Rpm包素材离线准备完毕，直接按照上面的步骤依次执行安装即可，也不需要强行去走他的源去自动化安装了。少走很多坑吧。。。。。

通过以上艰难险阻，服务算是都安装上了，确认一下：

```bash
$ systemctl status heplify
● heplify.service - Captures packets from wire and sends them to Homer
   Loaded: loaded (/etc/systemd/system/heplify.service; enabled; vendor preset: disabled)
   Active: inactive (dead) since 三 2024-08-21 13:11:49 CST; 3 weeks 1 days ago
  Process: 22817 ExecStop=/bin/kill ${MAINPID} (code=exited, status=0/SUCCESS)
  Process: 29535 ExecStart=/opt/heplify/heplify -i lo -hs 192.168.5.167:9060 -m SIP -dim REGISTER -pr 5060-5090 (code=exited, status=0/SUCCESS)
 Main PID: 29535 (code=exited, status=0/SUCCESS)

Warning: Journal has been rotated since unit was started. Log output is incomplete or unavailable.


$ systemctl status heplify-server
● heplify-server.service - HEP Server & Switch in Go
   Loaded: loaded (/usr/lib/systemd/system/heplify-server.service; enabled; vendor preset: disabled)
   Active: inactive (dead) since 四 2024-08-29 11:59:57 CST; 2 weeks 0 days ago
  Process: 901 ExecStart=/usr/local/bin/heplify-server $HEPLIFY_CONFIG (code=exited, status=0/SUCCESS)
 Main PID: 901 (code=exited, status=0/SUCCESS)

Warning: Journal has been rotated since unit was started. Log output is incomplete or unavailable.


$ systemctl status homer-app
● homer-app.service - Homer API Server and UI Webapplication
   Loaded: loaded (/usr/lib/systemd/system/homer-app.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
[root@192 ~]# systemctl status homer-web
```

下面我们就对其开始配置、数据初始化，将其启动起来。
