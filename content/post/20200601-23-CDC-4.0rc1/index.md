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

- 未测试
  - [ ] 主从跨机房部署
  - [ ] 事物一致性还原（过程一致性、最终一致性）
  - [ ] 主备灾难时 CDC 状态
  - [ ] CDC 自定义数据源输出

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

## 0x01 case

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
  - capture / jianjie 等于一个 CDC server
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
