---
title: Drainer & Binlog crc mismatch
date: 2018-05-12T00:00:00+08:00
lastmod: 2018-05-22T00:00:00+08:00
categories:
  - Troubleshooting
tags:
  - Binlog
  - Drainer
  - PUMP
---
## 0x00 借口

> 该问题出现时间为 2018 年 5 月 12 日，影响版本范围在 1.0.x & 2.0 附近；基本上算运维事故，一定要配置告警啊，就算系统不重要

- 现象总结
  - 出现 crc mismatch 可能是 pump binlog 数据损坏，drainer 对 binlog 内容校验失败
  - tidb 进程存在，业务端口不可用, 当 pump 不可用后，tidb 自动丢掉业务端口
    - `listener stopped, waiting for manual kill`
- 触发原因可能是
  - 服务器重启
  - kill -9
  - 磁盘写满

### Drainer Log

> Drainer 服务日志持续刷以下日志  
> Drainer 需要去每个 PUMP 节点收集 Binlog 数据，汇总时按照时许排序并做一致性校验，根据提示看时数据校验未通过  

```log
2018/01/08 15:44:59 pump.go:386: [warning] [stream] node ip-10-0-70-42:8250, pos {Suffix:351 Offset:520960160}, error rpc error: code = Unknown desc = binlogger: crc mismatch
2018/01/08 15:45:00 pump.go:386: [warning] [stream] node ip-10-0-70-40:8250, pos {Suffix:354 Offset:182450922}, error rpc error: code = Unknown desc = binlogger: crc mismatch
```

### PUMP Log

> 在 drainer 出现问题集群中发现一个 PUMP 磁盘写满的现象，猜测可能是这里先出现写满然后导致部分数据未写入成功  

```log
2018/01/05 11:54:28 server.go:364: ^[[0;31m[error] generate forward binlog, write binlog err write /data1/deploy/data.pump/clusters/6485556578803970722/binlog-0000000000000354: no space left on device^[[0m
```

### 修复

根据官方指导，这种情况只能通过重新搭建 Binlog 架构进行修复。重新开始做以下流程：

1. Binlog 重新搭建，根据数据情况调整 PUMP 节点 GC 参数，默认 7 天开始 GC Binlog
2. 获取上游主库的 tso 时间点，也就是断点信息
3. 数据的全量导出：使用 mydumper 工具导出，数据导出时间视数据量、并发数、磁盘读写速度、网络速度而定
4. 全量导入：使用的官方 loader 工具，没办法使用 myloader 或者 mysqlload 工具，因为事物大小问题无法提交
5. 开启 Drainer 服务：使用第二步获取的 tso 断点信息，如果遇见重复数据，drainer 开启时有个 5 分钟的 safe mode 模式可以强制刷写下游数据
   - 其实就是开启了 sql 转换，比如 insert 转 replace、update 转 delete + replace
6. 注意观察监控，比如 binlog 上下游延迟、binlog tps 消费速度、磁盘空间大小，最后设置下告警

## 0x01 买一送一

> Drainer 同步下游是个 mysql 数据库，Drainer 同步采用的是高并发提交事物模式写入数据，偶遇一个死锁错误  
> 这种错误只能靠人工清理了，备库无业务流量，选择了直接重启处理 死锁 deadlock  

```log
2018/01/08 20:09:23 syncer.go:517: [fatal] Error 1213: Deadlock found when trying to get lock; try restarting transaction
/home/pingcap/go/src/github.com/pingcap/tidb-binlog/drainer/executor/util.go:72:
/home/pingcap/go/src/github.com/pingcap/tidb-binlog/drainer/executor/util.go:52:
```

## 0x02 再送一条存货

> Drainer 数据同步下游慢，PUMP GC Binlog 的惨案

