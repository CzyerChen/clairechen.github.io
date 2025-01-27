---
layout:     post
title:      Kamailio-HTable 万年缓存技术

subtitle:   Kamailio HTable
date:       2024-10-17
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kamailio
    - HTable
---

在程序开发的时候我们都知道，缓存可以存放临时的数据、高性能需求的数据，典型的就是Redis，使用高效的Hash数据结构提供极致的性能需求，而不是依赖数据库。那么在Kamailio这边，也有一些这样的需求，比如黑白名单、访问频次、业务静态属性等，Htable出现了。

速览：

- [HTable](#htable)
- [功能介绍](#功能介绍)
  - [从数据库初始化](#从数据库初始化)
  - [如何定义 htable 数据](#如何定义-htable-数据)
  - [如何写入和读取？](#如何写入和读取)
  - [伪变量](#伪变量)
  - [RPC指令](#rpc指令)

## HTable

Htable 正如它的名字，它就是一个哈希表结构容器，在共享内存中保存，不同线程可以共同读取，可以支持多种类型的定义，可以支持从数据库载入缓存。在Sip服务器中一个典型的场景就是作为缓存系统，元素也支持过期。

什么是hash table ? http://en.wikipedia.org/wiki/Hash_table.

## 功能介绍

### 从数据库初始化

如果配置了DB url，就能够从数据库获取初始化数据，那么就是取决于四个字段key name、key type、value type、key value，当然这四个字段也支持自定义。你可以选择配置指定的字段，或者独立设置一个与之对应的只读视图来获取数据

- `key name`: string类型，是key名
- `key type`: key的类型
  - 0 代表 普通类型，key就是'key_name'
  - 1 代表 数组类型，key就是'key_name[n]'
- `value type`：内容的类型
  - 0 - string类型
  - 1 - integer类型
- `key value`：string类型 key的内容

```bash
modparam("htable", "key_name_column", "kname") #Default value is 'key_name'.
modparam("htable", "key_type_column", "ktype") #Default value is 'key_type'.
modparam("htable", "value_type_column", "vtype") #Default value is 'value_type'.
modparam("htable", "key_value_column", "kvalue") #Default value is 'key_value'.
modparam("htable", "expires_column", "expiry") #Default value is 'expires'.
```

使用如下配置数据库信息：

```bash
modparam("htable", "db_url", "mysql://kamailio:kamailiorw@localhost/kamailio")
```

如果你是pg，那么

```bash
modparam("htable", "db_url", "postgres://kamailio:kamailiorw@localhost/kamailio")
```

### 如何定义 htable 数据

`"htname=>size=_number_;autoexpire=_number_;dbtable=_string_"`

- htname 是具体的哈希表名
- size 来控制要为哈希表创建的槽 （存储桶） ，数越大，意味着插槽越多，冲突越少的可能性越高。为表创建的实际数字槽（或存储桶）为 2^size。此值的可能范围为 2 到 31，较小或较大的值将增加到 3（8 个槽）或减少到 14（16384 个槽）。请注意，当为键计算的哈希 ID 发生冲突时，每个槽可以存储多个项目。同一插槽中的项目存储在链表中。换句话说，大小并没有设置哈希表中可以存储的项目数量的限制，只要有足够的可用共享内存，就可以添加新项目
- autoexpire 如果未对哈希表进行更新，则从哈希表中删除项目的时间（以秒为单位）。如果缺少或设置为 0，则项目不会过期
- dbtable 启动时要加载的数据库名称，不写就不加载
- cols 列名
- dbmode 0 不回写入DB-默认，1 服务暂停时回写入DB
- initval 返回为空时，会返回一个数字，而不是null
- updateexpire 如果设置为1，则一个值的过期时间会被充值当值被更新
- dmqreplicate 如果设置为1，就会通过DMQ模块，将数据同步到其他副本节点
- coldelim 列分隔符
- colnull 数据库里面Null值进入htable可转的值，一般使用就是null转空字符串

结合以上定义，来看看这些样例

`modparam("htable", "htable", "a=>size=4;autoexpire=7200;dbtable=htable_a;")`
缓存名叫a, 2小时自动过期，db里面加载过来的表名叫htable_a

`modparam("htable", "htable", "b=>size=5;")`
缓存名叫b, 为哈希表初始换2^5个卡槽

`modparam("htable", "htable", "c=>size=4;autoexpire=7200;initval=1;dmqreplicate=1;")`
缓存名叫c,为哈希表初始换2^4个卡槽，2小时自动过期，返回为空默认为1，使用DMQ进行副本数据同步

其他参数：

- array_size_suffix (str) 要添加的后缀
- fetch_rows (integer) 一次要从数据库中获取多少行
- timer_interval (integer) 检查过期 htable 值的间隔（s）
- db_expires (integer) 如果设置为 1，则模块从/到数据库加载/保存哈希表中项目的过期值。这个值的刷新最终还是需要依赖RPC指令，不是服务自行完成
- enable_dmq (integer) 如果设置为 1，则开启DMQ复制
- dmq_init_sync (integer) 如果设置为 1，将在启动时请求其他节点同步
- timer_procs (integer) 如果设置为 1 或更大，则模块将创建自己的计时器进程来扫描哈希表中的过期数据
- event_callback (str) 事件回调定义
- event_callback_mode (int) 控制何时执行 event_route[htable：init]： 0 - 所有模块初始化后;1 - 在第一个工作进程中

### 如何写入和读取？

**增**

```bash
$sht(test=>x)='aaa';
$sht(test=>5)=0;
$sht(test=>x[0])='abc';
```

**删**

```bash
#在ha中删除key为test的数据
sht_rm("ha", "test"); 

#按照key正则表达去删除
sht_rm_name_re("ha=>.*");

#按照value正则表达去删除
sht_rm_value_re("ha=>.*");
```

`sht_rm_name("ha", "re", ".*")`，sht_rm_name(htable, op, val) 更多灵活的批量根据keyname的情况去删除方式
The op parameter:

- re - match the val parameter as regular expression.
- sw - match the val parameter as 'starts with'.
- ew - match the val parameter as 'ends with'.
- in - match the val parameter as 'includes'.

value也是同样的：`sht_rm_value("ha", "re", ".*");`, sht_rm_value(htable, op, val)

- re - match the val parameter as regular expression.
- sw - match the val parameter as 'starts with'.
- ew - match the val parameter as 'ends with'.
- in - match the val parameter as 'includes'.

`sht_reset("ha$var(x)");` 删除哈希表中所有内容

**改**

修改string类型：

`sht_setxs("ha", "test", "abc", "10");` 将test=10 改为 test=abc，过期test=10 并将 test设置为abc

修改integer类型：

`sht_setxs("ha", "test", "100", "10");`


**查**

`sht_match_name(htable, op, mval)` 对于key查询，支持四种匹配方式

- eq - match the val parameter as string equal expression.
- ne - match the val parameter as string not-equal expression.
- re - match the val parameter as regular expression.
- sw - match the val parameter as 'starts with' expression.

以下都是类似的：

`sht_has_name(htable, op, mval)`
`sht_match_str_value(htable, op, mval)`
`sht_has_str_value(htable, op, mval)`

**其他**

`sht_lock(htable=>key)` 锁定key
`sht_unlock(htable=>key)` 解锁key


`sht_iterator_start("i1", "h1");`
`sht_iterator_end("i1");`
`sht_iterator_next(iname)`
`sht_iterator_rm(iname)`
`sht_iterator_sets(iname, sval)`
`sht_iterator_seti(iname, ival)`
`sht_iterator_setex(iname, exval)`

### 伪变量

- $sht(htable=>key)

- $shtex(htable=>key)

- $shtcn(htable=>key)

- $shtcv(htable=>key)

- $shtinc(htable=>key)

- $shtitkey(iname)

- $shtitval(iname)

- $shtrecord(attribute)

### RPC指令

- htable.get htable key 获取一个值
  
```bash
...
# Dump $sht(students=>alice)
kamcmd htable.get students alice

# Dump $sht(students=>15)
kamcmd htable.get students s:15

# Dump first entry in array key course $sht(students=>course[0])
kamcmd htable.get students course[0]
```

- htable.delete htable key 删除一个值

```bash
# Delete $sht(students=>alice)
kamcmd htable.delete students alice

# Delete $sht(students=>15)
kamcmd htable.delete students s:15

# Delete first entry in array key course $sht(students=>course[0])
kamcmd htable.delete students course[0]
```

- htable.sets htable key value 设置一个字符串的值

```bash
# Set $sht(test=>x) as string
kamcmd htable.sets test x abc

# Set $sht(test=>5) as string
kamcmd htable.sets test s:5 abc

# Set firsti entry in array key x $sht(test=>x[0]) as string
kamcmd htable.sets test x[0] abc
```

- htable.seti htable key value 设置一个数字的值

```bash
# Set $sht(test=>x) as integer
kamcmd htable.seti test x 123

# Set $sht(test=>5) as integer
kamcmd htable.seti test s:5 123

# Set first entry in array key x $sht(test=>x[0]) as integer
kamcmd htable.seti test x[0] 123
```

- htable.setex htable key expire、htable.setxs htable key value expire、 htable.setxi htable key value expire 过期一个key，顺便是否需要设置一个新的值(字符串或数字)
- htable.dump htable 导出哈希表所有数据
- htable.reload htable 重新加载刷新哈希表
- htable.store htable 将哈希表保存到数据库
- htable.flush htable 清空哈希表
- htable.listTables 列出所有表格
- htable.stats 统计
- htable.dmqsync [htable] dmq同步
