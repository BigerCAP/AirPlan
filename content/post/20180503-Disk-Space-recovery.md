---
title: TiKV 磁盘空间回收步骤记录
date: 2018-05-03T00:00:00+08:00
lastmod: 2020-01-20T00:00:00+08:00
categories:
  - Troubleshooting
tags:
  - TiKV
  - Disk
---
## 0x00 前景介绍

> 该版本文档内容适用于 TiKV 3.0 之前版本；不包含使用 TiKV Titan engine 架构的内容

- TiKV 节点磁盘占用主要为以下三个目录
  - `{{deploy_dir}}/log`  TiKV 日志目录，目前日志 GC 需要单独脚本操作
  - `{{deploy_dir}}/data` TiKV rocksDB 数据目录，用来存放 KV 数据
  - `{{deploy_dir}}/data/snap` TiKV snap 数据目录，region 迁移时会先生成一个 snapshot 文件，然后通过网络传输到其他节点

### 场景

- 业务：持续写入数据，仅保留最近一月数据；要求快速回收磁盘空间或磁盘空间相对均匀
- 集群：有数十台 TiKV ，每台 TiKV 磁盘大小仅为 500G
- 现象：仅有三十几台 TiKV 磁盘占用超过 60%，其余 TiKV 在空跑状态；个别 TiKV 持续写入达到 90%

## 0x01 - 操作

- 调整磁盘占用比较高得 store 权重，控制磁盘 TiKV 写入量。
  - release-2.0(RC & GA) 之后，通过调整 store 权重，可以降低 leader 与 region 的数量，进而控制磁盘空间的占用率
  - release-1.0 调整 store 权重，在不涉及 null region 时，可以控制磁盘空间占用率 (如果长期持续删除大量输出会造成 null region)
- release-1.0 之前，通过下线方式控制磁盘空间占用
- 本文中未涉及的信息点
  - TiKV `dynamic-level-bytes` 参数（动态管理 LSM 技术，配合 LSM compactions 食用）
  - TiKV 2.0 版本 `use-delete-range` 参数
  - 2.0 版本 `region-merge` 功能（主动合并 table 纬度内 null region 或小于阈值参数的 region）
  - 修改 TiKV LSM 每层压缩参数，如：
    - no:     kNoCompression
    - snappy: kSnappyCompression
    - zlib:   kZlibCompression
    - bzip2:  kBZip2Compression
    - lz4:    kLZ4Compression
    - lz4hc:  kLZ4HCCompression
    - zstd:   kZSTD

### 调整 store 权重

通过 PD http API 快速查看集群内所有 Store 权重信息

```bash
curl http://10.10.10.15:2379/pd/api/v1/stores |  egrep '(id|address|state_name|region_weight|available)' | awk '{if(NR%5!=0)ORS=" "; else ORS="\n"}1' | sed 's/[ ][ ]*/  /g' | sed 's/"//g' | sort -nrk 8
```

- 通过 pd-ctl 工具调整 store 权重
  - 优势：使用 API 调用，操作方便，不涉及 store 上下线等危险操作
  - 缺点：当磁盘空间仅剩 50G 以内，且数据依旧高歌猛进写入时，无法使用；调整权重是使用 `balance-region-scheduler` 与 `balance-leader-scheduler` 调度器，需要一定的时间调度 region ，调度 region 需要额外的空间，写入 snapshot

- 通过 pd-ctl 工具连接 PD Cluster

```bash
./pd-ctl -u http://10.10.10.15:2379
```

- help store ，查看命令使用方法

```bash
» help store
show the store status

Usage:
  pdctl store [delete|label|weight] <store_id>
  store [command]

Available Commands:
  delete      delete the store
  label       set a store's label value
  weight      set a store's leader and region balance weight

Use "help store [command] " for more information about a command.
```

- 调整 store 194 节点的 leader 与 region 权重
  - 权重可调范围 `0.000001 到 1` ，默认值为 1
  - 格式为： `store weight storeID leader权重 region权重`

