---
layout:     post
title:      sysbench-强大的性能基准测试工具黑马

subtitle:   sysbench mysql 
date:       2025-01-26
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - sysbench
    - mysql
---

一个很常见的需求，就是需要对部署之后的组件进行压测，验证其默认参数/调整后的参数，是否符合预期，是否满足交付条件
今天就介绍一个很常见的组件，sysbench!

- [sysbench 是怎么样一个东西？](#sysbench-是怎么样一个东西)
- [安装一个 sysbench](#安装一个-sysbench)
- [如何使用？](#如何使用)
- [压测一下 MySQL的性能](#压测一下-mysql的性能)
  - [脚本准备](#脚本准备)
  - [执行测试](#执行测试)

## sysbench 是怎么样一个东西？

它是一个模块化的、跨平台的基准测试工具，主要用于评估系统性能和数据库性能。它最初设计用于 CPU、内存、线程和文件 I/O 的基准测试，但后来扩展了对数据库操作的支持，成为评估数据库性能的强大工具。它可以支持多种数据库管理系统（DBMS），如 MySQL、PostgreSQL 和 MariaDB，并且可以通过编写自定义 Lua 脚本来支持其他类型的数据库。

上面介绍到的，比如服务器CPU、服务器内存、服务器I/O都是日常十分关注的基础指标。

内置模块有：

- CPU 测试：评估 CPU 性能。
- 内存测试：评估内存读写速度。
- 线程测试：评估线程创建和切换性能。
- 文件 I/O 测试：评估磁盘读写性能。
- OLTP 测试：评估数据库的在线事务处理（OLTP）性能。
- 可扩展性：用户可以编写自定义 Lua 脚本，以执行特定的测试逻辑或模拟复杂的业务场景。
- 测试报告：提供详细的性能统计数据，帮助识别性能瓶颈。

一个工具在github上面累积了 6.2k 的star数，它的功能一定长在了很多人的心坎上。

## 安装一个 sysbench

由于这个工具是如此常用，安装也是非常方便 ，直接从官方的安装包一键安装：

`Debian/Ubuntu`

```bash
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench
```

`RHEL/CentOS`

```bash
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
sudo yum -y install sysbench
```

`Fedora`

```bashbash
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash	
sudo dnf -y install sysbench
```

`Arch Linux`

```bash
sudo pacman -Suy sysbench
```

`macOS`

```bash
brew install sysbench
```

## 如何使用？

基础语法：`	  sysbench [options]... [testname] [command] `

`sysbench --help `可以帮你了解它的所有参数和用法

```bash
sysbench --help
Usage:
  sysbench [options]... [testname] [command]

Commands implemented by most tests: prepare run cleanup help

General options:
  --threads=N                     number of threads to use [1]
  --events=N                      limit for total number of events [0]
  --time=N                        limit for total execution time in seconds [10]
  --forced-shutdown=STRING        number of seconds to wait after the --time limit before forcing shutdown, or 'off' to disable [off]
  --thread-stack-size=SIZE        size of stack per thread [64K]
  --rate=N                        average transactions rate. 0 for unlimited rate [0]
  --report-interval=N             periodically report intermediate statistics with a specified interval in seconds. 0 disables intermediate reports [0]
  --report-checkpoints=[LIST,...] dump full statistics and reset all counters at specified points in time. The argument is a list of comma-separated values representing the amount of time in seconds elapsed from start of test when report checkpoint(s) must be performed. Report checkpoints are off by default. []
  --debug[=on|off]                print more debugging info [off]
  --validate[=on|off]             perform validation checks where possible [off]
  --help[=on|off]                 print help and exit [off]
  --version[=on|off]              print version and exit [off]
  --config-file=FILENAME          File containing command line options
  --tx-rate=N                     deprecated alias for --rate [0]
  --max-requests=N                deprecated alias for --events [0]
  --max-time=N                    deprecated alias for --time [0]
  --num-threads=N                 deprecated alias for --threads [1]

Pseudo-Random Numbers Generator options:
  --rand-type=STRING random numbers distribution {uniform,gaussian,special,pareto} [special]
  --rand-spec-iter=N number of iterations used for numbers generation [12]
  --rand-spec-pct=N  percentage of values to be treated as 'special' (for special distribution) [1]
  --rand-spec-res=N  percentage of 'special' values to use (for special distribution) [75]
  --rand-seed=N      seed for random number generator. When 0, the current time is used as a RNG seed. [0]
  --rand-pareto-h=N  parameter h for pareto distribution [0.2]

Log options:
  --verbosity=N verbosity level {5 - debug, 0 - only critical messages} [3]

  --percentile=N       percentile to calculate in latency statistics (1-100). Use the special value of 0 to disable percentile calculations [95]
  --histogram[=on|off] print latency histogram in report [off]

General database options:

  --db-driver=STRING  specifies database driver to use ('help' to get list of available drivers) [mysql]
  --db-ps-mode=STRING prepared statements usage mode {auto, disable} [auto]
  --db-debug[=on|off] print database-specific debug information [off]


Compiled-in database drivers:
  mysql - MySQL driver
  pgsql - PostgreSQL driver

mysql options:
  --mysql-host=[LIST,...]          MySQL server host [localhost]
  --mysql-port=[LIST,...]          MySQL server port [3306]
  --mysql-socket=[LIST,...]        MySQL socket
  --mysql-user=STRING              MySQL user [sbtest]
  --mysql-password=STRING          MySQL password []
  --mysql-db=STRING                MySQL database name [sbtest]
  --mysql-ssl[=on|off]             use SSL connections, if available in the client library [off]
  --mysql-ssl-cipher=STRING        use specific cipher for SSL connections []
  --mysql-compression[=on|off]     use compression, if available in the client library [off]
  --mysql-debug[=on|off]           trace all client library calls [off]
  --mysql-ignore-errors=[LIST,...] list of errors to ignore, or "all" [1213,1020,1205]
  --mysql-dry-run[=on|off]         Dry run, pretend that all MySQL client API calls are successful without executing them [off]

pgsql options:
  --pgsql-host=STRING     PostgreSQL server host [localhost]
  --pgsql-port=N          PostgreSQL server port [5432]
  --pgsql-user=STRING     PostgreSQL user [sbtest]
  --pgsql-password=STRING PostgreSQL password []
  --pgsql-db=STRING       PostgreSQL database name [sbtest]

Compiled-in tests:
  fileio - File I/O test
  cpu - CPU performance test
  memory - Memory functions speed test
  threads - Threads subsystem performance test
  mutex - Mutex performance test

See 'sysbench <testname> help' for a list of options for each test.
```

那么哪些参数是日常使用，比较重要的？

- `--threads`: 执行压测脚本的工作线程数，默认1个，但是如果要更好地模拟业务使用，肯定是一定数量的多线程数量，具体合适的值可以与开发人员确定。
- `--events`: 请求总数限制。0 （默认值） 表示无限制。
- `--time`: 整体执行的秒数，0代表无限制，默认10秒，通常这是代表一个观察周期，具体合适的值也需要与开发人员沟通
- `--warmup-time`: 在启用统计信息的实际基准测试运行之前，在禁用统计信息的情况下执行事件数秒。当您想从统计数据中排除基准测试运行的初始周期时，这非常有用。在许多基准测试中，初始阶段不具有代表性，因为 CPU/数据库/页面和其他缓存需要一些时间来预热。默认是0。
- `--rate`: 平均交易率。该数字指定所有线程平均每秒应执行多少个事件 （事务）。0（默认）表示无限制速率，即事件以最快的速度执行。
- `--thread-init-timeout`: 等待工作线程初始化的时间（以秒为单位），默认30秒。
- `--thread-stack-size`: 每个线程的堆栈大小，默认32K。
- `--report-interval`: 以指定的时间间隔（以秒为单位）定期报告中间统计信息。请注意，此选项生成的统计信息是按间隔生成的，而不是累积的。0 代表禁用。通常设置一个合理的值，过于密集可能影响测试结果。
- `--debug`: 打印更多调试信息，默认关闭。
- `--validate`:	尽可能对测试结果进行验证，默认关闭。
- `--help`
- `--verbosity`: 详细级别（0 - 仅关键消息，5 - 调试），默认是4。
- `--percentile`: 测量所有已处理请求的执行时间，默认是95，通常还有99,999,9999需求的测试，根据实际情况可调整。
- `--luajit-cmd`: 执行LuaJIT控制指令。

除了执行指令，剩下就是后续的指令：` prepare run cleanup help`

- `help` 就是展示常用的帮助页面
- `prepare` 就是执行脚本压测之前的准备工作，例如数据库就需要创建数据库、创建数据库表等
- `run` 就是真实开启执行压测
- `cleanup` 是执行压测后，对数据库表进行一些清理，便于下一次压测的开展

## 压测一下 MySQL的性能

### 脚本准备

我们在sysbench的目录下：

```bash
[@ sysbench]# ls -l
total 2097284
-rwxr-xr-x 1 root root      1452 Mar 16  2019 bulk_insert.lua
-rw-r--r-- 1 root root     14369 Mar 16  2019 oltp_common.lua
-rwxr-xr-x 1 root root      1290 Mar 16  2019 oltp_delete.lua
-rwxr-xr-x 1 root root      2415 Mar 16  2019 oltp_insert.lua
-rwxr-xr-x 1 root root      1265 Mar 16  2019 oltp_point_select.lua
-rwxr-xr-x 1 root root      1649 Mar 16  2019 oltp_read_only.lua
-rwxr-xr-x 1 root root      1824 Mar 16  2019 oltp_read_write.lua
-rwxr-xr-x 1 root root      1118 Mar 16  2019 oltp_update_index.lua
-rwxr-xr-x 1 root root      1127 Mar 16  2019 oltp_update_non_index.lua
-rwxr-xr-x 1 root root      1440 Mar 16  2019 oltp_write_only.lua
-rwxr-xr-x 1 root root      1919 Mar 16  2019 select_random_points.lua
-rwxr-xr-x 1 root root      2118 Mar 16  2019 select_random_ranges.lua
-rw------- 1 root root 429506560 Jan  8 09:55 test_file.0
-rw------- 1 root root 429506560 Jan  8 09:55 test_file.1
-rw------- 1 root root 429506560 Jan  8 09:55 test_file.2
-rw------- 1 root root 429506560 Jan  8 09:55 test_file.3
-rw------- 1 root root 429506560 Jan  8 09:55 test_file.4
drwxr-xr-x 4 root root        49 Jan  8 09:02 tests
```

能看到这些脚本，分别都是一些针对数据库压测的lua脚本，用于不同阶段，比如common可能代表压测前的准备和压测后的清理，有insert\delete\update\bulk insert等脚本，可以用于数据库读写的测试。

默认执行的sbtest表，结构十分简单

```sql
CREATE TABLE sbtest%d(
  id %s,
  k INTEGER DEFAULT '0' NOT NULL,
  c CHAR(120) DEFAULT '' NOT NULL,
  pad CHAR(60) DEFAULT '' NOT NULL,
  %s (id)
)
```

通常我们的测试场景为了尽可能准确，可能会使用一个更为接近的表字段、字段数量，以期望测出业务情况下更为准确的性能情况。
这个时候就需要我们自己来撰写压测脚本，就是修改其中的lua脚本，步骤包括：

- 需要自定义表结构，例如修改common中的建表语句

```sql
CREATE TABLE sbtest%d (
  id  %s,
  method char(16) NOT NULL DEFAULT '',
  callid char(64)   NOT NULL DEFAULT '',
  time datetime NOT NULL,
  duration int unsigned NOT NULL DEFAULT '0',
  ms_duration int unsigned NOT NULL DEFAULT '0',
  setuptime int NOT NULL DEFAULT '0',
  created datetime DEFAULT CURRENT_TIMESTAMP,
  src_ip varchar(25)   DEFAULT NULL,
  dst_ip varchar(25)   DEFAULT NULL,
  caller varchar(255)   DEFAULT NULL,
  callee varchar(255)   DEFAULT NULL,
  call_start_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3),
  call_end_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3),
  calllevel int unsigned NOT NULL DEFAULT '0',
  routinglevel int unsigned NOT NULL DEFAULT '0',
  calleraccount varchar(255)  SET utf8mb4 DEFAULT NULL,
  calleeaccount varchar(255)  SET utf8mb4 DEFAULT NULL,
  caller_callid char(100)   NOT NULL DEFAULT '',
  callee_callid char(100)   NOT NULL DEFAULT '',
  area varchar(10)   DEFAULT NULL,
  real_duration bigint unsigned DEFAULT '0',
   %s (id)
)
```

替换默认的sbtest，当然名称都可以修改

- 对于增删改查的SQL，根据自己的情况定义，修改字符和模拟值：

```lua
con:query(string.format("INSERT INTO %s (method, from_tag, to_tag, callid, sip_code, sip_reason, time, duration, ms_duration, setuptime, created, src_ip, dst_ip, caller, callee, call_start_time, call_end_time, caller_in, callee_in, caller_out, callee_out, callergateway, calleegateway, calllevel, routinglevel, calleraccount, calleeaccount, caller_callid, callee_callid, area, end_side, end_code, end_reason, real_duration) VALUES " ..
                                 "('%s', '%s', '%s','%s',%d,'%s','%s',%d,%d,'%s','%s','%s','%s','%s','%s','%s','%s','%s','%s','%s','%s','%s','%s',%d,%d,'%s','%s','%s','%s','%s',%d,%d,'%s',%d)",
                              table_name, c_val, pad_val, c_val, c_val,0, c_val, os.date("%Y-%m-%d %H:%M:%S",os.time()),0,0, os.date("%Y-%m-%d %H:%M:%S",os.time()), os.date("%Y-%m-%d %H:%M:%S",os.time()), c_val, c_val, c_val, c_val, os.date("%Y-%m-%d %H:%M:%S",os.time()), os.date("%Y-%m-%d %H:%M:%S",os.time()), c_val, c_val, c_val, c_val, c_val, c_val, 0,0,c_val, c_val, c_val, c_val, c_val,0,0, c_val,0))
```

### 执行测试

脚本一切准备完毕就可以开始使用命令开始测试：

- `prepare` 阶段

```bash
[@ sysbench]# sysbench /usr/share/sysbench/oltp_insert_new.lua --mysql-host=ip --mysql-port=port --mysql-user=user --mysql-password='pass' --mysql-db=dbname  --db-driver=mysql --tables=1 --table-size=1 --report-interval=10 --threads=1 --time=60 prepare
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Creating table 'sbtest1'...
Inserting 1 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
```

- `run` 阶段

```bash
[@ sysbench]# sysbench /usr/share/sysbench/oltp_insert_new.lua --mysql-host=ip --mysql-port=port --mysql-user=user --mysql-password='pass' --mysql-db=dbname  --db-driver=mysql --tables=1 --table-size=1 --report-interval=10 --threads=1 --time=60 run
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 1
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 1 tps: 506.41 qps: 506.41 (r/w/o: 0.00/506.41/0.00) lat (ms,95%): 2.35 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 1 tps: 472.72 qps: 472.72 (r/w/o: 0.00/472.72/0.00) lat (ms,95%): 2.76 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 1 tps: 430.74 qps: 430.74 (r/w/o: 0.00/430.74/0.00) lat (ms,95%): 2.97 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 1 tps: 489.78 qps: 489.78 (r/w/o: 0.00/489.78/0.00) lat (ms,95%): 2.39 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 1 tps: 514.12 qps: 514.12 (r/w/o: 0.00/514.12/0.00) lat (ms,95%): 2.26 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            0
        write:                           29224
        other:                           0
        total:                           29224
    transactions:                        29224  (487.02 per sec.)
    queries:                             29224  (487.02 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0038s
    total number of events:              29224

Latency (ms):
         min:                                    1.15
         avg:                                    2.05
         max:                                   30.09
         95th percentile:                        2.61
         sum:                                59930.19

Threads fairness:
    events (avg/stddev):           29224.0000/0.00
    execution time (avg/stddev):   59.9302/0.00
```

压测后上面这个报告就是最终说明问题的证据，满足/不满足就依靠上面的数据描述。

- `cleanup` 阶段

```bash
[@ sysbench]# sysbench /usr/share/sysbench/oltp_insert_new.lua --mysql-host=ip --mysql-port=port --mysql-user=user --mysql-password='pass' --mysql-db=dbname  --db-driver=mysql --tables=1 --table-size=1 --report-interval=10 --threads=1 --time=60 cleanup
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Dropping table 'sbtest1'...
```