早起刚做 Binlog 同步时，上游是 `TiDB * 4、PD * 3、TiKV * 9` 规模的集群，下游使用 8U 16G 虚拟机外加 2T sas 磁盘安装了一个 mysql 服务做数据同步。当初没有考虑同步性能，刚开始数据量比较小，虽然有些延迟但是能忍。  
逐渐逐渐发现越来越慢，终于有一天业务提示下游备库查不到数据了（刚开始也没有配置同步链路告警，被踢了一觉）。  
然后开始排查 Drainer & PUMP 日志，看见 PUMP 日志中提示 `commitTs(396914383516074000) of binlog exceeds the lower boundary of window 396981384481603586, may miss processing, ITEM(&{0xc50ebf8980 {75 0} tidb-online-tidb-2:8250 <nil>})ESC[0m` 信息，看的我一脸懵逼。咨询官方后说是 PUMP 已经 GC binlog 了，Drainer 拿不到数据所以同步就断了。  
详细的解释就是下面这样：

1. a b c 三个 pump 共存储了 100 条数据，每个 pump 隔三差五的存了一些数据
2. Drainer 同步到了 77 ，想要 78 的时候发现是在 a pump 节点
3. a pump gc 时间到了，在早些时间已经把 78 数据删除了
4. Drainer 拿不到数据啊，就炸火了
5. 优化方案两种，第一个是调整 PUMP GC 时间、第二个是改善下游机器配置
   - 因为 xx 原因，先把 PUMP GC 时间调大了…… ，然后就出现上面的惨案，磁盘写满

### Drainer log2

