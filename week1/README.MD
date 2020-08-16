# Week 1 homework

## 课程作业

分值：200

题目描述：

本地下载 TiDB，TiKV，PD 源代码，改写源码并编译部署以下环境：

- 1 TiDB
- 1 PD
- 3 TiKV

改写后：使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的 日志

输出：一篇文章介绍以上过程

截止时间：本周日 24:00:00（逾期不给分）
作业提交方式：提交至个人 github ，将链接发送给 talent-plan@tidb.io

## 完成作业

环境: darwin

### 编译

分别在 https://github.com/pingcap/pd, https://github.com/pingcap/tidb, https://github.com/tikv/tikv 克隆源码

#### tikv 编译

无脑 `make` 一下, 25min52s 后编译完成, `target/release` 目录下得到 `tikv-ctl`, `tikv-server`.

#### tidb 编译

无脑 `make` 一下, 4min 后编译完成, `bin` 目录下得到 `pd-ctl`, `pd-recover`,`pd-server`.

#### pd 编译

无脑 `make` 一下,1min54 后编译完成, `bin` 目录下得到 `tidb-server`.

### 启动

整理一下需要的二进制文件:

```shell
❯ ./tidb-server -V
Release Version: v4.0.0-beta.2-960-g5184a0d70
Edition: Community
Git Commit Hash: 5184a0d7060906e2022d18f11532f119f5df3f39
Git Branch: master
UTC Build Time: 2020-08-15 09:01:48
GoVersion: go1.14.4
Race Enabled: false
TiKV Min Version: v3.0.0-60965b006877ca7234adaced7890d7b029ed1306
Check Table Before Drop: false

❯ ./pd-server --version
Release Version: v4.0.0-rc.2-140-g865fbd82
Edition: Community
Git Commit Hash: 865fbd82a028aecfb875a20b932fee3ba4b8c73c
Git Branch: master
UTC Build Time:  2020-08-15 09:01:06

❯ ./tikv-server --version
TiKV
Release Version:   4.1.0-alpha
Edition:           Community
Git Commit Hash:   ae7a6ecee6e3367da016df0293a9ffe9cc2b5705
Git Commit Branch: master
UTC Build Time:    2020-08-15 08:39:17
Rust Version:      rustc 1.46.0-nightly (16957bd4d 2020-06-30)
Enable Features:   jemalloc sse protobuf-codec
Profile:           release
```

在 https://docs.pingcap.com/tidb/dev/command-line-flags-for-tidb-configuration 参考启动参数, 最终的启动命令如下:

pd

```shell
./pd-server --client-urls http://0.0.0.0:2379 --data-dir /tmp/tidb-cluster/pd --log-file /tmp/tidb-cluster/logs/pd/pd.log
```

tikv1

```shell
./tikv-server --config $HOME/temp/tidb-cluster/config/tikv.toml --pd 127.0.0.1:2379 -A 0.0.0.0:20160 --advertise-addr 192.168.1.240:20160 --status-addr 0.0.0.0:20180 --data-dir /tmp/tidb-cluster/tikv1 --log-file /tmp/tidb-cluster/logs/tikv1/tikv.log
```

tikv2

```shell
./tikv-server --config $HOME/temp/tidb-cluster/config/tikv.toml --pd 127.0.0.1:2379 -A 0.0.0.0:20161 --advertise-addr 192.168.1.240:20161 --status-addr 0.0.0.0:20181 --data-dir /tmp/tidb-cluster/tikv2 --log-file /tmp/tidb-cluster/logs/tikv2/tikv.log
```

tikv3

```shell
./tikv-server --config $HOME/temp/tidb-cluster/config/tikv.toml --pd 127.0.0.1:2379 -A 0.0.0.0:20162 --advertise-addr 192.168.1.240:20162 --status-addr 0.0.0.0:20182 --data-dir /tmp/tidb-cluster/tikv3 --log-file /tmp/tidb-cluster/logs/tikv3/tikv.log
```

> 由于我是直接在 macos 上启动 tikv, 一开始启动时报了`the maximum number of open file descriptors is too small`. 参考 tiup 更改了 rocksdb 和 raftdb 的 max-open-files 得以解决.

tidb

```shell
./tidb-server --store tikv --path 127.0.0.1:2379 --log-file /tmp/tidb-cluster/logs/tidb/tidb.log
```

尝试连接

```shell
❯ mysql -h127.0.0.1 -P4000 -uroot
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.25-TiDB-v4.0.0-beta.2-960-g5184a0d70 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

### 启动事务时打印消息

分析题意, 我理解为在**执行 SQL**时, 开启事务时, 打印"hello transaction".

事务作为一个重要的概念, 肯定是有抽象的. 找到源码, 搜索`transaction`类型, 果然搜到一个接口:

![](https://i.imgur.com/45tDOSY.png)

这个接口的实现有很多, 但是有两个实现比较在意:`TxnState`和`tikvTxn`;  
研究了一下, `tikvTxn`是真正承载事务的实现, 而每个 session 中 hold 的 txn 其实是`TxnState`, 而且开启事务在`session#Txn(active bool)`这个方法中;

这应该就是我们要找的东西; 简单看了下代码后, 决定在`session/session.go`的 1454 行加入`logutil.BgLogger().Info("hello transaction")`;

测试效果如下:
![](https://i.imgur.com/9dOZFXC.png)

> 看样子会有一个每隔三秒的 `LoadBindInfoLoop` 行为, 会记录一些元信息到数据库里; 好吧, 框框里是我随便写的 select 触发的 txn;
