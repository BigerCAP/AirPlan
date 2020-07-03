---
title: TiDB - Backup/Restore 测试
date: 2020-06-17T00:00:00+08:00
lastmod: 2020-06-18T00:00:00+08:00
categories:
  - TiDB
tags:
  - AirPlan
  - BR
---
## 0x00 BR

> Backup & Restore（以下简称 BR）是 TiDB 分布式备份恢复的命令行工具，用于对 TiDB 集群进行数据备份和恢复。相比 mydumper/loader，BR 更适合大数据量的场景。本文档介绍了 BR 的使用限制、工作原理、命令行描述、备份恢复用例以及最佳实践。  
> 以上内容复制于 [PingCAP 官网文档-使用 BR 进行备份与恢复](https://docs.pingcap.com/zh/tidb/dev/backup-and-restore-tool "PingCAP 官网文档-使用 BR 进行备份与恢复")  
> 提前详细阅读 [PingCAP 官网文档-Backup & Restore 常见问题](https://docs.pingcap.com/zh/tidb/dev/backup-and-restore-faq)  
>  
> - 详细阅读官网文档中的**使用限制**后再进行使用  
> - 详细阅读官网文档中的**使用限制**后再进行使用  
> - 详细阅读官网文档中的**使用限制**后再进行使用  

时空穿梭，书接上文：上文 《软件测试 - S3 协议与 minio》是为了搭建一套 S3 协议存储，该存储相比自己搭建一套 NFS 要方便的多。  
BR 支持三种存储（本地存储、NFS、S3 存储），本次主要测试了本地存储备份恢复、S3 存储备份恢复；此处 NFS 备份恢复与 S3 备份恢复逻辑是一样的。  

> 本文中测试为 2020 年 6 月版本，该工具目前处于飞速迭代期，不对后续功能做预测。

## 0x01 思考

![br 架构](./br-arch.png)

- 看图（文章）识字，猜测下运行思路：
  1. BR 向 PD 发送 backup / restore 指令
  2. PD 获取当前 TSO 信息（**backupmeta** 文件中的断点记录）
  3. BR 向 TiDB httpapi 获取 database/table schema 信息（记录在**backupmeta** 文件中，还原时使用）
     - 参见 [tidb_http_api](https://github.com/pingcap/tidb/blob/master/docs/tidb_http_api.md) ：`curl http://{TiDBIP}:10080/schema/{db}/{table}`
  4. TiKV 收到 backup 指令；获取目标 schema 所辖 kv range 的所有 sst 文件（输出到 本地存储 / 网络存储）
      - TiKV 原生自带 S3 / NFS 等接口；无需安装第三方工具外挂到 tikv-server
      - 每个 TiKV 自身发起 backup / restore 动作，并发操作减少总任务时长；单个 TiKV 数据量足够少，总备份时间相应会变短
      - TiKV 数据备份支持流控，通过 BR 参数传递
  5. 存储场景使用区别：
     1. 本地存储：backup 备份至每个 tikv-server 服务器本地，**restore 时每台机器都必须有全量备份数据**；好处是磁盘 IOPS 最大化使用，网络 IO 主机内使用。
     2. 网络存储：backup 备份时传输至网络存储服务器「提前预算足够大的空间」，restore 时从网络空间下载对应数据到每个 tikv-server 节点；磁盘 IOPS 局限单台服务器能力、网络 IO 跨主机使用。

## 0x02 backupmeta

> 默认文件内容为二进制编码不可以编辑，打开时提示 `"backupmeta" may be a binary file.  See it anyway?`  

- 使用 `./br validate decode -s local:///tmp/tmpuserbackup` 将 backupmeta 二进制文件解码为 backupmeta.json 文件（该文件与 backupmeta 同级目录）  
  - backupmeta.json 文件内容为超长的一句话 JSON；key 信息保留，value 内容被 base64 编码
- 使用 `./br validate encode -s` 可以将 backupmeta.json 编码为 backupmeta 二进制文件  
- **以上操作出现失误将会导致数据还原出现问题**
  - 操作上可以实现 backup database A ，restore database B 的场景；
  - 「schema 信息在每个表上都有，需要全部修改」

### SST file

> SST (Sorted Sequence Table)

- 本次 backup 任务所有的 SST 文件信息，下面是其中一部分内容
  - restore 时会按照 backupmeta 文件内容进行恢复
  - 目标数据目录中相比 backupmeta 信息中缺少文件，提示 `The specified key does not exist.` 或者 `permission denied: download sst failed;`
  - 目标数据目录中相比 backupmeta 信息中多余文件，不做操作

```json
{
    "name":"2476_46_23_80992061af3e5194c3f28a5b79d486c5e9db2feda1afb3f84b4ca229ddce9932_write.sst",
    "sha256":"AXCDqiNxpPxx4MmWz/nHwqy1N8kcRxc1HKgcDV3gnkI=",
    "start_key":"dIAAAAAAAAAxX3IAAAAAAAAAAA==",
    "end_key":"dIAAAAAAAAAxX3L//////////wA=",
    "end_version":417071760901406802,
    "crc64xor":12503741000804331466,
    "total_kvs":50,
    "total_bytes":6216,
    "cf":"write",
    "size":6475
}
```

### schema

- 本次 backup 任务所有的 schema 信息「以下仅展示部分信息」

```json
"schemas":[{"db":"eyJpZCI6NDUsImRiX25hbWUiOnsiTyI6InRwY2MtMiIsIkwiOiJ0cGNjLTIifSwiY2hhcnNldCI6InV0ZjhtYjQiLCJjb2xsYXRlIjoidXRmOG1iNF9iaW4iLCJzdGF0ZSI6NX0=",
```

- schema 解码后的信息

```json
{
    "id":45,            // schema id
    "db_name":{
        "O":"tpcc-2",  // 代表 create database 时原始字段信息
        "L":"tpcc-2"   // 程序转换为小写后的信息
    },
    "charset":"utf8mb4",
    "collate":"utf8mb4_bin",
    "state":5
}
```

### checkpoint

- 获取 backupmeta 文件中的 起始时间、结束时间
  - 全量与增量备份注意事项：按照严格准确的全量数据 end-version 来指定为增量备份 start-ts
  - 如果指定非精准 start-ts，出现数据重叠部分有 DDL 动作，可能会造成数据异常

```s
./br validate decode  -s local:///tmp/tmpuserbackup  --field="start-version"
Detial BR log in /tmp/br.log.2020-06-02T09.19.46+0800
0
// 该场景为全量备份，获取 start-version；即该次备份数据起始位置；全量备份默认起始位置为 0

./br validate decode  -s local:///tmp/tmpuserbackup  --field="end-version"
Detial BR log in /tmp/br.log.2020-06-02T09.19.52+0800
417071760901406802
// 该场景为全量备份，获取 end-version；即该次备份数据结束位置

./br validate decode  -s local:///tmp/tmpuserbackup-zl-2  --field="start-version"
Detial BR log in /tmp/br.log.2020-06-02T09.20.07+0800
417071732760248335
// 增量备份：此处 start-version 与全量场景 end-version 理应一一对应「此处不对应是因为 start-version 为手动指定」，正确操作姿势详情见 [PingCAP 官网-增量备份](https://docs.pingcap.com/zh/tidb/dev/backup-and-restore-tool#%E5%A2%9E%E9%87%8F%E5%A4%87%E4%BB%BD)

./br validate decode  -s local:///tmp/tmpuserbackup-zl-2  --field="end-version"
Detial BR log in /tmp/br.log.2020-06-02T09.20.01+0800
417071840829112353
// 增量备份：增量备份结束时间
```

### checksum

- 本地备份场景注意事项；以下是 checksum 检查不完整 bakckup 数据目录的报错
  - 将所有的 backup 数据收集到一台机器，得到一份全量备份数据
  - 然后将全量备份数据复制到每台 tikv-server 机器，此时才可以进行 restore 操作

```s
 ./br validate checksum --pd 172.16.4.51:14379  --storage "local:///data3/tmpuser/full/full"  
Detial BR log in /tmp/br.log.2020-06-02T11.21.18+0800
Error: open /data3/tmpuser/full/full/1644_58401_49_1970ce6c62b57fa0ad6226e45738144e10c519941f21cd1b7d14d5d255997975_default.sst: no such file or directory
```

- 备份数据目录包含子目录，且子目录可能为增量备份

```s
 ./br validate checksum --storage "s3://full/full/zl-2/?endpoint=http://172.16.4.51:99"
Detial BR log in /tmp/br.log.2020-06-02T17.20.55+0800
Error: NoSuchKey: The specified key does not exist.
        status code: 404, request id: 1614AF9CB410A291, host id:
```

- 备份数据检测完整；可以进行数据恢复

```S
./br validate checksum --storage "s3://full/full/zl-2/?endpoint=http://172.16.4.51:99"
Detial BR log in /tmp/br.log.2020-06-02T17.21.51+0800
backup data checksum succeed!

./br validate checksum --pd 172.16.4.51:14379  --storage "local:///data3/tmpuser/full/full"  
Detial BR log in /tmp/br.log.2020-06-02T11.18.32+0800
backup data checksum succeed!
```

- backupmeta 已存在目录中，该目录无法创建备份；

```s
./br backup table  --pd 172.16.4.51:14379 --db tpcc --table order_line --storage "local:///tmp/tmpuserbackup"  --log-file backupfull.log-2
Detial BR log in backupfull.log-2
Error: backup meta exists, may be some backup files in the path already
```

- 该目录中已有 SST 文件不影响创建备份操作「不检测目录是否为空」，如果备份 SST 文件与已有文件重名会停止备份
  - 文件名带有 storeid_regionid_schemaid_key 信息，动态数据重名几率较小

```s
./br backup table  --pd 172.16.4.51:14379 --db tpcc --table order_line --storage "local:///tmp/tmpuserbackup-2"  --log-file backupfull.log-2
Detial BR log in backupfull.log-2
Table backup <..............................................................................................................................................................................................> 0.00%Error: msg:"Io(Custom { kind: AlreadyExists, error: \"[1_112_45_393d3575060c0d616300c1199ef1c015784fdd3d13e950a6251007bbcbaf2c06_write.sst] is already exists in /tmp/tmpuserbackup-2\" })"
```

## 0x03 测试

> BR 使用简单、快捷，填充参数增加文章长度；

```s
br is a TiDB/TiKV cluster backup restore tool.

Usage:
  br [command]

Available Commands:
  backup      backup a TiDB/TiKV cluster
  help        Help about any command
  restore     restore a TiDB/TiKV cluster

Flags:
      --ca string                     CA certificate path for TLS connection
      --cert string                   Certificate path for TLS connection
      --checksum                      Run checksum at end of task (default true)
      --gcs.credentials-file string   (experimental) Set the GCS credentials file path
      --gcs.endpoint string           (experimental) Set the GCS endpoint URL
      --gcs.predefined-acl string     (experimental) Specify the GCS predefined acl for objects
      --gcs.storage-class string      (experimental) Specify the GCS storage class for objects
  -h, --help                          help for br
      --key string                    Private key path for TLS connection
      --log-file string               Set the log file path. If not set, logs will output to temp file (default "/tmp/br.log.2020-07-02T14.25.25+0800")
  -L, --log-level string              Set the log level (default "info")
  -u, --pd strings                    PD address (default [127.0.0.1:2379])
      --ratelimit uint                The rate limit of the task, MB/s per node
      --s3.acl string                 (experimental) Set the S3 canned ACLs, e.g. authenticated-read
      --s3.endpoint string            (experimental) Set the S3 endpoint URL, please specify the http or https scheme explicitly
      --s3.provider string            (experimental) Set the S3 provider, e.g. aws, alibaba, ceph
      --s3.region string              (experimental) Set the S3 region, e.g. us-east-1
      --s3.sse string                 Set S3 server-side encryption, e.g. aws:kms
      --s3.sse-kms-key-id string      KMS CMK key id to use with S3 server-side encryption.Leave empty to use S3 owned key.
      --s3.storage-class string       (experimental) Set the S3 storage class, e.g. STANDARD
  -c, --send-credentials-to-tikv      Whether send credentials to tikv (default true)
      --status-addr string            Set the HTTP listening address for the status report service. Set to empty string to disable
  -s, --storage string                specify the url where backup storage, eg, "s3://bucket/path/prefix"
  -V, --version                       Display version information about BR

Use "br [command] --help" for more information about a command.
```

> 以下内容请移步参观官网详细文档 [PingCAP 官网文档-使用 BR 进行备份与恢复](https://docs.pingcap.com/zh/tidb/dev/backup-and-restore-tool "PingCAP 官网文档-使用 BR 进行备份与恢复")

### local BR

- 备份数据到本地存储「提前准备好全量数据备份的磁盘空间（如果需要本地恢复的话）」

```s
./br backup table  --pd 172.16.4.51:14379 --db tpcc --table history --storage "local:///tmp/tmpuserbackup-3"  --log-file backupfull.log-2
Detial BR log in backupfull.log-2
Table backup <--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
Checksum <------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
{"level":"warn","ts":"2020-06-01T15:59:34.671+0800","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"endpoint://client-3be8bd38-4e9e-4166-a102-afcab137eee9/172.16.4.51:14379","attempt":0,"error":"rpc error: code = Canceled desc = grpc: the client connection is closing"}
{"level":"warn","ts":"2020-06-01T15:59:34.697+0800","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"endpoint://client-3be8bd38-4e9e-4166-a102-afcab137eee9/172.16.4.51:14379","attempt":1,"error":"rpc error: code = Canceled desc = grpc: the client connection is closing"}
{"level":"warn","ts":"2020-06-01T15:59:34.723+0800","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"endpoint://client-3be8bd38-4e9e-4166-a102-afcab137eee9/172.16.4.51:14379","attempt":2,"error":"rpc error: code = Canceled desc = grpc: the client connection is closing"}
[2020/06/01 15:59:37.355 +08:00] [INFO] [collector.go:58] ["Table backup Success summary: total backup ranges: 3, total success: 3, total failed: 0, total take(s): 24.02, total kv: 53717061, total size(MB): 2859.38, avg speed(MB/s): 119.02"] ["backup checksum"=3.148382555s] ["backup fast checksum"=634.588µs] ["backup total regions"=62]
```

- permission denied
  - **首先检查每个 tikv-server 主机上是否有全量数据**
  - 然后检查 br -s 指定的数据目录是否正确，**每个 tikv 节点路径必须一致**
  - 然后检查**目录权限是否被 tikv-server 运行用户可读**

```s
./br restore full --pd 172.16.4.51:14379  --storage "local:///data3/tmpuser/full/full"  --status-addr 172.16.4.51:18880
Detial BR log in /tmp/br.log.2020-06-02T11.30.48+0800
Full restore <---------.................................................................................................................................................................................> 4.74%Error: permission denied: download sst failed; permission denied: download sst failed; permission denied: download sst failed; permission denied: download sst failed; permission denied: download sst failed; permission denied: download sst failed; permission denied: download sst failed; permission denied: download sst failed
```

### S3 BR

- 备份数据到网络存储
  - export AWS_ACCESS_KEY_ID=minioadmin
  - export AWS_SECRET_ACCESS_KEY=minioadmin

```s
 ./br backup db --db br --pd 172.16.4.51:14379  --storage "s3://br-order-line/?endpoint=http://172.16.4.51:99"
Detial BR log in /tmp/br.log.2020-06-02T17.32.47+0800
Database backup <----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
Checksum <-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
[2020/06/02 17:39:37.879 +08:00] [INFO] [collector.go:58] ["Database backup Success summary: total backup ranges: 20, total success: 20, total failed: 0, total take(s): 366.26, total kv: 1390969552, total size(MB): 128968.74, avg speed(MB/s): 352.12"] ["backup checksum"=44.411943312s] ["backup fast checksum"=5.160845ms] ["backup total regions"=2402]
```

- 从网络存储恢复数据
  - export AWS_ACCESS_KEY_ID=minioadmin
  - export AWS_SECRET_ACCESS_KEY=minioadmin

```s
 ./br restore full --pd 172.16.4.51:14379  --storage "s3://full/full/full/?endpoint=http://172.16.4.51:99" --status-addr 172.16.4.51:18880
Detial BR log in /tmp/br.log.2020-06-02T17.13.34+0800
Full restore <-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
Checksum <-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
[2020/06/02 17:15:38.093 +08:00] [INFO] [collector.go:58] ["Full restore Success summary: total restore files: 866, total success: 866, total failed: 0, total take(s): 84.16, total kv: 501863655, total size(MB): 38876.69, avg speed(MB/s): 461.96"] ["split region"=12.75036978s] ["restore checksum"=16.45494056s] ["restore ranges"=822]

```

- 手动提前创建好 S3/backup 信息；backup == S3 目录

```s
 ./br backup db --db br --pd 172.16.4.51:14379  --storage "s3://br-order-line/?endpoint=http://172.16.4.51:99"
Detial BR log in /tmp/br.log.2020-06-02T17.32.11+0800
Error: Bucket br-order-line is not accessible: NotFound: Not Found
        status code: 404, request id: 1614B03A0CD600B5, host id:
```

## FAQ

数据一边大量写入，一边做增量备份；BR 是怎么控制结束时间的呢？「有奖问答」

- 贴一段 BR 运行日志

```s
[2020/06/01 16:15:38.439 +08:00] [INFO] [coprocessor.go:870] ["[TIME_COP_PROCESS] resp_time:303.5223ms txnStartTS:417070892274155527 region_id:148 store_addr:172.16.4.52:15160 kv_process_ms:301 scan_total_write:145255 scan_processed_write:145254 scan_total_data:145254 scan_processed_data:145254 scan_total_lock:1 scan_processed_lock:0"]
[2020/06/01 16:15:54.730 +08:00] [INFO] [schema.go:94] ["table checksum finished"] [table=`tpcc`.`order_line`] [Crc64Xor=18378085887538976218] [TotalKvs=369955866] [TotalBytes=30627398952] [take=16.601811036s]
[2020/06/01 16:15:54.730 +08:00] [INFO] [schema.go:107] ["backup checksum"] [take=16.602038025s]
[2020/06/01 16:15:54.734 +08:00] [INFO] [client.go:895] ["fast checksum calculated"] [db=tpcc] [table=warehouse]
[2020/06/01 16:15:54.734 +08:00] [INFO] [client.go:895] ["fast checksum calculated"] [db=tpcc] [table=district]
[2020/06/01 16:15:54.735 +08:00] [INFO] [client.go:895] ["fast checksum calculated"] [db=tpcc] [table=item]
[2020/06/01 16:15:54.735 +08:00] [INFO] [client.go:895] ["fast checksum calculated"] [db=tpcc] [table=new_order]
[2020/06/01 16:15:54.735 +08:00] [INFO] [client.go:895] ["fast checksum calculated"] [db=tpcc] [table=customer]
[2020/06/01 16:15:54.736 +08:00] [INFO] [client.go:895] ["fast checksum calculated"] [db=tpcc] [table=stock]
[2020/06/01 16:15:54.736 +08:00] [INFO] [client.go:895] ["fast checksum calculated"] [db=tpcc] [table=history]
[2020/06/01 16:15:54.736 +08:00] [INFO] [client.go:895] ["fast checksum calculated"] [db=tpcc] [table=orders]
[2020/06/01 16:15:54.737 +08:00] [INFO] [client.go:895] ["fast checksum calculated"] [db=tpcc] [table=order_line]
[2020/06/01 16:15:54.737 +08:00] [INFO] [client.go:841] ["fast checksum success"] [database=tpcc] [table=warehouse]
[2020/06/01 16:15:54.737 +08:00] [INFO] [client.go:841] ["fast checksum success"] [database=tpcc] [table=district]
[2020/06/01 16:15:54.737 +08:00] [INFO] [client.go:841] ["fast checksum success"] [database=tpcc] [table=item]
[2020/06/01 16:15:54.737 +08:00] [INFO] [client.go:841] ["fast checksum success"] [database=tpcc] [table=new_order]
[2020/06/01 16:15:54.738 +08:00] [INFO] [client.go:841] ["fast checksum success"] [database=tpcc] [table=customer]
[2020/06/01 16:15:54.739 +08:00] [INFO] [client.go:841] ["fast checksum success"] [database=tpcc] [table=stock]
[2020/06/01 16:15:54.739 +08:00] [INFO] [client.go:841] ["fast checksum success"] [database=tpcc] [table=history]
[2020/06/01 16:15:54.740 +08:00] [INFO] [client.go:841] ["fast checksum success"] [database=tpcc] [table=orders]
[2020/06/01 16:15:54.740 +08:00] [INFO] [client.go:841] ["fast checksum success"] [database=tpcc] [table=order_line]
[2020/06/01 16:15:54.741 +08:00] [INFO] [client.go:168] ["save backup meta"] [path=local:///tmp/tmpuserbackup] [jobs=0]
[2020/06/01 16:15:54.743 +08:00] [INFO] [ddl.go:407] ["[ddl] DDL closed"] [ID=85890430-0b99-4322-8908-9a6412da4ad8] ["take time"=788.052µs]
[2020/06/01 16:15:54.743 +08:00] [INFO] [ddl.go:301] ["[ddl] stop DDL"] [ID=85890430-0b99-4322-8908-9a6412da4ad8]
[2020/06/01 16:15:54.744 +08:00] [INFO] [domain.go:607] ["domain closed"] ["take time"=1.996407ms]
[2020/06/01 16:15:54.745 +08:00] [INFO] [collector.go:59] ["Database backup Success summary: total backup ranges: 19, total success: 19, total failed: 0, total take(s): 145.77, total kv: 498155202, total size(MB): 38602.90, avg speed(MB/s): 264.83"] ["backup fast checksum"=6.338075ms] ["backup checksum"=16.602056489s] ["backup total regions"=812]
```