```bash
» store weight 194 0.1 0.1
```

- 查看调整后结果

```bash
» store 194
{
  "store": {
    "id": 194,
    "address": "10.10.10.13:20160",
    "state": 0,
    "state_name": "Up"
  },
  "status": {
    "capacity": "500 GiB",
    "available": "82 GiB",
    "leader_weight": 0.1,   # leader 权重
    "region_count": 31316,
    "region_weight": 0.1,   # region 权重
    "region_score": 313160,
    "start_ts": "2018-04-22T08:05:13+08:00",
    "last_heartbeat_ts": "2018-05-03T16:27:12.85852309+08:00",
    "uptime": "272h21m59.85852309s"
  }
}
```

### 添加 store 调度

> 添加指定 store `evict scheduler` ，控制该 store 在集群内 region leader 数量为 0 或者接近 0 值，不能控制该 store 是否增加成员副本。

- 通过 pd-ctl 工具调整 store 权重
  - 优势：该节点添加调度器后，该 store 不存在 region leader ，该单节点短暂挂掉或重启不影响集群读写（该方案可与 region weight 同时使用，不过该方案更多是用于重启 tikv ，业务不受影响场景）
  - 缺点：该调度器并不能控制空间使用状态

- 添加调度器，将 store 194 上所有 leader 自动迁移到其他节点。10000 数量 leader 仅需 5 - 10 分钟
  - 该调度器添加后持续生效，需要删除调度器释放 store `evict scheduler` 状态

```bash
curl -X POST -d '{"name":"evict-leader-scheduler","store_id": 194}' "http://10.10.10.15:2379/pd/api/v1/schedulers"
```

- 查看当前所有调度器

```bash
# curl http://10.13.128.205:2379/pd/api/v1/schedulers

[
  "evict-leader-scheduler-194",   # store 194 调度器
  "balance-region-scheduler",     # 默认调度器，region 均衡调度
  "balance-leader-scheduler",     # 默认调度器，leader 均衡调度
  "balance-hot-region-scheduler"  # 默认调度器，热点调度
]
```

- 删除迁移 leader 调度器
  - 删除后，该节点参与集群内 leader 调度

```bash
curl -X DELETE "http://10.10.10.15:2379/pd/api/v1/schedulers/evict-leader-scheduler-194"
```

### 下线 Store

> 该方案可作为压轴方案 (集群内只有三台 TiKV 时，下线一台后，region 无处调度，但剩余两副本可继续维持集群使用)

- 通过 pd-ctl 工具连接 PD Cluster

```bash
./pd-ctl -u http://10.10.10.15:2379
```

- help store ，查看命令使用方法

```bash
» help store
show the store status

Usage:
  pdctl store [delete|label|weight] <store_id>
  store [command]

Available Commands:
  delete      delete the store
  label       set a store's label value
  weight      set a store's leader and region balance weight

Use "help store [command] " for more information about a command.
```

- 通过 pd-ctl 删除 store 194 节点
  - 节点会先进入 offline 状态，当该节点数据迁移完成后 (数据迁移与磁盘读写和其他 store 节点繁忙程度有关)
  - 节点进入 Tomestone 状态，此时可物理删除该节点数据 (`{{deploy_dir/data}}` 存放 rocksDB 数据)

```bash
» store delete 194
```

- 查看调整后结果

```bash
» store 194
{
  "store": {
    "id": 194,
    "address": "10.10.10.13:20160",
    "state": 0,
    "state_name": "Offline"  # store 状态 ( UP 、Down 、Offline 、Tomestone)
  },
  "status": {
    "capacity": "500 GiB",
    "available": "82 GiB",
    "leader_weight": 0.1,   # leader 权重
    "region_count": 31316,
    "region_weight": 0.1,   # region 权重
    "region_score": 313160,
    "start_ts": "2018-04-22T08:05:13+08:00",
    "last_heartbeat_ts": "2018-05-03T16:27:12.85852309+08:00",
    "uptime": "272h21m59.85852309s"
  }
}
```
