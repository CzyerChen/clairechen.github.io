---
layout:     post
title:      Kamailio-dispatcher负载均衡

subtitle:   kamailio dispatcher
date:       2024-09-12
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kamailio
    - sipp
---

Kamailio 负载均衡的功能主要依靠 Dispatcher 模块完成，[模块官方文档参看](https://www.kamailio.org/docs/modules/devel/modules/dispatcher.html)

- [为什么要引入负载均衡？](#为什么要引入负载均衡)
- [如何使用？](#如何使用)
	- [引入和配置](#引入和配置)
	- [功能路由调用](#功能路由调用)
	- [命令行指令](#命令行指令)

## 为什么要引入负载均衡？

Q: 如果单台VOIP服务的性能不能满足业务需求了，需要扩展至多个节点，那么Kamailio将如何进行请求的代理呢？

A: 就是需要负载均衡

Q: 如果多个后端VOIP服务有不可用，转过去的请求全部失败，那么Kamailio可以如何保障整体流程的健壮性？

A: 就是需要负载均衡剔除异常服务节点，转发至可用节点

此外，依赖负载均衡的基础功能，还可以实现AB测试、区分权重、通道量分发等业务相关逻辑

既然看上去功能强大且十分诱人的功能，要如何才能使用呢？

![img](../img/blog/Kamailio-Dispatcher.png)

## 如何使用？

这个模块是自带的，不需要额外加入并构建，在modules的目录下存在

### 引入和配置

在 `kamailio.cfg` 中，引入模块，并增加配置项

常规的DB方式配置

```xml
loadmodule "dispatcher.so"

# ----- dispatcher params -----
modparam("dispatcher", "db_url", DBURL)
modparam("dispatcher", "table_name", "dispatcher")
modparam("dispatcher", "flags", 2)
modparam("dispatcher", "xavp_dst", "_dsdst_")
modparam("dispatcher", "xavp_ctx", "_dsctx_")
modparam("dispatcher", "ds_ping_from", "sip:proxy@kamailio.org")
modparam("dispatcher", "ds_ping_interval", 60)
modparam("dispatcher", "ds_probing_mode", 1)
modparam("dispatcher", "ds_timer_mode", 1)
```

可以通过DB的方式加载分发的列表，也可以通过最基础的文件的方式引入，都支持动态加载

以下是文件方式

```xml
loadmodule "dispatcher.so"

# ----- dispatcher params -----
modparam("dispatcher", "list_file", "/usr/local/etc/kamailio/dispatcher.list")
modparam("dispatcher", "force_dst", 1)
#剩余参数取决于你的负载均衡模式
```

### 功能路由调用

```xml
request_route {
    ....
	# dispatch destinations
	route(DISPATCH);
}

# Dispatch requests
route[DISPATCH] {
	# round robin dispatching on gateways group '1'
	if(!ds_select_dst("1", "4")) {
		send_reply("404", "No destination");
		exit;
	}
	xdbg("--- SCRIPT: going to <$ru> via <$du> (attrs: $xavp(_dsdst_=>attrs))\n");
	t_on_failure("RTF_DISPATCH");
	route(RELAY);
	exit;
}
```

着重需要关注的方法就是 `ds_select_dst(set, alg[, limit])`，即挑选哪一个分组的地址列表，采用什么样的算法进行路由选择

第一个参数set，就是地址列表，第二参数alg就是算法，第三个limit是指在AVP列表中最多可以存储的item数 。

算法有很多种，也是对负载均衡流程核心的参数，除了12种官方支持的，还可以自定义开发去使用：

- “0” - hash over callid
- “1” - hash over from URI.
- “2” - hash over to URI.
- “3” - hash over request-URI user.
- “4” - round-robin (next destination).
- -“5” - hash over authorization-username (Proxy-Authorization or "normal" authorization). If no username is found, round robin is used.
- “6” - random destination (using rand()).
- “7” - hash over the content of PVs string. Note: This works only when the parameter hash_pvar is set.
- “8” - select destination sorted by priority attribute value (serial forking ordered by priority).
- “9” - use weight based load distribution. You have to set the attribute 'weight' for each address (gateway) in destination set. See also the description of the 'weight' attribute in the 'Special Attributes' section.
- “10” - use call load distribution. You have to set the attribute 'duid' (as an unique string id) per each address in destination set. Also, you must set the parameter 'ds_hash_size'.
- “11” - use relative weight based load distribution. You have to set the attribute 'rweight' per each address in destination set. Active host usage probability is rweight/(SUM of all active host rweights in destination group).
- “12” - dispatch to all destination in setid at once (parallel forking). Note that the XAVPs are no longer set with the values of the destination records, no re-routing making sense in this case.
- “13” - latency optimized dispatching
- “64” - round-robin (next destination) with over load control (destination record is skipped based on over load control rate).
- “X” - if the algorithm is not implemented, the first entry in set is chosen.

默认样例都是用round-robin去做负载，初期使用可以用这个方式，后续生产中，应该 9、10、11、13 结合实际业务压力做更好地拥塞控制

alg-4 测试示例:

```yml
...
#!KAMAILIO
#
# sample config file for dispatcher module
# - load balancing of VoIP calls with round robin
# - no TPC listening
# - don't dispatch REGISTER and presence requests
#
# Kamailio SIP Server
#     - web: https://www.kamailio.org
#     - git: http://github.com/kamailio/
#
# Direct your questions about this file to: sr-users@lists.kamailio.org
#
# Refer to the Core CookBook at https://www.kamailio.org/w/documentation/
# for an explanation of possible statements, functions and parameters.
#
# Several features can be enabled using '#!define WITH_FEATURE' directives:
#
# *** To run in debug mode:
#     - define WITH_DEBUG
#

#!ifndef DBURL
#!define DBURL "mysql://kamailio:kamailiorw@localhost/kamailio"
#!endif

# - flags
#   FLT_ - per transaction (message) flags
#	FLB_ - per branch flags
#!define FLT_ACC 1
#!define FLT_ACCMISSED 2
#!define FLT_ACCFAILED 3

####### Global Parameters #########

#!ifdef WITH_DEBUG
debug=4
log_stderror=yes
#!else
debug=2
log_stderror=no
#!endif

memdbg=5
memlog=5

log_facility=LOG_LOCAL0

fork=yes
children=4

/* comment the next line to enable TCP */
disable_tcp=yes

/* uncomment the next line to disable the auto discovery of local aliases
   based on revers DNS on IPs (default on) */
auto_aliases=no

/* add local domain aliases */
# alias="mysipserver.com"

port=5060

/* uncomment and configure the following line if you want Kamailio to
   bind on a specific interface/port/proto (default bind on all available) */
# listen=udp:127.0.0.1:5060

sip_warning=no

####### Modules Section ########

# set module path
#mpath="/usr/local/lib/kamailio/modules/"

loadmodule "db_mysql.so"
loadmodule "jsonrpcs.so"
loadmodule "kex.so"
loadmodule "corex.so"
loadmodule "tm.so"
loadmodule "tmx.so"
loadmodule "sl.so"
loadmodule "rr.so"
loadmodule "pv.so"
loadmodule "maxfwd.so"
loadmodule "textops.so"
loadmodule "siputils.so"
loadmodule "xlog.so"
loadmodule "sanity.so"
loadmodule "ctl.so"
loadmodule "acc.so"
loadmodule "dispatcher.so"


# ----------------- setting module-specific parameters ---------------


# ----- jsonrpcs params -----
modparam("jsonrpcs", "pretty_format", 1)


# ----- rr params -----
# add value to ;lr param to cope with most of the UAs
modparam("rr", "enable_full_lr", 1)
# do not append from tag to the RR (no need for this script)
modparam("rr", "append_fromtag", 0)


# ----- acc params -----
modparam("acc", "log_flag", FLT_ACC)
modparam("acc", "failed_transaction_flag", FLT_ACCFAILED)
modparam("acc", "log_extra",
	"src_user=$fU;src_domain=$fd;dst_ouser=$tU;dst_user=$rU;dst_domain=$rd;src_ip=$si")

# ----- tm params -----
modparam("tm", "fr_timer", 2000)
modparam("tm", "fr_inv_timer", 40000)

# ----- dispatcher params -----
modparam("dispatcher", "db_url", DBURL)
modparam("dispatcher", "table_name", "dispatcher")
modparam("dispatcher", "flags", 2)
modparam("dispatcher", "xavp_dst", "_dsdst_")
modparam("dispatcher", "xavp_ctx", "_dsctx_")
modparam("dispatcher", "ds_ping_from", "sip:proxy@kamailio.org")
modparam("dispatcher", "ds_ping_interval", 60)
modparam("dispatcher", "ds_probing_mode", 1)
modparam("dispatcher", "ds_timer_mode", 1)

####### Routing Logic ########


# main request routing logic

request_route {

	# per request initial checks
	route(REQINIT);

	# CANCEL processing
	if (is_method("CANCEL")) {
		if (t_check_trans()) {
			route(RELAY);
		}
		exit;
	}

	# handle retransmissions
	if (!is_method("ACK")) {
		if(t_precheck_trans()) {
			t_check_trans();
			exit;
		}
		t_check_trans();
	}

	# handle requests within SIP dialogs
	route(WITHINDLG);

	### only initial requests (no To tag)

	# record routing for dialog forming requests (in case they are routed)
	# - remove preloaded route headers
	remove_hf("Route");
	if (is_method("INVITE|SUBSCRIBE")) {
		record_route();
	}

	# account only INVITEs
	if (is_method("INVITE")) {
		setflag(FLT_ACC); # do accounting
	}

	# handle presence related requests
	route(PRESENCE);

	# handle registrations
	route(REGISTRAR);

	if ($rU==$null) {
		# request with no Username in RURI
		sl_send_reply("484","Address Incomplete");
		exit;
	}

	# dispatch destinations
	route(DISPATCH);
}


route[RELAY] {
	if (!t_relay()) {
		sl_reply_error();
	}
	exit;
}

# Per SIP request initial checks
route[REQINIT] {
	if (!mf_process_maxfwd_header("10")) {
		sl_send_reply("483","Too Many Hops");
		exit;
	}

	if(!sanity_check("1511", "7")) {
		xlog("Malformed SIP message from $si:$sp\n");
		exit;
	}
}

# Handle requests within SIP dialogs
route[WITHINDLG] {
	if (!has_totag()) {
		return;
	}

	# sequential request within a dialog should
	# take the path determined by record-routing
	if (loose_route()) {
		if (is_method("BYE")) {
			setflag(FLT_ACC); # do accounting ...
			setflag(FLT_ACCFAILED); # ... even if the transaction fails
		}
		route(RELAY);
	}

	if (is_method("SUBSCRIBE") && uri == myself) {
		# in-dialog subscribe requests
		route(PRESENCE);
	}

	if ( is_method("ACK") ) {
		if ( t_check_trans() ) {
			# non loose-route, but stateful ACK;
			# must be ACK after a 487 or e.g. 404 from upstream server
			t_relay();
			exit;
		} else {
			# ACK without matching transaction ... ignore and discard.
			exit;
		}
	}

	sl_send_reply("404","Not here");
	exit;
}

# Handle SIP registrations
route[REGISTRAR] {
	if(!is_method("REGISTER"))
		return;

	sl_send_reply("404", "No registrar");
	exit;
}

# Presence server route
route[PRESENCE] {
	if(!is_method("PUBLISH|SUBSCRIBE"))
		return;

	sl_send_reply("404", "Not here");
	exit;
}

# Dispatch requests
route[DISPATCH] {
	# round robin dispatching on gateways group '1'
	if(!ds_select_dst("1", "4")) {
		send_reply("404", "No destination");
		exit;
	}
	xdbg("--- SCRIPT: going to <$ru> via <$du> (attrs: $xavp(_dsdst_=>attrs))\n");
	t_on_failure("RTF_DISPATCH");
	route(RELAY);
	exit;
}

# Try next destionations in failure route
failure_route[RTF_DISPATCH] {
	if (t_is_canceled()) {
		exit;
	}
	# next DST - only for 500 or local timeout
	if (t_check_status("500")
			or (t_branch_timeout() and !t_branch_replied())) {
		if(ds_next_dst()) {
			xdbg("--- SCRIPT: retrying to <$ru> via <$du> (attrs: $xavp(_dsdst_=>attrs))\n");
			t_on_failure("RTF_DISPATCH");
			route(RELAY);
			exit;
		}
	}
}

...
```

aalg-11 测试示例:

```yml
...
# sample of SQL provisioning statements
INSERT INTO "dispatcher"
VALUES(1,1,'sip:192.168.0.1:5060',0,12,'rweight=50;weight=50;cc=1;','');
INSERT INTO "dispatcher"
VALUES(2,1,'sip:192.168.0.2:5060',0,12,'rweight=50;weight=50;cc=1;','');
...
modparam("dispatcher", "ds_ping_interval", 1) # ping gateways once/second
modparam("dispatcher", "ds_ping_latency_stats", 1) # update congestion metrics
# configure the latency estimator
modparam("dispatcher", "ds_latency_estimator_alpha", 900)
...
if (!ds_select_dst("1", "11")) { # use relative weight based load distribution
...
# sample of output from 'kamcmd dispatcher.list'
DEST: {
	URI: sip:192.168.0.1:5060
	FLAGS: AP
	PRIORITY: 12
	ATTRS: {
		BODY: rweight=50;weight=50;cc=1 # configuration values
		DUID:
		MAXLOAD: 0
		WEIGHT: 50
		RWEIGHT: 50
		SOCKET:
		SOCKNAME:
		OBPROXY:
	}
	LATENCY: {
		AVG: 20.104000
		STD: 1.273000
		# estimated congestion is currently 25ms = 45ms(EST) -20ms(AVG)
		EST: 45.005000
		MAX: 132
		TIMEOUT: 3
	}
}
...
```

### 命令行指令

```bash
# prototype: kamcmd dispatcher.set_state _state_ _group_ _address_
kamcmd dispatcher.set_state ip 2 sip:127.0.0.1:5080
kamcmd dispatcher.set_state ip 3 all

# prototype: kamcmd dispatcher.set_duid_state _state_ _group_ _duid_
kamcmd dispatcher.set_duid_state ip 2 xyz

kamcmd dispatcher.list
...
DEST: {
        URI: sip:192.168.0.1:5060
        FLAGS: AP
        PRIORITY: 12
}
...

kamcmd dispatcher.reload

# prototype: kamcmd dispatcher.ping_active _state_
kamcmd dispatcher.ping_active 0

# prototype: kamcmd dispatcher.add _group_ _address_ _flags_ _priority_ _attrs_
kamcmd dispatcher.add 2 sip:127.0.0.1:5080
kamcmd dispatcher.add 3 sip:127.0.0.1:5075 8
kamcmd dispatcher.add 3 sip:127.0.0.1:5075 0 0 duid=abc;socket=udp:127.0.0.1:5060
...

# prototype: kamcmd dispatcher.remove _group_ _address_
kamcmd dispatcher.remove 2 sip:127.0.0.1:5080
kamcmd dispatcher.remove 3 sip:127.0.0.1:5075;transport=udp

# prototype: kamctl rpc dispatcher.hash _nslots_ _val1_ [_val2_]
kamctl rpc dispatcher.hash 0 alice server.com
kamctl rpc dispatcher.hash 4 bob server.com

# prototype: kamcli dispatcher.oclist _group_
kamcli dispatcher.oclist 1
```

最为常用的就是增、删、查、热加载，因为通常是新节点加入、老节点剔除、配置热加载这样的需求。

