---
title: 测试在线修改 TiDB Cluster 组件 IP 地址
date: 2018-03-27T00:00:00+08:00
lastmod: 2020-01-20T00:00:00+08:00
categories:
  - TiDB
tags:
  - Deploy
---
## 0x00 背景

物理网络在实际生产环境会存在 vlan、IP 集群部署可能横跨多台机器甚至横跨机房，在有离线或在线业务情况下应该怎么操作变更 IP 地址信息，配合系统 & 网络整体调整。

> 以下采用 TiDB 1.0 版本测试 // 20200120 略有更新

## 0x01 - PD

> PD 不可以直接通过启动脚本修改地址，因相关信息已注册到 etcd 。需要通过扩容减容方式更换 PD 地址，*扩容需要避免长时间存在偶数副本，执行 transfer PD lerader 期间可能会造成 sql 请求延迟抖动*。

- PD 启动脚本示例：

```bash
exec bin/pd-server \
    --name="pd1" \
    --client-urls="http://0.0.0.0:2379" \  # PD 集群内部通信地址
    --advertise-client-urls="http://172.16.10.103:2379" \  # PD 集群对外提供服务地址，该地址不可为 0.0.0.0
    --peer-urls="http://0.0.0.0:2380" \  # PD 集群内部通信地址
    --advertise-peer-urls="http://172.16.10.103:2380" \  # PD 集群对外提供服务地址，该地址不可为 0.0.0.0
    --data-dir="/data1/deploy/data.pd" \
    --initial-cluster="pd1=http://172.16.10.103:2380" \  # 初始化 PD 集群时使用，扩容时应当换成  --join 参数
    --config=conf/pd.toml \
    --log-file="/data1/deploy/log/pd.log" 2> "/data1/deploy/log/pd_stderr.log"
```

### 扩容 PD Cluster

1. 例如，如果要添加一个 PD 节点 (node103)，IP 地址为 172.16.10.103，可以进行如下操作：

  | Name | Host IP | Services |
  | ---- | ------- | -------- |
  | node1 | 172.16.10.1 | PD1 |
  | node2 | 172.16.10.2 | PD2 |
  | node3 | 172.16.10.3 | PD3, Monitor |
  | **node103** | **172.16.10.103** | **PD4** |
  | node4 | 172.16.10.4 | TiDB1 |
  | node5 | 172.16.10.5 | TiDB2 |
  | node6 | 172.16.10.6 | TiKV1 |
  | node7 | 172.16.10.7 | TiKV2 |
  | node8 | 172.16.10.8 | TiKV3 |
  | node9 | 172.16.10.9 | TiKV4 |

2. 需要将 172.16.10.103 信息刷新到 inventory.ini 配置文件

3. 初始化新增节点：
    - `ansible-playbook bootstrap.yml -l 172.16.10.103`

4. 部署新增节点：
    - `ansible-playbook deploy.yml -l 172.16.10.103`

5. 登录新增的 PD 节点，编辑启动脚本：`{{deploy_dir}}/scripts/run_pd.sh`
    - 移除  `--initial-cluster="xxxx" \`  配置
    - 添加  `--join="http://172.16.10.1:2379" \`  # IP 地址 (172.16.10.1) 可以是集群内现有 PD IP 地址中的任意一个
    - 在新增 PD 节点中手动启动 PD 服务：
        - `{{deploy_dir}}/scripts/start_pd.sh`

6. 在 tidb-ansible/resource/bin 中，使用 pd-ctl 检查新节点是否添加成功(刷新有短暂延时)：
    - `./pd-ctl -u "http://172.16.10.1:2379" -d member`

7. 滚动升级整个集群，刷新 TiKV 与 TiDB 相关配置文件 // 新版本 tidb-ansible 此处需要单独刷新监控配置文件
    - `ansible-playbook rolling_update.yml`

8. 打开浏览器访问监控平台：`http://172.16.10.3:3000`，查看整个集群节点的状态。

### 缩容 PD Cluster

如果要移除一个 PD 节点 (node2)，IP 地址为 172.16.10.2，可以进行如下操作

1. 查看 node2 节点的 name
    - `./pd-ctl -u "http://172.16.10.1:2379" -d member`
2. 从集群中移除 node2，假如 name 为 pd2
    - `./pd-ctl -u "http://172.16.10.1:2379" -d member delete name pd2`
3. 使用 pd-ctl 检查节点是否下线成功（PD 下线会很快，结果中没有 node2 节点信息即为下线成功）
    - `./pd-ctl -u "http://172.16.10.1:2379" -d member`
