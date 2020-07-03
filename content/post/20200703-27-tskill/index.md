---
title: 九死一生 - 运维小技巧
date: 2020-07-02T00:00:00+08:00
lastmod: 2020-07-03T00:00:00+08:00
categories:
  - troubleshooting
tags:
  - AirPlan
---
## 0x00 陷入江局

回顾下前两篇文档 [《纸上谈兵 - TiDB 性能调优》](/post/20200311-11-performance-map/)& [《沙盘点将 - TiDB 监控知识初始化》](/post/20200416-16-init-monitoring-map/)；处理问题的时候从总是一脸懵逼到偶尔无证懵逼；然后尝试给某个组件加个「无证之罪」先绕过这个棘手的问题。

![陷入江局](./jiangju.png)

> 程序在干什么？观察 tidb 监控和 profile 获取 SQL 运行在 TiDB Cluster 的情况
> 系统在干什么？使用 linux 命令和 node export 查看运行负载
> 程序和系统一起在干什么？比如 disk util 高的原因？CPU / MEM 占用高的原因？
>  
> 重点回顾；《一条 SQL 的使命》

- TiDB
  - QPS / duration / statement / slow query
  - parser / compile / PD tso query
  - kv count / kv error / kv duration
- TiKV
  - GRPC「TiKV 进出口港」 / thread cpu / cluster / error
  - coprocessor / schedule prewrite & commit
    - coprocessor 与 store readpool 是两种查询方式，store readpool 主要应用**不回表主键点查**
  - task / Raft / RocksDB
- overview
  - CPU / Memory / Disk performance / OPEN FD / Network IO
- SQL 在 TiDB cluster 内运行的过程均有打点记录，资源/耗时 可在 Grafana 上分析后查证
- SQL 进入 TiDB 之前、TiKV 与磁盘的交互；该怎么和 SQL 做一一对应呢？

## 0x01 Slow Query

