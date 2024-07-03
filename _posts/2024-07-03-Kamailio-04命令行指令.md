---
layout:     post
title:      Kamailio-命令行指令kamctl与kamcmd

subtitle:   kamctl/kamcmd
date:       2024-07-02
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kamailio
    - kamctl
    - kamcmd
---

前文也有提到几种指令的用处，与web页面相比，它就是更原始、面向运维的，正常如果有管理页面也需要使用到：

- kamailio - SIP 服务器脚本
- kamdbctl - 创建和管理数据库的脚本，比如你使用MySQL作为其存储时就需要使用到这个
- kamctl - 管理和控制SIP服务器的脚本
- kamcmd - CLI 可以与SIP服务器交互的命令行接口

今天主要讨论kamctl、kamcmd 两个运行时访问工具。

- [kamctl](#kamctl)
  - [示例](#示例)
- [kamcmd](#kamcmd)

![img](../img/blog/Kamailio-cmd.png)

## kamctl

kamctl 是一个 shell 脚本，用于控制 Kamailio SIP 服务器，可用于管理用户、域、别名和其他服务器选项。


`以下是新版kamctl-5.9.0指令`

命令：

```bash
start #启动 Kamalio
restart #重启 Kamalio
stop #停止 Kamalio
trap #通过RPC捕获kamailio进程
pstrap #通过ps捕获Kamailio进程

# 权限管理的指令
acl show [user]
acl grant <user> <group>
acl revoke <user> <group>

# 管理最低成本路由
lcr xxxx

# 管理 carrier 路由
cr xxx

# 管理 全程组织ID - Remote-Party-ID(RPID)
rpid xxxx

# add/passwd/rm
add user password
show user
passwd user password
rm user
set user attr val 
setn user attr val

# 管理可信的
trusted show
trusted dump
trusted reload
trusted add
trusted rm

# 管理地址
address show
address dump
address reload
address add
address rm

# 管理调度器
dispatcher show 
dispatcher reload
dispatcher dump
dispatcher add 
dispatcher rm 
dispatcher rmip
dispatcher rmset

# 管理用户地址或别名
ul show
ul rm
ul add
ul dbcleaan

# 展示在线用户
online

# ping sip uri
ping uri

# 展示状态
monitor

# 管理本地domian
domain reload
domain show
domain showdb
domain add
domain rm

# 管理数据库别名
alias_db show
alias_db list
alias_db add 
alias_db rm

# 管理AVPs
avp list
avp add 
avp rm

# 数据库指令
db exec
db run
db show
db connect

# 管理账户记录
acc initdb
acc showdb

# manage mtree
mtree show
mtree dumo
mtree reload
mtree add
mtree rm

# 服务端管理指令
srv sockets
srv aliases
srv rpclist
srv modules
src version

# 管理会话记录
dialog show
dialog showddb

#
kamcmd
```

### 示例

- add

```bash
$: /usr/local/sbin/kamctl  add user@127.0.0.1 password
```

- rm

```bash
$: /usr/local/sbin/kamctl  rm user@127.0.0.1 
```

- ul show

```bash
$ : /usr/local/sbin/kamctl ul show
{
  "jsonrpc":  "2.0",
  "result": {
    "Domains":  [{
        "Domain": {
          "Domain": "location",
          "Size": 1024,
          "AoRs": [{
              "Info": {
                "AoR":  "1001",
                "HashID": 1790834316,
                "Contacts": [{
                    "Contact":  {
                      "Address":  "sip:172.17.0.1:49911;transport=udp",
                      "Expires":  400,
                      "Q":  -1,
                      "Call-ID":  "87d9f5d7-a8b6-4b0a-9534-ea53d2530390",
                      "CSeq": 578833,
                      "User-Agent": "SIPExer v1.1.0",
                      "Received": "[not set]",
                      "Path": "[not set]",
                      "State":  "CS_SYNC",
                      "Flags":  0,
                      "CFlags": 0,
                      "Socket": "udp:172.17.0.1:5060",
                      "Methods":  4294967295,
                      "Ruid": "uloc-6673c56d-484b9-1",
                      "Instance": "[not set]",
                      "Reg-Id": 0,
                      "Server-Id":  0,
                      "Tcpconn-Id": -1,
                      "Keepalive":  0,
                      "Last-Keepalive": 1718863898,
                      "KA-Roundtrip": 0,
                      "Last-Modified":  1718863898
                    }
                  }]
              }
            }
  ],
          "Stats":  {
            "Records":  1,
            "Max-Slots":  1
          }
        }
      }]
  },
  "id": 296483
}

```

- db show

```bash
$: /usr/local/sbin/kamctl db show subscriber
```

## kamcmd

kamcmd 是与 Kamailio SIP 服务器交互的命令行，可用于管理用户、域、别名和其他服务器选项。

```bash
version: kamcmd 1.5
Usage: kamcmd [options][-s address] [ cmd ]
Options:
    -s address  unix socket name or host name to send the commands on
    -R name     force reply socket name, for the unix datagram socket mode
    -D dir      create the reply socket in the directory <dir> if no reply
                socket is forced (-R) and a unix datagram socket is selected
                as the transport
    -f format   print the result using format. Format is a string containing
                %v at the places where values read from the reply should be
                substituted. To print '%v', escape it using '%': %%v.
    -v          Verbose
    -V          Version number
    -h          This help message
address:
    [proto:]name[:port]   where proto is one of tcp, udp, unixs or unixd
                          e.g.:  tcp:localhost:2049 , unixs:/tmp/kamailio_ctl
cmd:
    method  [arg1 [arg2...]]
arg:
     string or number; to force a number to be interpreted as string
     prefix it by "s:", e.g. s:1
Examples:
        kamcmd -s unixs:/tmp/kamcmd_ctl system.listMethods
        kamcmd -f "pid: %v  desc: %v\n" -s udp:localhost:2047 core.ps
        kamcmd ps  # uses default ctl socket
        kamcmd     # enters interactive mode on the default socket
        kamcmd -s tcp:localhost # interactive mode, default port
```
