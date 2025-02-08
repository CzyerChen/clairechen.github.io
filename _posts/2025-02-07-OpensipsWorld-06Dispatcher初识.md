---
layout:     post
title:      OpenSIPS-Dispatcher模块详解：优化SIP流量分发的利器

subtitle:   OpenSIPS 
date:       2025-02-08
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - OpenSIPS
    - sip
---

在 OpenSIPS 中，dispatcher 模块用于实现负载均衡和故障转移。通过 dispatcher 模块，你可以将 SIP 请求分发到一组后端服务器（如媒体服务器、代理服务器等），并根据配置的算法和策略动态调整分发逻辑。

- [模块功能](#模块功能)
- [使用样例](#使用样例)
	- [1. 加载模块](#1-加载模块)
	- [2. 配置模块参数](#2-配置模块参数)
	- [3. 调用方法](#3-调用方法)
	- [4. 样例脚本](#4-样例脚本)
	- [5. 测试和验证](#5-测试和验证)

## 模块功能

主要功能包括：

- 负载均衡：将 SIP 请求分发到多个后端服务器，分担负载。
- 故障转移：自动检测后端服务器的可用性，并将请求路由到健康的服务器。
- 动态更新：支持通过数据库或 MI（Management Interface）动态更新后端服务器列表。
- 多种算法：支持多种负载均衡算法，如轮询、随机、权重等。

## 使用样例

对于opensips，使用起来非常简单，引入模块，调用方法，无需额外处理。

### 1. 加载模块

不需要额外编译，就可以引入

```bash
loadmodule "dispatcher.so"
```

### 2. 配置模块参数

```bash
# 数据库连接 URL
modparam("dispatcher", "db_url", "mysql://opensips:opensips@localhost/opensips")

# 健康检查间隔（单位：秒），默认0，关闭检查
modparam("dispatcher", "ds_ping_interval", 10)

# 健康检查方法（如 OPTIONS 是默认方法，你也可以选择INFO）
modparam("dispatcher", "ds_ping_method", "OPTIONS")

# 健康检查失败阈值，默认3
modparam("dispatcher", "ds_ping_threshold", 3)
```

注意较新的版本已经移除了对于从文件读取配置的支持，因此DB的方式是必选的
`
已删除对文本文件（用于 Provisioning Destinations）的支持。现在只有数据库支持（通过数据库表进行预置）可用 - 如果您仍想使用文本文件进行预置，请使用 db_text DB 驱动程序（通过文本文件模拟的数据库）`

### 3. 调用方法

使用的方法很简单 `ds_select_dst(set, alg, [flags], [partition], [max_res])`，主要看选择的算法参数

|算法 ID	|算法名称	|描述|
|--|--|--|
|“0”| hash over callid| 根据callid hash值进行分配|
|“1”| hash over from uri |  根据请求 from uri hash值进行分配|
|“2”| hash over to uri |  根据请求 to uri hash值进行分配|
|“3”| hash over request-uri | 根据请求 ruri hash值进行分配|
|“4”| weighted round-robin |加权轮询 （下一个目标）|
|“5”| hash over authorization-username |授权用户名的哈希值（Proxy-Authorization 或 “normal” 授权）。如果未找到用户名，则使用加权轮询。|
|“6”| random (using rand())|随机 （使用 rand（））|
|“7”|hash over the content of PVs string| 对 PV 字符串的内容进行哈希处理。注： 仅当设置了参数 hash_pvar 时，此选项才有效|
|“8”| the first entry in set is chosen|选择 set 中的第一个条目|
|“9”| The pvar_algo_pattern parameter is used to determine the load on each server|pvar_algo_pattern 参数用于确定每台服务器上的负载。如果未指定参数，则选择集中的第一个条目|
|“10”| The algo_route OpenSIPS route is called for each dispatcher entry in the setid, in order to decide the routing order|为 setid 中的每个调度程序条目调用 algo_route OpenSIPS 路由，以确定路由顺序。有关使用示例，请参阅 algo_route 参数|
|“X”| if the algorithm is not implemented, the first entry in set is chosen|如果未实现算法，则选择 set 中的第一个条目。|

```conf
if (!ds_select_dst(1, 0)) {
	xlog("ERROR: no active destinations found!\n");
	send_reply(503, "Service Unavailable");
	exit;
}
...
ds_select_dst(1, 0, , "fs_boxes", 5);
...
ds_select_dst(1, 0, "fUD", "ask_boxes");
...
ds_select_dst(2, 0, "fud", "pstn_gws", 5);
ds_select_dst(3, 1, "fua", "pstn_gws", 2);
...
# using variables
$var(part) = "pstn_gws"
$var(setid) = 1;
$var(alg) = 4;
$var(flags) = "fdu";
$var(max_res) = 2;
ds_select_dst($var(setid), $var(alg), $var(flags), $var(part), $var(max_res));
...
```

### 4. 样例脚本

```conf
debug_mode=yes
socket= udp:127.0.0.1:5060

udp_workers = 2
check_via = off     # (cmd. line: -v)
dns = off           # (cmd. line: -r)
rev_dns = off       # (cmd. line: -R)

# for more info: opensips -h

# ------------------ module loading ----------------------------------
mpath="/usr/local//lib64/opensips/modules/"

loadmodule "maxfwd.so"
loadmodule "signaling.so"
loadmodule "sl.so"
loadmodule "tm.so"
loadmodule "rr.so"
loadmodule "sipmsgops.so"
loadmodule "db_mysql.so"
loadmodule "dispatcher.so"

loadmodule "proto_udp.so"

# ----------------- setting module-specific parameters ---------------
modparam("dispatcher", "db_url", "mysql://root:pass@127.0.0.1:3306/opensips")

route {
	if (!mf_process_maxfwd_header(10)) {
		send_reply(483, "Too Many Hops");
		exit;
	}

    # 记录路由
    if (!has_totag()) {
        record_route();
    }

    if(is_method("INVITE")) {  
		if (!ds_select_dst(1, 4)) {
			send_reply(503, "Service Unavailable");
			exit;
		}
	}

	t_relay();
}

route[MANAGE_FAILURE] {
    # 处理失败情况
    if (t_check_status("5[0-9][0-9]|408")) {
        ds_mark_dst("b");  # 标记失败的后端服务器
        t_relay();
    }
}

```

### 5. 测试和验证

在数据库写入分发配置数据，你可以采用页面配置的方法，API请求的方法，或者直接SQL插入的方法

```sql
INSERT INTO dispatcher (setid, destination, description)
VALUES
(1, 'sip:192.168.1.10:5060', 'Server 1'),
(1, 'sip:192.168.1.11:5060', 'Server 2'),
(1, 'sip:192.168.1.12:5060', 'Server 3');
```

启动 OpenSIPS：

```bash
/usr/sbin/opensips -f /etc/opensips/opensips.cfg
```

使用 SIP 客户端（如 Zoiper 或 Linphone）向 OpenSIPS 发送请求，观察请求是否被分发到不同的后端服务器。

使用 opensipsctl 工具查看 dispatcher 状态：

```bash
opensipsctl dispatcher list
```

