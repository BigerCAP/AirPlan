---
title: TiDB - CDC RC1 测试
date: 2020-06-01T00:00:00+08:00
lastmod: 2020-06-01T00:00:00+08:00
categories:
  - TiDB
tags:
  - AirPlan
  - CDC
---
## 0x00 CDC

> 嗯，同场竞技 Binlog 与 CDC 组件  
> 本页内容仅针对 v4.0.0-rc.1-25 版本，不对后续版本做任何预测性的评论 //  最终解释权归制图者  

- 参考各种文档
  - 官网文档 [安装部署](https://pingcap.com/docs-cn/stable/ticdc/ticdc-overview/)
  - Github Repo [TiCDC](https://github.com/pingcap/ticdc)
    - Improve usability in cdc client [Issue 542](https://github.com/pingcap/ticdc/issues/542)
  - 官网文档 TiKV [GC 机制介绍](https://pingcap.com/docs-cn/stable/garbage-collection-overview/)
  - 官网文档 [TiDB Binlog 简介](https://pingcap.com/docs-cn/stable/tidb-binlog/tidb-binlog-overview/)

![CDC 测试脑图](./CDC.png)

- 测试状态 lastdate 20200707
  - [ ] 主从跨机房部署
  - [ ] 事物一致性还原（过程一致性、最终一致性）
  - [ ] 主备灾难时 CDC 状态
  - [x] CDC 自定义数据源输出；支持 canal 协议「写入到 kafka」
  - [x] db/table 过滤；支持完整的 [Wildcards](https://github.com/pingcap/tidb-tools/blob/master/pkg/table-filter/README.md#wildcards)
  - [x] 支持环形复制

## 0x01 GC safepoint

> 通过 `max(tikv_gcworker_autogc_safe_point) by (instance) / (2^18)` 可以在 Grafana 或者 Prometheus 查看 TiKV 最后一次的 GC 时间。  
> 或者在数据库中通过 `select * from mysql.tidb` 查询  

![gc-safepoint](./gc-safepoint.png)

当 CDC 运行状态中，**checkpoint ts > gc safe point ts** 正常现象，默认 “启动 CDC server 时可以通过 gc-ttl 指定 GC safepoint 的 TTL，这个值的含义是当 TiCDC 服务全部挂掉后，由 TiCDC 在 PD 所设置的 GC safepoint 保存的最长时间，该值默认为 86400 秒。（[来自 PingCAP 官网](https://pingcap.com/docs-cn/stable/ticdc/troubleshoot-ticdc/#gc-ttl-%E5%92%8C%E6%96%87%E4%BB%B6%E6%8E%92%E5%BA%8F)）”  
该场景可能会导致 **TiKV_GC_can_not_work** 告警 `sum(increase(tikv_gcworker_gc_tasks_vec{task="gc"}[1d])) < 1` ; 该告警提示最近 1 天 TiKV GC 没有工作，可能会造成过多的 MVCC 版本从而影响读性能  

![GC](./GC.png)

> 通过 etcd-ctl 获取 ctcd 中的信息，etcd-ctl 通过 [Github etcd](https://github.com/etcd-io/etcd/releases) 下载  
> 执行命令：`./etcdctl --endpoints=http://10.10.10.10:14379 get /tidb/cdc --prefix`

```json
/tidb/store/gcworker/saved_safe_point
416997967863480321
```

## 0x02 CASE

- 创建一个同步任务
  - tpcc.history 表没有 PK 或者 UK，CDC 无法同步类似表

  ```json
  ./cdc cli --pd http://10.10.10.10:14379 changefeed create --sink-uri="mysql://root:root@127.0.0.1:44/?worker-count=1&max-txn-row=50"
  [WARN] some tables are not eligible to replicate, []entry.TableName{entry.TableName{Schema:"tpcc", Table:"history"}}
  Could you agree to ignore those tables, and continue to replicate [Y/N]
  y
  Create changefeed successfully!
  ID: ac6d034a-c2b4-41bf-801a-798bb1390734
  Info: {"sink-uri":"mysql://root:root@127.0.0.1:44/?worker-count=1\u0026max-txn-row=50","opts":{},"create-time":"2020-05-27T16:14:08.529050855+08:00","start-ts":416957660797337610,"target-ts":0,"admin-job-type":0,"sort-engine":"memory","sort-dir":".","config":{"ddl-white-list":null,"filter-case-sensitive":false,"filter-rules":null,"ignore-txn-commit-ts":null,"sink-dispatch-rules":null,"mounter-worker-num":0}}
  ```

- 各种查看
  - capture / 间接等于一个 CDC server
  - change feed / 等于一个同步任务

  ```json
  $ ./cdc cli --pd http://10.10.10.10:14379 capture list
  [
    {
      "id": "7a8e4800-56ae-46a5-a1a6-3eb57b6ce935",
      "is-owner": true,
      "address": "10.10.10.10:14800"
    }
  ]

  $ ./cdc cli --pd http://10.10.10.10:14379 changefeed list
  [
    {
      "id": "ac6d034a-c2b4-41bf-801a-798bb1390734"
    }
  ]
  ```

- 查看上下游同步延迟

  ```json
  ./cdc cli --pd http://10.10.10.10:14379 changefeed statistics  --changefeed-id="ac6d034a-c2b4-41bf-801a-798bb1390734"
  {
    "ops": 0,
    "count": 0,
    "sink_gap": "1389300ms",
    "replication_gap": "1391250ms"
  }
  {
    "ops": 0,
    "count": 0,
    "sink_gap": "1399300ms",
    "replication_gap": "1401250ms"
  }
  {
    "ops": 0,
    "count": 0,
    "sink_gap": "1409351ms",
    "replication_gap": "1411250ms"
  }
  ```

- delete meta 操作
  - cdc stop 时不会删除 change feed 信息，会删除 capture 信息
  - **操作不可逆，谨慎执行**，主要是 checkpoint ts 信息

  ```json
   ./cdc cli --pd http://10.10.10.10:14379 changefeed list
  [
    {
      "id": "f96916c7-1f55-41b5-9d16-406e51aab6d4"
    },
    {
      "id": "3e943df4-6950-4568-84c8-d81fd83de2d8"
    },
    {
      "id": "7de99356-8389-4644-b71b-dc7b6ba7389e"
    }
  ]
   ./cdc cli --pd http://10.10.10.10:14379 meta delete
  already truncate all meta in etcd!

   ./cdc cli --pd http://10.10.10.10:14379 changefeed list
  []
  ```

  ```json
  /tidb/cdc/changefeed/info/3e943df4-6950-4568-84c8-d81fd83de2d8
  {"sink-uri":"mysql://root:root@127.0.0.1:44/?worker-count=16\u0026max-txn-row=50","opts":{},"create-time":"2020-05-28T17:36:26.62800987+08:00","start-ts":416981604520427521,"target-ts":0,"admin-job-type":2,"sort-engine":"memory","sort-dir":".","config":{"ddl-white-list":null,"filter-case-sensitive":false,"filter-rules":{"do-tables":null,"do-dbs":["br"],"ignore-tables":[{"db-name":"br","tbl-name":"history"}],"ignore-dbs":["tpcc","tpch","mysql","test"]},"ignore-txn-commit-ts":[],"sink-dispatch-rules":null,"mounter-worker-num":0}}
  /tidb/cdc/changefeed/info/7de99356-8389-4644-b71b-dc7b6ba7389e
  {"sink-uri":"mysql://root:root@127.0.0.1:44/?worker-count=16\u0026max-txn-row=50","opts":{},"create-time":"2020-05-29T11:24:24.682842675+08:00","start-ts":416997967863480321,"target-ts":0,"admin-job-type":0,"sort-engine":"memory","sort-dir":".","config":{"ddl-white-list":null,"filter-case-sensitive":false,"filter-rules":{"do-tables":null,"do-dbs":["br"],"ignore-tables":[{"db-name":"br","tbl-name":"history"}],"ignore-dbs":["tpcc","tpch","mysql","test"]},"ignore-txn-commit-ts":[],"sink-dispatch-rules":null,"mounter-worker-num":0}}
  /tidb/cdc/changefeed/info/f96916c7-1f55-41b5-9d16-406e51aab6d4
  {"sink-uri":"mysql://root:root@127.0.0.1:44/?worker-count=16\u0026max-txn-row=50","opts":{},"create-time":"2020-06-01T10:52:23.204398589+08:00","start-ts":416997967863480321,"target-ts":0,"admin-job-type":0,"sort-engine":"memory","sort-dir":".","config":{"ddl-white-list":null,"filter-case-sensitive":false,"filter-rules":{"do-tables":null,"do-dbs":["br"],"ignore-tables":[{"db-name":"br","tbl-name":"history"}],"ignore-dbs":["tpcc","tpch","mysql","test"]},"ignore-txn-commit-ts":[],"sink-dispatch-rules":null,"mounter-worker-num":0}}
  /tidb/cdc/job/0240d5b9-a81f-417c-834b-e16329eba09b
  {"resolved-ts":416958926651392001,"checkpoint-ts":416958926389248001,"admin-job-type":3}
  /tidb/cdc/job/3e943df4-6950-4568-84c8-d81fd83de2d8
  {"resolved-ts":416997967863480321,"checkpoint-ts":416997967863480321,"admin-job-type":0}
  /tidb/cdc/job/6281d51f-f62c-46c6-b10e-33013470fbaa
  {"resolved-ts":416974641884037174,"checkpoint-ts":416974641884037174,"admin-job-type":3}
  /tidb/cdc/job/7de99356-8389-4644-b71b-dc7b6ba7389e
  {"resolved-ts":416997967863480321,"checkpoint-ts":416997967863480321,"admin-job-type":0}
  /tidb/cdc/job/ac6d034a-c2b4-41bf-801a-798bb1390734
  {"resolved-ts":416958671598387201,"checkpoint-ts":416957660797337610,"admin-job-type":3}
  /tidb/cdc/job/e01adb9c-70ad-4600-b4de-d20da914bd9a
  {"resolved-ts":416975975436779549,"checkpoint-ts":416975975436779549,"admin-job-type":3}
  /tidb/cdc/job/f96916c7-1f55-41b5-9d16-406e51aab6d4
  {"resolved-ts":416997967863480321,"checkpoint-ts":416997967863480321,"admin-job-type":0}
  /tidb/cdc/task/position/a99683bc-4af4-45d3-aa7b-5bbd398fb6b5/3e943df4-6950-4568-84c8-d81fd83de2d8
  {"checkpoint-ts":417002603526225942,"resolved-ts":416997967863480321,"count":0}
  /tidb/cdc/task/position/a99683bc-4af4-45d3-aa7b-5bbd398fb6b5/7de99356-8389-4644-b71b-dc7b6ba7389e
  {"checkpoint-ts":416997967863480321,"resolved-ts":416997967863480321,"count":0}
  /tidb/cdc/task/position/a99683bc-4af4-45d3-aa7b-5bbd398fb6b5/f96916c7-1f55-41b5-9d16-406e51aab6d4
  {"checkpoint-ts":417002603526225942,"resolved-ts":416997967863480321,"count":0}
  /tidb/cdc/task/status/a99683bc-4af4-45d3-aa7b-5bbd398fb6b5/3e943df4-6950-4568-84c8-d81fd83de2d8
  {"table-infos":[{"id":7441,"start-ts":416997967863480321},{"id":7443,"start-ts":416997967863480321},{"id":7445,"start-ts":416997967863480321},{"id":7449,"start-ts":416997967863480321},{"id":7451,"start-ts":416997967863480321},{"id":7453,"start-ts":416997967863480321},{"id":7455,"start-ts":416997967863480321},{"id":7457,"start-ts":416997967863480321}],"table-p-lock":{"ts":417160718610857984,"creator-id":"a99683bc-4af4-45d3-aa7b-5bbd398fb6b5","checkpoint-ts":0},"table-c-lock":{"ts":417160718610857984,"creator-id":"","checkpoint-ts":417002603526225942},"admin-job-type":0}
  /tidb/cdc/task/status/a99683bc-4af4-45d3-aa7b-5bbd398fb6b5/7de99356-8389-4644-b71b-dc7b6ba7389e
  {"table-infos":[{"id":7441,"start-ts":416997967863480321},{"id":7443,"start-ts":416997967863480321},{"id":7445,"start-ts":416997967863480321},{"id":7449,"start-ts":416997967863480321},{"id":7451,"start-ts":416997967863480321},{"id":7453,"start-ts":416997967863480321},{"id":7455,"start-ts":416997967863480321},{"id":7457,"start-ts":416997967863480321}],"table-p-lock":{"ts":417160718611382272,"creator-id":"a99683bc-4af4-45d3-aa7b-5bbd398fb6b5","checkpoint-ts":0},"table-c-lock":{"ts":417160718611382272,"creator-id":"","checkpoint-ts":416997967863480321},"admin-job-type":0}
  /tidb/cdc/task/status/a99683bc-4af4-45d3-aa7b-5bbd398fb6b5/f96916c7-1f55-41b5-9d16-406e51aab6d4
  {"table-infos":[{"id":7441,"start-ts":416997967863480321},{"id":7443,"start-ts":416997967863480321},{"id":7445,"start-ts":416997967863480321},{"id":7449,"start-ts":416997967863480321},{"id":7451,"start-ts":416997967863480321},{"id":7453,"start-ts":416997967863480321},{"id":7455,"start-ts":416997967863480321},{"id":7457,"start-ts":416997967863480321}],"table-p-lock":{"ts":417160718611644416,"creator-id":"a99683bc-4af4-45d3-aa7b-5bbd398fb6b5","checkpoint-ts":0},"table-c-lock":{"ts":417160718611644416,"creator-id":"","checkpoint-ts":417002603526225942},"admin-job-type":0}
  ```

## 0x03 FAQ

> 测试时问题处理思路

- GC not work 告警
  - 影响：MVCC 版本过多，TikV 数据存储空间变大
  - 原因：changefeed 依赖 TiKV MVCC + Raft log 解析数据，mvcc 数据在未同步到下游时被 GC 删除会导致数据同步丢失；
    - 场景 1：changefeed 同步的快慢与 GC worker 有关系，怎么解决 change feed 同步速度问题；手下要了解 change feed 怎么来的
      1. CDC 链接 PD 获取 TiKV 节点信息
      2. CDC 创建 change feed，change feed 信息存储在 PD etcd 中「cdc 高可用中心节点」
      3. CDC 接受 TiKV 主动推送数据「与 change feed 有关系」放在 **cdc 内存** 里解析、转换 SQL「数据量过大可能造成 OOM，官网有解决方案」「此处非全局有序（单个 change feed 可以放在多个 CDC 上负载方式运行）」
      4. CDC 将 SQL 写入到下游 TiDB「并发由 worker-count&max-txn-row 控制」「不支持同步无主键或无唯一索引的表」
      5. 回顾下：CDC 个数、下游的吞吐能力、上下游网络延迟 决定了同步速度快慢（隐藏条件 CDC 内存处理性能）
    - 场景 2：CDC 创建多个 change feed 任务，其中有任务状态为 pause（[admin job type](https://pingcap.com/docs-cn/stable/ticdc/manage-ticdc/#%E6%9F%A5%E8%AF%A2%E7%89%B9%E5%AE%9A%E5%90%8C%E6%AD%A5%E4%BB%BB%E5%8A%A1) = 1）；
      - pause 状态下防止丢失“数据窗口”信息，会停止 MVCC 数据 GC 直到状态更改为 remove（3）或者等 resume（2）正常消费
      - 使用 `cdc cli changefeed query` 查看

> 其他

- 创建一条 changefeed 过程

```go
[owner.go:822] ["watch capture exited"]
[server.go:159] ["campaign owner successfully"] [capture=fde93803-d051-42f6-92c9-dfa7896a4b71]
[client.go:141] ["[pd] create pd client with endpoints"] [pd-address="[http://172.16.4.51:14379]"]
[base_client.go:242] ["[pd] switch leader"] [new-leader=http://172.16.4.51:14379] [old-leader=]
[base_client.go:92] ["[pd] init cluster id"] [cluster-id=6826949771068502967]
[owner.go:810] ["start to watch captures"]
[owner.go:740] ["monitoring captures"] [key=/tidb/cdc/capture] [rev=1574342]
[owner.go:416] ["handle admin job"] [changefeed=6281d51f-f62c-46c6-b10e-33013470fbaa] [type="resume changefeed"]
[owner.go:159] ["Find new changefeed"] [info="{\"sink-uri\":\"mysql://root:root@127.0.0.1:44/?worker-count=1\\u0026max-txn-row=50\",\"opts\":{},\"create-time\":\"2020-05-27T17:34:43.836728552+08:00\",\"start-ts\":416958928342220801,\"target-ts\":0,\"admin-job-type\":2,\"sort-engine\":\"memory\",\"sort-dir\":\".\",\"config\":{\"ddl-white-list\":null,\"filter-case-sensitive\":false,\"filter-rules\":{\"do-tables\":null,\"do-dbs\":[\"ddcc\"],\"ignore-tables\":null,\"ignore-dbs\":[\"tpcc\",\"tpch\",\"mysql\",\"test\"]},\"ignore-txn-commit-ts\":[],\"sink-dispatch-rules\":null,\"mounter-worker-num\":0}}"] [id=6281d51f-f62c-46c6-b10e-33013470fbaa] ["checkpoint ts"=416974641884037174]
[store.go:68] ["new store"] [path="tikv://172.16.4.51:14379?disableGC=true"]
[client.go:141] ["[pd] create pd client with endpoints"] [pd-address="[172.16.4.51:14379]"]
[base_client.go:242] ["[pd] switch leader"] [new-leader=http://172.16.4.51:14379] [old-leader=]
[base_client.go:92] ["[pd] init cluster id"] [cluster-id=6826949771068502967]
[store.go:74] ["new store with retry success"]
[client_batch.go:628] ["recycle idle connection"] [target=172.16.4.53:15160]
[client_batch.go:628] ["recycle idle connection"] [target=172.16.4.52:15160]
[client_batch.go:628] ["recycle idle connection"] [target=172.16.4.51:14160]
```

- 指定一个错误或者小于当前 savepoint 的方式启动 CDC changefeed
  - `GC life time is shorter than transaction duration, transaction starts at 2020-05-28 10:13:46.23 +0800 CST, GC safe point is 2020-05-28 11:38:33.33 +0800 CST`

```go
[2020/05/28 17:02:26.760 +08:00] [ERROR] [owner.go:539] ["owner exited with error"] [error="create change feed 6281d51f-f62c-46c6-b10e-33013470fbaa: [tikv:9006]GC life time is shorter than transaction duration, transaction starts at 2020-05-28 10:13:46.23 +0800 CST, GC safe point is 2020-05-28 11:38:33.33 +0800 CST"] [errorVerbose="[tikv:9006]GC life time is shorter than transaction duration, transaction starts at 2020-05-28 10:13:46.23 +0800 CST, GC safe point is 2020-05-28 11:38:33.33 +0800 CST\ngithub.com/pingcap/errors.AddStack\n\tgithub.com/pingcap/errors@v0.11.5-0.20190809092503-95897b64e011/errors.go:174\ngithub.com/pingcap/parser/terror.(*Error).GenWithStackByArgs\n\tgithub.com/pingcap/parser@v0.0.0-20200422082501-7329d80eaf2c/terror/terror.go:243\ngithub.com/pingcap/tidb/store/tikv.(*tikvStore).CheckVisibility\n\tgithub.com/pingcap/tidb@v1.1.0-beta.0.20200424111313-5e4a6948977f/store/tikv/kv.go:189\ngithub.com/pingcap/tidb/store/tikv.(*Scanner).getData\n\tgithub.com/pingcap/tidb@v1.1.0-beta.0.20200424111313-5e4a6948977f/store/tikv/scan.go:227\ngithub.com/pingcap/tidb/store/tikv.(*Scanner).Next\n\tgithub.com/pingcap/tidb@v1.1.0-beta.0.20200424111313-5e4a6948977f/store/tikv/scan.go:101\ngithub.com/pingcap/tidb/store/tikv.newScanner\n\tgithub.com/pingcap/tidb@v1.1.0-beta.0.20200424111313-5e4a6948977f/store/tikv/scan.go:59\ngithub.com/pingcap/tidb/store/tikv.(*tikvSnapshot).Iter\n\tgithub.com/pingcap/tidb@v1.1.0-beta.0.20200424111313-5e4a6948977f/store/tikv/snapshot.go:390\ngithub.com/pingcap/tidb/structure.(*TxStructure).iterateHash\n\tgithub.com/pingcap/tidb@v1.1.0-beta.0.20200424111313-5e4a6948977f/structure/hash.go:259\ngithub.com/pingcap/tidb/structure.(*TxStructure).HGetAll\n\tgithub.com/pingcap/tidb@v1.1.0-beta.0.20200424111313-5e4a6948977f/structure/hash.go:208\ngithub.com/pingcap/tidb/meta.(*Meta).ListDatabases\n\tgithub.com/pingcap/tidb@v1.1.0-beta.0.20200424111313-5e4a6948977f/meta/meta.go:494\ngithub.com/pingcap/ticdc/cdc/entry.newSchemaSnapshotFromMeta\n\tgithub.com/pingcap/ticdc@/cdc/entry/schema_storage.go:81\ngithub.com/pingcap/ticdc/cdc/entry.NewSingleSchemaSnapshotFromMeta\n\tgithub.com/pingcap/ticdc@/cdc/entry/schema_storage.go:63\ngithub.com/pingcap/ticdc/cdc.(*Owner).newChangeFeed\n\tgithub.com/pingcap/ticdc@/cdc/owner.go:171\ngithub.com/pingcap/ticdc/cdc.(*Owner).loadChangeFeeds\n\tgithub.com/pingcap/ticdc@/cdc/owner.go:303\ngithub.com/pingcap/ticdc/cdc.(*Owner).run\n\tgithub.com/pingcap/ticdc@/cdc/owner.go:584\ngithub.com/pingcap/ticdc/cdc.(*Owner).Run\n\tgithub.com/pingcap/ticdc@/cdc/owner.go:536\ngithub.com/pingcap/ticdc/cdc.(*Server).campaignOwnerLoop\n\tgithub.com/pingcap/ticdc@/cdc/server.go:167\ngithub.com/pingcap/ticdc/cdc.(*Server).run.func1\n\tgithub.com/pingcap/ticdc@/cdc/server.go:199\ngolang.org/x/sync/errgroup.(*Group).Go.func1\n\tgolang.org/x/sync@v0.0.0-20190911185100-cd5d95a43a6e/errgroup/errgroup.go:57\nruntime.goexit\n\truntime/asm_amd64.s:1357\ncreate change feed 6281d51f-f62c-46c6-b10e-33013470fbaa"]
```

- 下游未创建目标数据库
  - `Error 1146: Table 'tpcc.district' doesn't exist`

```go
[INFO] [client.go:642] ["start new request"] [request="{\"header\":{\"cluster_id\":6826949771068502967},\"region_id\":56301,\"region_epoch\":{\"conf_ver\":14,\"version\":203},\" checkpoint_ts\":416957675215257608,\"start_key\":\"dIAAAAAAAAD/PV9ygAAAAAj/aLrDAAAAAAD6\",\"end_key\":\"dIAAAAAAAAD/PV9ygAAAAAj/bdPyAAAAAAD6\",\"request_id\":988}"] [addr=172.16.4.52:15160]
[WARN] [mysql.go:445] ["execute DMLs with error, retry later"] [error="Error 1146: Table 'tpcc.district' doesn't exist"] [errorVerbose="Error 1146: Table 'tpcc.district' doesn't exist\ngithub.com/pingcap/errors.AddStack\n\tgithub.com/pingcap/errors@v0.11.5-0.20190809092503-95897b64e011/errors.go:174\ngithub.com/pingcap/errors.Trace\n\tgithub.com/pingcap/errors@v0.11.5-0.20190809092503 -95897b64e011/juju_adaptor.go:15\ngithub.com/pingcap/ticdc/cdc/sink.(*mysqlSink).execDMLWithMaxRetries.func2.2\n\tgithub.com/pingcap/ticdc@/cdc/sink/mysql.go:461\ngithub.com/pingcap/ticdc/cdc/sink.(*Statistics) .RecordBatchExecution\n\tgithub.com/pingcap/ticdc@/cdc/sink/statistics.go:47\ngithub.com/pingcap/ticdc/cdc/sink.(*mysqlSink).execDMLWithMaxRetries.func2\n\tgithub.com/pingcap/ticdc@/cdc/sink/mysql.go:453\ngithu b.com/pingcap/ticdc/pkg/retry.Run.func1\n\tgithub.com/pingcap/ticdc@/pkg/retry/retry.go:31\ngithub.com/cenkalti/backoff.RetryNotify\n\tgithub.com/cenkalti/backoff@v2.2.1+incompatible/retry.go:37\ngithub.com/cen kalti/backoff.Retry\n\tgithub.com/cenkalti/backoff@v2.2.1+incompatible/retry.go:24\ngithub.com/pingcap/ticdc/pkg/retry.Run\n\tgithub.com/pingcap/ticdc@/pkg/retry/retry.go:30\ngithub.com/pingcap/ticdc/cdc/sink.( *mysqlSink).execDMLWithMaxRetries\n\tgithub.com/pingcap/ticdc@/cdc/sink/mysql.go:448\ngithub.com/pingcap/ticdc/cdc/sink.(*mysqlSink).execDMLs\n\tgithub.com/pingcap/ticdc@/cdc/sink/mysql.go:506\ngithub.com/pingc ap/ticdc/cdc/sink.(*mysqlSinkWorker).run\n\tgithub.com/pingcap/ticdc@/cdc/sink/mysql.go:399\nruntime.goexit\n\truntime/asm_amd64.s:1357"]
```

- 附图两张

![cdc-memroy ap.tidb.cc](./cdc-memory.png "cdc ap.tidb.cc")  
![cdc-uptime ap.tidb.cc](./cdc-uptime.png "ap.tidb.cc cdc")  