```log
2017/12/27 07:40:06 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-1:8250, pos {Suffix:80 Offset:495074285}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/27 08:14:25 pump.go:386: ESC[0;33m[warning] [stream] node ip-10-10-8-6:8250, pos {Suffix:76 Offset:392531590}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/27 09:59:30 pump.go:386: ESC[0;33m[warning] [stream] node ip-10-10-8-7:8250, pos {Suffix:80 Offset:39373906}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/27 12:14:59 pump.go:386: ESC[0;33m[warning] [stream] node ip-10-10-8-6:8250, pos {Suffix:76 Offset:472984304}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/27 15:12:19 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-1:8250, pos {Suffix:81 Offset:111758786}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/27 16:36:42 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-2:8250, pos {Suffix:77 Offset:26308812}, error rpc error: code = Internal desc = transport is closingESC[0m
2017/12/27 16:36:42 pump.go:386: ESC[0;33m[warning] [stream] node ip-10-10-8-6:8250, pos {Suffix:77 Offset:26308812}, error rpc error: code = Internal desc = transport is closingESC[0m
2017/12/27 16:36:45 collector.go:184: ESC[0;37m[info] node(tidb-online-tidb-2:8250) of cluster(6493070954582498647)  has been removed and release the connection to itESC[0m
2017/12/27 16:36:45 pump.go:378: ESC[0;33m[warning] [Get pull binlogs stream ip-10-10-8-6:8250] rpc error: code = Unavailable desc = grpc: the connection is unavailableESC[0m
2017/12/27 16:36:48 pump.go:378: ESC[0;33m[warning] [Get pull binlogs stream ip-10-10-8-6:8250] rpc error: code = Unavailable desc = grpc: the connection is unavailableESC[0m
2017/12/27 16:36:51 pump.go:378: ESC[0;33m[warning] [Get pull binlogs stream ip-10-10-8-6:8250] rpc error: code = Unavailable desc = grpc: the connection is unavailableESC[0m
2017/12/27 16:36:54 pump.go:378: ESC[0;33m[warning] [Get pull binlogs stream ip-10-10-8-6:8250] rpc error: code = Unavailable desc = grpc: the connection is unavailableESC[0m
2017/12/27 16:36:57 pump.go:378: ESC[0;33m[warning] [Get pull binlogs stream ip-10-10-8-6:8250] rpc error: code = Unavailable desc = grpc: the connection is unavailableESC[0m
2017/12/27 16:37:00 pump.go:378: ESC[0;33m[warning] [Get pull binlogs stream ip-10-10-8-6:8250] rpc error: code = Unavailable desc = grpc: the connection is unavailableESC[0m
2017/12/27 16:37:03 pump.go:378: ESC[0;33m[warning] [Get pull binlogs stream ip-10-10-8-6:8250] rpc error: code = Unavailable desc = grpc: the connection is unavailableESC[0m
2017/12/27 16:37:05 collector.go:166: ESC[0;37m[info] node tidb-online-tidb-2:8250 get save point {75 0}ESC[0m
2017/12/27 16:37:05 pump.go:378: ESC[0;33m[warning] [Get pull binlogs stream tidb-online-tidb-2:8250] rpc error: code = Unavailable desc = grpc: the connection is unavailableESC[0m
2017/12/27 16:37:06 pump.go:378: ESC[0;33m[warning] [Get pull binlogs stream ip-10-10-8-6:8250] rpc error: code = Unavailable desc = grpc: the connection is unavailableESC[0m
2017/12/27 16:37:08 pump.go:284: ESC[0;31m[error] FATAL ERROR: commitTs(396914383516074000) of binlog exceeds the lower boundary of window 396981384481603586, may miss processing, ITEM(&{0xc50ebf8980 {75 0} tidb-online-tidb-2:8250 <nil>})ESC[0m
2017/12/27 16:37:08 pump.go:284: ESC[0;31m[error] FATAL ERROR: commitTs(396914383542288387) of binlog exceeds the lower boundary of window 396981384481603586, may miss processing, ITEM(&{0xc50201cf80 {75 1105} tidb-online-tidb-2:8250 <nil>})ESC[0m
2017/12/27 16:37:08 pump.go:284: ESC[0;31m[error] FATAL ERROR: commitTs(396914383542288392) of binlog exceeds the lower boundary of window 396981384481603586, may miss processing, ITEM(&{0xc50201d580 {75 1865} tidb-online-tidb-2:8250 <nil>})ESC[0m
2017/12/27 16:37:08 pump.go:284: ESC[0;31m[error] FATAL ERROR: commitTs(396914383555395593) of binlog exceeds the lower boundary of window 396981384481603586, may miss processing, ITEM(&{0xc4f4b98580 {75 2599} tidb-online-tidb-2:8250 <nil>})ESC[0m
2017/12/27 17:28:35 pump.go:386: ESC[0;33m[warning] [stream] node ip-10-10-8-7:8250, pos {Suffix:80 Offset:210137628}, error rpc error: code = Canceled desc = context canceledESC[0m
2017/12/27 17:28:38 collector.go:184: ESC[0;37m[info] node(ip-10-10-8-7:8250) of cluster(6493070954582498647)  has been removed and release the connection to itESC[0m
2017/12/27 17:28:58 pump.go:386: ESC[0;33m[warning] [stream] node ip-10-10-8-6:8250, pos {Suffix:77 Offset:26341996}, error rpc error: code = Canceled desc = context canceledESC[0m
2017/12/27 17:29:01 collector.go:184: ESC[0;37m[info] node(ip-10-10-8-6:8250) of cluster(6493070954582498647)  has been removed and release the connection to itESC[0m
2017/12/27 17:58:47 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-1:8250, pos {Suffix:81 Offset:177552746}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/27 18:34:29 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-3:8250, pos {Suffix:80 Offset:273173616}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/27 19:16:06 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-1:8250, pos {Suffix:81 Offset:212949758}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/27 19:28:17 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-4:8250, pos {Suffix:80 Offset:526663116}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/29 11:58:13 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-4:8250, pos {Suffix:81 Offset:480434923}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/29 12:38:00 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-1:8250, pos {Suffix:82 Offset:184323012}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/29 14:25:16 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-4:8250, pos {Suffix:82 Offset:3389669}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/29 18:50:41 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-1:8250, pos {Suffix:82 Offset:365276552}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/12 20:38:38 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-1:8250, pos {Suffix:70 Offset:517918272}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/12 20:59:17 pump.go:386: ESC[0;33m[warning] [stream] node tidb-online-tidb-3:8250, pos {Suffix:69 Offset:505588696}, error rpc error: code = Unknown desc = unexpected EOFESC[0m
2017/12/12 23:52:17 util.go:161: ESC[0;31m[error] close db failed - driver: bad connectionESC[0m
```

![helper-tools](/global/helpertools.jpg "ap.tidb.cc helper tools")