slow query 文件支持 [pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html) 工具分析  
[PingCAP 官网-slow query 慢日志](https://docs.pingcap.com/zh/tidb/dev/identify-slow-queries)；TiDB 会将执行时间超过 slow-threshold（默认值为 300 毫秒）的语句输出到 slow-query-file（默认值："tidb-slow.log"）日志文件中，用于帮助用户定位慢查询语句，分析和解决 SQL 执行的性能问题

```sql
# Time: 2020-07-02T16:45:48.948834124+08:00  // 执行 query 时间
# Txn_start_ts: 417773531665268886          // 事物 tso  ，类似  mysql gtid
# User: dms@172.16.4.51                     // 执行用户与客户端 IP「前置为 Proxy 时，此处显示 Proxy IP」
# Conn_ID: 327382                           // 链接 ID
# Query_time: 32.452025271                  // query 从 begin 至 commit 总时长，begin 开启后发送 select 语句后，空置一段时间后提交也会被计入
# Parse_time: 0.056166789                   // TiDB-server AST 解析 text to sql 耗时
# Compile_time: 0.000314969                 // TiDB-server Plan 编译耗时
# Process_time: 0.001 Wait_time: 0.001 Request_count: 2 Prewrite_time: 0.003820812 Commit_time: 0.003007832 Get_commit_ts_time: 0.000107308 Write_keys: 1 Write_size: 539 Prewrite_region: 1
# DB: dms                                     // 数据库信息
# Index_names: [t_data_0210:index_id]        // index 使用信息
# Is_internal: false
# Digest: 26becfd8135c8f31011bdb317406018a8de4473c26af18180ef97a91c224a77e
# Stats: t_data_0210:pseudo  // # Stats: t_data_0210:417773784057512074
# Num_cop_tasks: 2
# Cop_proc_avg: 0.0005 Cop_proc_p90: 0.001 Cop_proc_max: 0.001 Cop_proc_addr: 172.16.4.51:20160  // store readpool 不显示该部分信息
# Cop_wait_avg: 0.0005 Cop_wait_p90: 0.001 Cop_wait_max: 0.001 Cop_wait_addr: 172.16.4.51:20160
# Mem_max: 27644
# Prepared: false
# Has_more_results: false
# Succ: true
# Plan: tidb_decode_plan('2gHwaTAJMzFfNgkwCTEJCjEJMTNfNAkxCTEJdGFibGU6VF9EQVRBXzAyMTkxMDE3MDc0NDk4NjAsIGluZGV4OmRldmljZV9pZCwgcmFuZ2U6WyIyMDAxMTgwMDc2NDE4MSIsIjIwMDExODAwNzYFEYBdLCBrZWVwIG9yZGVyOmZhbHNlLCBzdGF0czpwc2V1ZG8BhQgwXzWOhQB6SQA=')
UPDATE t_data_0210 SET `pos`='114' WHERE (`id`='800764181');
```

### exPlan

```sql
(root@127.0.0.1) [(none)]>select tidb_decode_plan('2gHwaTAJMzFfNgkwCTEJCjEJMTNfNAkxCTEJdGFibGU6VF9EQVRBXzAyMTkxMDE3MDc0NDk4NjAsIGluZGV4OmRldmljZV9pZCwgcmFuZ2U6WyIyMDAxMTgwMDc2NDE4MSIsIjIwMDExODAwNzYFEYBdLCBrZWVwIG9yZGVyOmZhbHNlLCBzdGF0czpwc2V1ZG8BhQgwXzWOhQB6SQA=');
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| tidb_decode_plan('2gHwaTAJMzFfNgkwCTEJCjEJMTNfNAkxCTEJdGFibGU6VF9EQVRBXzAyMTkxMDE3MDc0NDk4NjAsIGluZGV4OmRldmljZV9pZCwgcmFuZ2U6WyIyMDAxMTgwMDc2NDE4MSIsIjIwMDExODAwNzYFEYBdLCBrZWVwIG9yZGVyOmZhbHNlLCBzdGF0czpwc2V1ZG8BhQgwXzWOhQB6SQA=')                           |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 	TableReader_6	root	1	
	├─IndexScan_4	cop 	1	table:t_data_0210, index:id, range:["800764181","800764181"], keep order:false, stats:pseudo
	└─TableScan_5	cop 	1	table:t_data_0210, keep order:false, stats:pseudo         |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

### Analyze

[PingCAP 官网-Analyze](https://docs.pingcap.com/zh/tidb/v4.0/sql-statement-explain-analyze)；EXPLAIN ANALYZE 语句的工作方式类似于 EXPLAIN，主要区别在于前者实际上会执行语句。这样可以将查询计划中的估计值与执行时所遇到的实际值进行比较。如果估计值与实际值显著不同，那么应考虑在受影响的表上运行 ANALYZE TABLE。

> explain analyze 是将 SQL 语句在数据库中执行了一遍，然后通过打点收集到的数据；如果是 insert / update /delete 会被实际运行并且操作数据

```sql
explain analyze select id from T_DATA_0210  WHERE (`id`='800764181');
+---------------------+-------+------+-------------------------------------------------------------------------------------------------------------+----------------------------------+---------+
| id                  | count | task | operator info                                                                                               | execution info                   | memory  |
+---------------------+-------+------+-------------------------------------------------------------------------------------------------------------+----------------------------------+---------+
| Projection_4        | 2.00  | root | paas_dms.t_data_0210.id                                                                         | time:1.195879ms, loops:2, rows:1 | N/A     |
| └─IndexLookUp_7     | 2.00  | root |                                                                                                             | time:1.189899ms, loops:2, rows:1 | 1.25 KB |
|   ├─IndexScan_5     | 2.00  | cop  | table:t_data_0210, index:id, range:["800764181","800764181"], keep order:false | time:0s, loops:1, rows:1         | N/A     |
|   └─TableScan_6     | 2.00  | cop  | table:t_data_0210, keep order:false                                                             | time:0s, loops:1, rows:1         | N/A     |
+---------------------+-------+------+-------------------------------------------------------------------------------------------------------------+----------------------------------+---------+
4 rows in set (0.00 sec)
```

### Trace

[PingCAP 官网-Trace](https://docs.pingcap.com/zh/tidb/v4.0/sql-statement-trace#trace)；TRACE 语句用于提供查询执行的详细信息，可通过 TiDB 服务器状态端口所公开的图形界面进行查看。

```sql
trace format = 'row' select id from T_DATA_0210  WHERE (`id`='800764181');
+---------------------------+-----------------+------------+
| operation                 | startTS         | duration   |
+---------------------------+-----------------+------------+
| session.getTxnFuture      | 17:10:42.721708 | 10.991µs   |
|   ├─session.Execute       | 17:10:42.721705 | 978.341µs  |
|   ├─session.ParseSQL      | 17:10:42.722064 | 28.325µs   |
|   ├─executor.Compile      | 17:10:42.722106 | 191.556µs  |
|   ├─session.runStmt       | 17:10:42.722318 | 314.567µs  |
|   ├─session.CommitTxn     | 17:10:42.722616 | 2.942µs    |
|   ├─recordSet.Next        | 17:10:42.722694 | 812.092µs  |
|   ├─projection.Next       | 17:10:42.722696 | 801.486µs  |
|   ├─tableReader.Next      | 17:10:42.723073 | 394.131µs  |
|   ├─tableReader.Next      | 17:10:42.723484 | 947ns      |
|   ├─recordSet.Next        | 17:10:42.723516 | 12.577µs   |
|   └─projection.Next       | 17:10:42.723518 | 1.747µs    |
+---------------------------+-----------------+------------+
12 rows in set (0.01 sec)
```

## 0x02 SSR

- Cluster info

[PingCAP 官网-Cluster info](https://docs.pingcap.com/zh/tidb/v4.0/system-table-cluster-info)；集群拓扑表 CLUSTER_INFO 提供集群当前的拓扑信息，以及各个节点的版本信息、版本对应的 Git Hash、各节点的启动时间、各实例的运行时间。

- TiDB-dashboard

[TiDB Dashboard](https://docs.pingcap.com/zh/tidb/v4.0/dashboard-intro#tidb-dashboard-%E4%BB%8B%E7%BB%8D)是 TiDB 自 4.0 版本起提供的图形化界面，可用于监控及诊断 TiDB 集群。TiDB Dashboard 内置于 TiDB 的 PD 组件中，无需独立部署。

## 0x03 Profile
  
> Profile 用于分析函数占用内存、CPU 资源信息，TiDB 4.0 版本可以在 tidb-dashboard 上一键获取；以下为分析 tidb-dashboard 后挖掘到的 HTTP API 信息

- 获取 PD-server Profile
  - 内容来自 [PD 代码](https://github.com/pingcap/pd/blob/master/server/api/router.go)

```json
curl http://{PD IP}:{PD 2379}/pd/api/v1/debug/pprof/profile?seconds={int} > CPU-profile
// 上述命令获取 CPU profile 信息

// PD 所有 Profile api 请求，每条 api 输出后二进制文件，使用 go tool pprof 解析
apiRouter.HandleFunc("/debug/pprof/profile", pprof.Profile)
apiRouter.Handle("/debug/pprof/heap", pprof.Handler("heap"))
apiRouter.Handle("/debug/pprof/mutex", pprof.Handler("mutex"))
apiRouter.Handle("/debug/pprof/allocs", pprof.Handler("allocs"))
apiRouter.Handle("/debug/pprof/block", pprof.Handler("block"))
apiRouter.Handle("/debug/pprof/goroutine", pprof.Handler("goroutine"))
```

- 获取 TiDB-server Profile
  - 该接口可以获取 tidb-server profile 压缩包，内涵 cpu、内存等信息
  - 内容来自 [TiDB http api 文档](https://github.com/pingcap/tidb/blob/master/docs/tidb_http_api.md) Download TiDB debug info

```bash
curl http://{TiDB IP}:{TiDB status_port}/debug/pprof/profile?seconds={int} > debug.zip

// zip file will include:
// Go heap pprof(after GC)
// Go cpu pprof(10s)
// Go mutex pprof
// Full goroutine
// TiDB config and version
// Param:
// seconds: profile time(s), default is 10s.
```

- 获取 TiKV-server Profile
  - 该接口获取为 svg 图片内容；内容来自 [TiKV issue #4600](https://github.com/tikv/tikv/pull/4600)
  - 2020 年以后发布的版本支持该功能

```bash
curl http://{TiKV IP}:{TiKV status_port}/debug/pprof/profile?seconds={int} > tikv.svg
```

- 查看 Profile

```bash
yum install go
// 安装 go 程序环境

go tool pprof -http 172.16.4.51:890 <profile-name>
// 使用 go tool pprof 工具解析 profile 数据，并开启一个网页以便在浏览器上查看

浏览器打开该网页访问并分析信息 http://172.16.4.51:890/
```

## 0x04 Linux

- vmstat / top / htop
- dstat / iotop / iostat
- lsof
- pstack // 该命令会阻断程序正常运行，禁止在 TiKV 使用；会导致 tikv 状态变为 Disconnected，从而丢失所有 region leader
- strace
- gdb

### Sar

SAR（System Activity Reporter），可以从 14 个大方面对系统的活动进行报告，包括文件的读写情况、系统调用的使用情况、串口、CPU 效率、内存使用状况、进程活动及 IPC 有关的活动等，使用也是较为复杂。

- 追溯过去的统计数据（默认）
- 周期性的查看当前数据

- 怀疑 CPU 存在瓶颈，可用 sar -u 和 sar -q 等来查看
- 怀疑内存存在瓶颈，可用 sar -B、sar -r 和 sar -W 等来查看
- 怀疑 I/O 存在瓶颈，可用 sar -b、sar -u 和 sar -d 等来查看

```bash
-A 汇总所有的报告
-a 报告文件读写使用情况
-B 报告附加的缓存的使用情况
-b 报告缓存的使用情况
-c 报告系统调用的使用情况
-d 报告磁盘的使用情况
-g 报告串口的使用情况
-h 报告关于 buffer 使用的统计数据
-m 报告 IPC 消息队列和信号量的使用情况
-n 报告命名 cache 的使用情况
-p 报告调页活动的使用情况
-q 报告运行队列和交换队列的平均长度
-R 报告进程的活动情况
-r 报告没有使用的内存页面和硬盘块
-u 报告 CPU 的利用率
-v 报告进程、i 节点、文件和锁表状态
-w 报告系统交换活动状况
-y 报告 TTY 设备活动状况
```

## 0x05 不友好的

- useCursorFetch=True&defaultFetchSize>0
- update batchexecutor