4. 下线成功后，停止 node2 上的服务，后续可手动删除相关数据目录
    - `ansible-playbook stop.yml -l 172.16.10.2`
5. 滚动升级整个集群，刷新 TiKV 与 TiDB 相关配置文件 // 新版本 tidb-ansible 此处需要单独刷新监控配置文件
    - `ansible-playbook rolling_update.yml`
6. 打开浏览器访问监控平台：`http://172.16.10.3:3000`，查看整个集群节点的状态

## 0x02 - TiDB

> TiDB 是无状态的，默认 4000 端口绑定在 0.0.0.0 网段。机器更改业务 IP ，只需要重启 TiDB 服务即可。
> TiDB 是通过 PD 地址获取集群 Region 与 Strore 信息，如果 PD 地址被更改，需要修改 TiDB 启动脚本。

- TiDB 脚本示例：

```bash
exec bin/tidb-server \
    -P 4000 \
    --status="10080" \
    --path="172.16.10.1:2379" \  # TiDB 连接 PD 地址
    --config=conf/tidb.toml \
    --log-file="/data1/deploy/log/tidb.log" 2> "/data1/deploy/log/tidb_stderr.log"
```

  - 脚本地址：`{{deploy_dir}}/scripts/run_tidb.sh`

## 0x03 - TiKV

> TiKV 是一个集群，通过 Raft 协议保持数据的一致性（副本数量可配置，默认保存三副本），并通过 PD 做负载均衡调度。
> TiKV 会在 PD 注册一些信息，并持续更新相关注册信息。TiKV 通过 IP:PORT 等信息在 PD 注册唯一性。
> TiKV 数据目录存放 TiKV 自身元数据信息与数据库数据，TiKV 本身可更改 IP ，IP 地址不可与 PD 已有的 store IP:PORT 重复。
> 3.0 以及后续版本增加了 metrics 端口参数

> **人工操作不当有可能导致数据损坏；通过扩缩容 TiKV 节点更换 IP 地址的方式相比该方案稳定安全**

- TiKV 脚本示例：

```bash
exec bin/tikv-server \
    --addr "0.0.0.0:20160" \  # 业务绑定
    --advertise-addr "127.0.0.1:20160" \  # 对外提供服务 IP:PORT，该地址不可为 0.0.0.0
    --pd "172.16.10.1:2379" \  # PD 地址
    --data-dir "/data1/deploy/data" \
    --config conf/tikv.toml \
    --log-file "/data1/deploy/log/tikv.log" 2> "/data1/deploy/log/tikv_stderr.log"
```

  - 脚本地址： `{{deploy_dir}}/scripts/run_tikv.sh`

## 0x04 寻找彩蛋

监控信息由 Prometheus 存储，TiDB cluster 节点信息有相关变动，并不会主动删除 Prometheus 内监控数据；在监控会看到之前的 IP:Port 信息为空或者出现数据异常，可选*通过停止 prometheus 进程、删除或迁移 Prometheus 数据目录、重启 Prometheus 进程方式暴力清理*。

> 左转站内有 Prometheus + Grafana 小知识

### 通过监控查看 PD 状态

PD cluster 三节点或多节点，只有 Leader 对外提供服务，只有提供服务节点会发送相关监控数据。
观察 PD 的信息应当观察 PD dashboard ，左上角有 instance 下选框，选择相应主机切换。

- 通过观察 PD dashboard → ETCD →  handle_txns_count 信息，可观察目前正在对外提供服务的 PD leader 节点。
- 观察  PD dashboard → ETCD →  99% wal_fsync_duration_seconds 信息，可观察到 PD cluster 中所有节点信息。

### 通过监控查看 TiDB 状态

> TiDB 为无状态组件，可通过多种方式检查 TiDB 运行状态。

- 通过查看 query 延迟
- 通过查看 TiDB 连接数
  - TiDB dashboard 中 Connection Count 信息
- 通过查看 TiDB 内存占用方式
  - TiDB dashboard 中 Heap Memory Usage 信息
- 通过监控 TiDB 端口状态，触发相关告警
- 通过 Blackbox_node export 监听 TiDB 业务端口(4000)

### 通过监控查看 TiKV 状态

- 查看 PD dashboard 中 store status 信息
- 查看 TiKV dashboard 中 leader 与 region 信息，tikv 如果不可用，相关信息应为 0 或者呈现垂直上下现象。
- 查看 TiKV dashboard 中 channel full 信息，判断 TiDB 到 TiKV 之间繁忙程度，正常状态应该为 0 或者 no data
- 查看 TiKV dashboard 中 server report failures 信息，可以判断 TiKV 与 TiKV 之间网络通信是否正常，正常状态应该为 0 或者 no data
