---
layout:     post
title:      Kamailio-基于Zabbix+Kamcli的SIP指标监控

subtitle:   Kamailio Zabbix Kamcli
date:       2024-09-12
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kamailio
    - Zabbix
    - Kamcli
---

## 什么是Kamailio?

Kamailio 是一个开源的 Session Initiation Protocol (SIP) 服务器，它主要用于建立和管理实时通信会话，如语音和视频通话，与opensips这个产品是同根同源的存在。它们相似，没有更好，是有更合适。

此篇文章适用于对Kamailio以及VOIP相关有一定基础知识的人，如需恶补请[参考专栏](https://blog.csdn.net/c_zyer/category_12717332.html)， zero to hero。

## 为什么要监控？

当我们开始使用Kamailio之后，就会产生一些担忧：

- 脚本适配性如何
- 是否有一些意料不到的问题无法知晓
- UDP信令是否有丢失
- 服务的资源使用情况如何？

因此监控对于服务来说，是必须的。

对于市面上很多监控、展示的监控软件来看，对SIP类型的监控的支持并不多，今天就介绍一个 `51star` 的小众github开源项目-Kamcli。

它是一个5人开发的，基于原生Kamctl命令，使用Python实现指令封装工具，主要定期调用原生指令，与zabbix-agent配合，最终将业务监控指标数据展示在Zabbix中。

先讲讲我的评价：

- 相对短小精悍
- 用于基础指标监控
- 是一个可与zabbix配合SIP监控小工具

它的实现逻辑也不算复杂，可以[自行观看](https://github.com/kamailio/kamcli)。

## Kamcli

### 安装

前置条件：

- python3 (python version 3.x, recommended at least Python 3.7)
- python3-pip
- python3-setuptools
- python3-dev (optional - needed to install mysqlclient via pip)
- python3-venv (optional - needed to install virtual environment)

关于服务器环境没有太多限制：可以虚拟机、Debian、Ubuntu、Mint、CentOS。区别可能就在于python基础环境安装的步骤有所差异。

安装依赖环境：

```bash
$ pip3 install -r requirements/requirements.txt
```

如果使用MySQL作为后台数据库的话，还需要安装 mysqlclient，这个很好理解。

```bash
$ pip3 install mysqlclient
```

此外后端存储还可以使用 PostgreSQL\SQLite。

拉取代码进行编译安装，我们这边就采用现有的一个MySQL数据库。

```bash
$ git clone https://github.com/kamailio/kamcli.git
$ cd kamcli
$ pip3 install -r requirements/requirements.txt
$ pip3 install mysqlclient
$ pip3 install --editable .
```

完成以上步骤后，环境版本没有冲突报错的话，就可以顺利安装完成：

### 使用 

```bash
$ /usr/local/bin/kamcli --help
Usage: kamcli [OPTIONS] COMMAND [ARGS]...

  Kamailio command line interface control tool.

  Help per command: kamcli <command> --help

  Default configuration files:
      - /etc/kamcli/kamcli.ini
      - ./kamcli.ini
      - ./kamcli/kamcli.ini
      - ~/.kamcli/kamctli.ini
  Configs loading order: default configs, then --config option

  License: GPLv2
  Copyright: asipto.com

Options:
  -d, --debug                     Enable debug mode.
  -c, --config TEXT               Configuration file.
  -w, --wdir DIRECTORY            Working directory.
  -n, --no-default-configs        Skip loading default configuration files.
  -F, --output-format [raw|json|table|dict|yaml]
                                  Format the output (overwriting db/jsonrpc
                                  outformat config attributes)

  --version                       Show the version and exit.
  -h, --help                      Show this message and exit.

Commands:
  acc         Accounting management
  address     Manage permissions address records
  aliasdb     Manage database user aliases
  apiban      Manage APIBan records
  avp         Manage AVP user preferences
  config      Manage the config file
  db          Raw database operations
  dialog      Manage dialog module
  dialplan    Manage dialplan module
  dispatcher  Manage dispatcher module
  dlgs        Manage dlgs module
  domain      Manage domain module
  group       Manage group module
  htable      Management of htable module
  jsonrpc     Execute JSONRPC commands
  moni        Monitor relevant statistics
  mtree       Manage mtree module
  pike        Manage pike module
  ping        Send an OPTIONS ping request
  pipelimit   Manage pipelimit module
  pkg         Private memory (pkg) management
  ps          Print the list of kamailio processes
  pstrap      Get runtime details and gdb backtrace with ps
  rpcmethods  Print the list of available raw RPC methods
  rtpengine   Manage rtpengine module
  shell       Run in interactive shell mode
  shm         Shared memory (shm) management
  shv         Manage $shv(name) variables
  sipreq      Send a SIP request via RPC command
  speeddial   Manage speed dial records
  srv         Common server interaction commands
  stats       Print internal statistics
  subscriber  Manage the subscribers
  tcp         Manage TCP options and connections
  tls         Manage tls module
  trap        Get runtime details and gdb full backtrace
  uacreg      Manage uac registrations
  ul          Manage user location records
  uptime      Print the uptime for kamailio
```

样例：

```bash
kamcli -d --help
kamcli -d --config=kamcli/kamcli.ini --help

kamcli subscriber show
kamcli subscriber add test test00
kamcli subscriber show test
kamcli subscriber show --help
kamcli -d subscriber passwd test01 test10
kamcli -d subscriber add -t no test02 test02
kamcli -d subscriber setattrs test01 rpid +123
kamcli -d subscriber setattrnull test01 rpid

kamcli -d jsonrpc --help
kamcli -d jsonrpc core.psx
kamcli -d jsonrpc system.listMethods
kamcli -d jsonrpc stats.get_statistics
kamcli -d jsonrpc stats.get_statistics all
kamcli -d jsonrpc stats.get_statistics shmem:
kamcli -d jsonrpc --dry-run system.listMethods

kamcli -d config raw
kamcli -d config show main db
kamcli -d -no-default-configs config show main db

kamcli -d db connect
kamcli -d db show -F table version
kamcli -d db show -F json subscriber
kamcli -d db showcreate version
kamcli -d db showcreate -F table version
kamcli -d db showcreate -F table -S html version
kamcli -d db clirun "describe version"
kamcli -d db clishow version
kamcli -d db clishowg subscriber


kamcli -d ul showdb
kamcli -d ul show
kamcli -d ul rm test
kamcli -d ul add test sip:test@127.0.0.1

kamcli -d stats
kamcli -d stats usrloc
kamcli -d stats -s registered_users
kamcli -d stats usrloc:registered_users
```

看到封装后的指令应该是非常熟悉，都是来自于kamctl命令行，就像它的slogan。

`kamcli is aiming at being a modern and extensible alternative to the shell script kamctl.`

对于监控统计而言，比较关注的就是 `kamcli stats`。

```bash
$ kamcli stats

id: 6694
jsonrpc: '2.0'
result:
  core.bad_URIs_rcvd: '0'
  core.bad_msg_hdr: '0'
  core.drop_replies: '0'
  core.drop_requests: '0'
  core.err_replies: '0'
  core.err_requests: '0'
  core.fwd_replies: '0'
  core.fwd_requests: '4'
  core.rcv_replies: '286'
  core.rcv_replies_18x: '3'
  core.rcv_replies_1xx: '141'
  core.rcv_replies_1xx_bye: '0'
  core.rcv_replies_1xx_cancel: '0'
  core.rcv_replies_1xx_invite: '141'
  core.rcv_replies_1xx_message: '0'
  core.rcv_replies_1xx_prack: '0'
  core.rcv_replies_1xx_refer: '0'
  core.rcv_replies_1xx_reg: '0'
  core.rcv_replies_1xx_update: '0'
  core.rcv_replies_2xx: '5'
  core.rcv_replies_2xx_bye: '1'
  core.rcv_replies_2xx_cancel: '0'
  core.rcv_replies_2xx_invite: '4'
  core.rcv_replies_2xx_message: '0'
  core.rcv_replies_2xx_prack: '0'
  core.rcv_replies_2xx_refer: '0'
  core.rcv_replies_2xx_reg: '0'
  core.rcv_replies_2xx_update: '0'
  core.rcv_replies_3xx: '0'
  core.rcv_replies_3xx_bye: '0'
  core.rcv_replies_3xx_cancel: '0'
  core.rcv_replies_3xx_invite: '0'
  core.rcv_replies_3xx_message: '0'
  core.rcv_replies_3xx_prack: '0'
  core.rcv_replies_3xx_refer: '0'
  core.rcv_replies_3xx_reg: '0'
  core.rcv_replies_3xx_update: '0'
  core.rcv_replies_401: '0'
  core.rcv_replies_404: '0'
  core.rcv_replies_407: '0'
  core.rcv_replies_480: '0'
  core.rcv_replies_486: '0'
  core.rcv_replies_4xx: '140'
  core.rcv_replies_4xx_bye: '0'
  core.rcv_replies_4xx_cancel: '0'
  core.rcv_replies_4xx_invite: '140'
  core.rcv_replies_4xx_message: '0'
  core.rcv_replies_4xx_prack: '0'
  core.rcv_replies_4xx_refer: '0'
  core.rcv_replies_4xx_reg: '0'
  core.rcv_replies_4xx_update: '0'
  core.rcv_replies_5xx: '0'
  core.rcv_replies_5xx_bye: '0'
  core.rcv_replies_5xx_cancel: '0'
  core.rcv_replies_5xx_invite: '0'
  core.rcv_replies_5xx_message: '0'
  core.rcv_replies_5xx_prack: '0'
  core.rcv_replies_5xx_refer: '0'
  core.rcv_replies_5xx_reg: '0'
  core.rcv_replies_5xx_update: '0'
  core.rcv_replies_6xx: '0'
  core.rcv_replies_6xx_bye: '0'
  core.rcv_replies_6xx_cancel: '0'
  core.rcv_replies_6xx_invite: '0'
  core.rcv_replies_6xx_message: '0'
  core.rcv_replies_6xx_prack: '0'
  core.rcv_replies_6xx_refer: '0'
  core.rcv_replies_6xx_reg: '0'
  core.rcv_replies_6xx_update: '0'
  core.rcv_requests: '293'
  core.rcv_requests_ack: '144'
  core.rcv_requests_bye: '3'
  core.rcv_requests_cancel: '0'
  core.rcv_requests_info: '0'
  core.rcv_requests_invite: '146'
  core.rcv_requests_message: '0'
  core.rcv_requests_notify: '0'
  core.rcv_requests_options: '0'
  core.rcv_requests_prack: '0'
  core.rcv_requests_publish: '0'
  core.rcv_requests_refer: '0'
  core.rcv_requests_register: '0'
  core.rcv_requests_subscribe: '0'
  core.rcv_requests_update: '0'
  core.unsupported_methods: '0'
  dns.failed_dns_request: '0'
  dns.slow_dns_request: '0'
  mysql.driver_errors: '0'
  registrar.accepted_regs: '0'
  registrar.default_expire: '3600'
  registrar.default_expires_range: '0'
  registrar.expires_range: '0'
  registrar.max_contacts: '0'
  registrar.max_expires: '3600'
  registrar.rejected_regs: '0'
  shmem.fragments: '6'
  shmem.free_size: '131134656'
  shmem.max_used_size: '5958976'
  shmem.real_used_size: '3083072'
  shmem.total_size: '134217728'
  shmem.used_size: '2801584'
  sl.1xx_replies: '0'
  sl.200_replies: '0'
  sl.202_replies: '0'
  sl.2xx_replies: '0'
  sl.300_replies: '0'
  sl.301_replies: '0'
  sl.302_replies: '0'
  sl.3xx_replies: '0'
  sl.400_replies: '0'
  sl.401_replies: '0'
  sl.403_replies: '0'
  sl.404_replies: '2'
  sl.407_replies: '0'
  sl.408_replies: '0'
  sl.483_replies: '2'
  sl.4xx_replies: '0'
  sl.500_replies: '0'
  sl.5xx_replies: '0'
  sl.6xx_replies: '0'
  sl.failures: '2'
  sl.received_ACKs: '0'
  sl.sent_err_replies: '0'
  sl.sent_replies: '4'
  sl.xxx_replies: '0'
  tcp.con_reset: '0'
  tcp.con_timeout: '0'
  tcp.connect_failed: '0'
  tcp.connect_success: '0'
  tcp.current_opened_connections: '0'
  tcp.current_write_queue_size: '0'
  tcp.established: '0'
  tcp.local_reject: '0'
  tcp.passive_open: '0'
  tcp.send_timeout: '0'
  tcp.sendq_full: '0'
  tmx.2xx_transactions: '5'
  tmx.3xx_transactions: '0'
  tmx.4xx_transactions: '146'
  tmx.5xx_transactions: '0'
  tmx.6xx_transactions: '0'
  tmx.UAC_transactions: '6'
  tmx.UAS_transactions: '151'
  tmx.active_transactions: '0'
  tmx.inuse_transactions: '0'
  tmx.rpl_absorbed: '138'
  tmx.rpl_generated: '150'
  tmx.rpl_received: '286'
  tmx.rpl_relayed: '148'
  tmx.rpl_sent: '298'
  usrloc.location_contacts: '0'
  usrloc.location_expires: '0'
  usrloc.location_users: '0'
  usrloc.registered_users: '0'
```

对比一下原生的 kamctl:

```bash
$ kamctl stats 
{
  "jsonrpc":  "2.0",
  "result": [
    "core:bad_URIs_rcvd = 0",
    "core:bad_msg_hdr = 0",
    "core:drop_replies = 0",
    "core:drop_requests = 0",
    "core:err_replies = 0",
    "core:err_requests = 0",
    "core:fwd_replies = 0",
    "core:fwd_requests = 4",
    "core:rcv_replies = 286",
    "core:rcv_replies_18x = 3",
    "core:rcv_replies_1xx = 141",
    "core:rcv_replies_1xx_bye = 0",
    "core:rcv_replies_1xx_cancel = 0",
    "core:rcv_replies_1xx_invite = 141",
    "core:rcv_replies_1xx_message = 0",
    "core:rcv_replies_1xx_prack = 0",
    "core:rcv_replies_1xx_refer = 0",
    "core:rcv_replies_1xx_reg = 0",
    "core:rcv_replies_1xx_update = 0",
    "core:rcv_replies_2xx = 5",
    "core:rcv_replies_2xx_bye = 1",
    "core:rcv_replies_2xx_cancel = 0",
    "core:rcv_replies_2xx_invite = 4",
    "core:rcv_replies_2xx_message = 0",
    "core:rcv_replies_2xx_prack = 0",
    "core:rcv_replies_2xx_refer = 0",
    "core:rcv_replies_2xx_reg = 0",
    "core:rcv_replies_2xx_update = 0",
    "core:rcv_replies_3xx = 0",
    "core:rcv_replies_3xx_bye = 0",
    "core:rcv_replies_3xx_cancel = 0",
    "core:rcv_replies_3xx_invite = 0",
    "core:rcv_replies_3xx_message = 0",
    "core:rcv_replies_3xx_prack = 0",
    "core:rcv_replies_3xx_refer = 0",
    "core:rcv_replies_3xx_reg = 0",
    "core:rcv_replies_3xx_update = 0",
    "core:rcv_replies_401 = 0",
    "core:rcv_replies_404 = 0",
    "core:rcv_replies_407 = 0",
    "core:rcv_replies_480 = 0",
    "core:rcv_replies_486 = 0",
    "core:rcv_replies_4xx = 140",
    "core:rcv_replies_4xx_bye = 0",
    "core:rcv_replies_4xx_cancel = 0",
    "core:rcv_replies_4xx_invite = 140",
    "core:rcv_replies_4xx_message = 0",
    "core:rcv_replies_4xx_prack = 0",
    "core:rcv_replies_4xx_refer = 0",
    "core:rcv_replies_4xx_reg = 0",
    "core:rcv_replies_4xx_update = 0",
    "core:rcv_replies_5xx = 0",
    "core:rcv_replies_5xx_bye = 0",
    "core:rcv_replies_5xx_cancel = 0",
    "core:rcv_replies_5xx_invite = 0",
    "core:rcv_replies_5xx_message = 0",
    "core:rcv_replies_5xx_prack = 0",
    "core:rcv_replies_5xx_refer = 0",
    "core:rcv_replies_5xx_reg = 0",
    "core:rcv_replies_5xx_update = 0",
    "core:rcv_replies_6xx = 0",
    "core:rcv_replies_6xx_bye = 0",
    "core:rcv_replies_6xx_cancel = 0",
    "core:rcv_replies_6xx_invite = 0",
    "core:rcv_replies_6xx_message = 0",
    "core:rcv_replies_6xx_prack = 0",
    "core:rcv_replies_6xx_refer = 0",
    "core:rcv_replies_6xx_reg = 0",
    "core:rcv_replies_6xx_update = 0",
    "core:rcv_requests = 293",
    "core:rcv_requests_ack = 144",
    "core:rcv_requests_bye = 3",
    "core:rcv_requests_cancel = 0",
    "core:rcv_requests_info = 0",
    "core:rcv_requests_invite = 146",
    "core:rcv_requests_message = 0",
    "core:rcv_requests_notify = 0",
    "core:rcv_requests_options = 0",
    "core:rcv_requests_prack = 0",
    "core:rcv_requests_publish = 0",
    "core:rcv_requests_refer = 0",
    "core:rcv_requests_register = 0",
    "core:rcv_requests_subscribe = 0",
    "core:rcv_requests_update = 0",
    "core:unsupported_methods = 0",
    "dns:failed_dns_request = 0",
    "dns:slow_dns_request = 0",
    "mysql:driver_errors = 0",
    "registrar:accepted_regs = 0",
    "registrar:default_expire = 3600",
    "registrar:default_expires_range = 0",
    "registrar:expires_range = 0",
    "registrar:max_contacts = 0",
    "registrar:max_expires = 3600",
    "registrar:rejected_regs = 0",
    "shmem:fragments = 6",
    "shmem:free_size = 131134656",
    "shmem:max_used_size = 5958976",
    "shmem:real_used_size = 3083072",
    "shmem:total_size = 134217728",
    "shmem:used_size = 2801584",
    "sl:1xx_replies = 0",
    "sl:200_replies = 0",
    "sl:202_replies = 0",
    "sl:2xx_replies = 0",
    "sl:300_replies = 0",
    "sl:301_replies = 0",
    "sl:302_replies = 0",
    "sl:3xx_replies = 0",
    "sl:400_replies = 0",
    "sl:401_replies = 0",
    "sl:403_replies = 0",
    "sl:404_replies = 2",
    "sl:407_replies = 0",
    "sl:408_replies = 0",
    "sl:483_replies = 2",
    "sl:4xx_replies = 0",
    "sl:500_replies = 0",
    "sl:5xx_replies = 0",
    "sl:6xx_replies = 0",
    "sl:failures = 2",
    "sl:received_ACKs = 0",
    "sl:sent_err_replies = 0",
    "sl:sent_replies = 4",
    "sl:xxx_replies = 0",
    "tcp:con_reset = 0",
    "tcp:con_timeout = 0",
    "tcp:connect_failed = 0",
    "tcp:connect_success = 0",
    "tcp:current_opened_connections = 0",
    "tcp:current_write_queue_size = 0",
    "tcp:established = 0",
    "tcp:local_reject = 0",
    "tcp:passive_open = 0",
    "tcp:send_timeout = 0",
    "tcp:sendq_full = 0",
    "tmx:2xx_transactions = 5",
    "tmx:3xx_transactions = 0",
    "tmx:4xx_transactions = 146",
    "tmx:5xx_transactions = 0",
    "tmx:6xx_transactions = 0",
    "tmx:UAC_transactions = 6",
    "tmx:UAS_transactions = 151",
    "tmx:active_transactions = 0",
    "tmx:inuse_transactions = 0",
    "tmx:rpl_absorbed = 138",
    "tmx:rpl_generated = 150",
    "tmx:rpl_received = 286",
    "tmx:rpl_relayed = 148",
    "tmx:rpl_sent = 298",
    "usrloc:location_contacts = 0",
    "usrloc:location_expires = 0",
    "usrloc:location_users = 0",
    "usrloc:registered_users = 0"
  ],
  "id": 22824
}
```

### 关于配置文件

配置文件可放置在这些目录下：

- ./kamcli/kamcli.ini
- ./kamcli.ini
- /etc/kamcli/kamcli.ini
- ~/.kamcli/kamcli.ini

看看配置文件的内容：

```yml
### main options
[main]
; SIP domain to be used when an AoR has no domain
domain=kamailio.org


### subcommand aliases
[cmdaliases]
# alias = subcommand
# - 'kamcli alias ...' becomes equivalent of 'kamcli subcommand ...'
mt = mtree
pl = pipelimit
sd = speeddial

### database connectivity - URLs are used for SQL Alchemy
[db]
; type of database
; - for MySQL: mysql,
; - for PostgreSQL: postgresql
; - for SQLite: sqlite
type=mysql
; driver to be used fro connecting
; - for MySQL: mysqldb
; - for PostgreSQL: psycopg2
; - for SQLite: pysqlite
driver=mysqldb
; host of database server
host=localhost
; port of database server
; - not enforced - see rwurl, rourl, adminurl
; - for MySQL: 3306
; - for PostgreSQL: 5432
dbport=3306
; kamailio database name for SQL server backends
dbname=kamailio
; kamailio database path for SQL file backends (e.g., sqlite)
dbpath=/etc/kamailio/kamailio.db
; read/write user
rwuser=kamailio
; password for read/write user
rwpassword=kamailiorw
; read only user
rouser=kamailioro
; password for read only user
ropassword=kamailioro
; admin user
adminuser=root
; password for admin user
adminpassword=
; database URLs
; - built using above attributes, don't change unless you know what you do
; - full format for SQL server backends (mysql, postgres, ...):
;     rwurl=%(type)s+%(driver)s://%(rwuser)s:%(rwpassword)s@%(host)s:%(dbport)s/%(dbname)s
;     rourl=%(type)s+%(driver)s://%(rouser)s:%(ropassword)s@%(host)s:%(dbport)s/%(dbname)s
;     adminurl=%(type)s+%(driver)s://%(adminuser)s:%(adminpassword)s@%(host)s:%(dbport)s
; - full format for SQL file backends (sqlite, ...):
;     rwurl=%(type)s+%(driver)s:///%(dbpath)s
;     rourl=%(type)s+%(driver)s:///%(dbpath)s
;     adminurl=%(type)s+%(driver)s:///%(dbpath)s
rwurl=%(type)s+%(driver)s://%(rwuser)s:%(rwpassword)s@%(host)s/%(dbname)s
rourl=%(type)s+%(driver)s://%(rouser)s:%(ropassword)s@%(host)s/%(dbname)s
adminurl=%(type)s+%(driver)s://%(adminuser)s:%(adminpassword)s@%(host)s

; host from where kamcli is used
accesshost=

; path to the folder with SQL scripts for creating database tables
; - used by `db create` subcommand if not provided via `-s` cli argument
; - example value for mysql: /usr/local/share/kamailio/mysql
; - example value for postgresql: /usr/local/share/kamailio/postgres
; - example value for sqlite: /usr/local/share/kamailio/db_sqlite
scriptsdirectory=/usr/local/share/kamailio/mysql

; outformat - the format to print database result
; - can be: table, json, yaml, dict or raw
outformat=table

; outstyle - the style to print database result with tabulate package
; - default: grid
# outstyle=grid


### control tool settings
[ctl]
; type - can be: jsonrpc
type=jsonrpc
; kamgroup - group of the running kamailio server process
kamgroup=kamailio


### jsonrpc settings
[jsonrpc]
; transport - can be: fifo, socket
transport=socket

; path - where kamailio is listening for JSONRPC FIFO commands
path=/var/run/kamailio/kamailio_rpc.fifo
rplnamebase=kamailio_rpc_reply.fifo
rpldir=/tmp

; srvaddr - where kamailio is listening for JSONRPC socket commands
;   - it has to be a path to unix socket file, udp:ipaddr:port
;     or tcp:ipaddr:port
srvaddr=/var/run/kamailio/kamailio_rpc.sock
; srvaddr=udp:127.0.0.1:9062
; srvaddr=tcp:127.0.0.1:9062

; rcvaddr - where kamcli is listening for the JSONRPC responses
;   - it has to be a path to unix socket file or udp:ipaddr:port
;   - pid of kamcli is added at the end to allow multiple use at same time
rcvaddr=/var/run/kamailio/kamailio_rpc_reply.sock
; rcvaddr=udp:127.0.0.1:9064

; outformat - the format to print RPC result
; - can be: json, yaml or raw
; - yaml is more compact output
outformat=yaml


### internal cmd shell settings
[shell]
; do not connect to Kamailio on start up (yes|no)
# noconnect=yes

; do not fetch RPC commands on start up for auto-complete (yes|no)
; - done only if 'noconnect=no'
# norpcautocomplete=yes

; do not track history of commands (yes|no)
# nohistory=yes

; do not enable syntax higlighting for shell command line (yes|no)
# nosyntax=yes

### command re-mapping for cmd shell
# - short name for full command with parameters
[shell.cmdremap]
dv=db show "version"
u=uptime


### apiban settings
[apiban]
; key - the APIBan key
# key=abcde...

; htname - htable name (if not set, defaults to 'ipban')
# htname=ipban
```

可以采用上面的内容自行新建，也可以通过指令生产默认的文件：

```bash
$ kamcli config install
# 文件将位于 /etc/kamcli/kamcli.ini

$ kamcli config install -u
# 文件将位于用户目录下 ~/.kamcli/kamcli.ini
```

### 结果展示

来看看 Zabbix采集后是什么样？

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-Zabbix-kamcli.png)

数据其实就是命令行可查的那些，通过Zabbix这个平台存储后，能够查看一些关键指标的时间变化趋势，对了解业务情况，发现业务瓶颈有很大帮助.
