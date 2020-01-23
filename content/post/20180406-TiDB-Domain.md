---
title: 测试域名代替 IP 部署 TiDB
date: 2018-04-06T00:00:00+08:00
lastmod: 2020-01-20T00:00:00+08:00
categories: 
  - TiDB
tags:
  - Deploy
  - Domain
---
## 0x00 假设

> 新建机房网络频繁变更或公司内部要求禁止使用 IP 做为服务启动监听（大公司规划时喜欢使用 hostname/server name + domain 做为服务监听信息，配合公司内网 DNS 做跨地域运维管理），所以尝试下 TiDB 使用域名方式是否可以在线更换 IP 以及服务是否可以运行。
> 测试时未搭建 DNS 服务，使用 HOSTNAME 本地解析
> 测试版本 1.0

## 0x01 场景&问题

- 部署场景
  - 使用 tidb-ansible 部署 binary 方式，启动使用 supervise
  - inventory.ini 文件中直接填写域名
  - 测试场景域名未经过 DNS 解析，直接使用 `/etc/hosts` 文件解析
    - 线上场景可以使用内网 DNS 或者批量修改 `/etc/hosts` 方式对 `hostname | domain` 映射管理

- 测试场景
  - 域名部署未影响实际功能，因此只测试个组件之间网络延迟与通讯是否正常
  - PD 更换 IP 地址映射，用于修改主机 IP 或者迁移时场景
  - TiDB 到 PD 之间通讯
  - TiKV 到 PD 之间通讯
  - PD 与 PD 之间通讯

- 问题
  - PD 启动时，`client-urls` 与 `peer-urls` 参数不能使用域名格式，只能使用 IP 或者 `0.0.0.0`
    - 参考 [invalid example: "http://example.com:2379" (domain name is invalid for binding)](https://coreos.com/etcd/docs/latest/v2/configuration.html)
    - 参考 [Configuration](https://coreos.com/etcd/docs/latest/faq.html)
  - TiKV 启动时，`--addr` 参数不能使用域名格式，只能使用 IP 或者 `0.0.0.0`
    - `--advertise-addr "tikv.p.cc:20160"` 无需添加 `http://`
  - 测试 TiDB 域名与 IP 行为一致，无特殊影响

### 部署架构

域名 | 服务 | 初始IP | 修改后
---|----|------|----
pd.p.cc | PD | 127.0.0.1 | 172.16.10.64
kv.p.cc | PD | 127.0.0.1 | 172.16.10.64
db.p.cc | PD | 127.0.0.1 | 172.16.10.64

### PD 启动参数调整

```bash
exec bin/pd-server \
    --name="pd1" \
    --client-urls="http://0.0.0.0:2379" \
    --advertise-client-urls="http://pd.p.cc:2379" \
    --peer-urls="http://0.0.0.0:2380" \
    --advertise-peer-urls="http://pd.p.cc:2380" \
    --data-dir="/data1/domain/data.pd" \
    --initial-cluster="pd1=http://pd.p.cc:2380" \
    --config=conf/pd.toml \
    --log-file="/data1/domain/log/pd.log" 2> "/data1/domain/log/pd_stderr.log"
```

### TiKV 参数启动示例

```bash
exec bin/tikv-server \
    --addr "0.0.0.0:20160" \
    --advertise-addr "tikv.p.cc:20160" \
    --pd "http://pd.p.cc:2379" \
    --data-dir "/data1/domain/data" \
    --config conf/tikv.toml \
    --log-file "/data1/domain/log/tikv.log" 2> "/data1/domain/log/tikv_stderr.log"
```

## 0x02 测试结果

- TiDB 到 PD 之间网络通信
  - PD 使用域名启动，如果经过 DNS 解析，TiDB 获取 TSO 会慢 200us 上下
  - PD cluster 置于 haproxy 后面，实际通讯时 tidb -- haproxy -- 某个 etcd(etcd proxy) -- pd leader ，会比直接连接 PD 慢 600us - 1ms 左右
- PD 到 PD
  - 部署单节点，未达到测试目的
- TiKV 到 PD
  - 部署单节点，未参与 PD 切换 leader 时对 TiKV 的影响
- TiKV 到 TiKV
  - 多实例部署未受到影响
  - 域名造成通讯延迟

### 测试修改 IP

- 修改 PD
  - 正常启动
- 修改 TiDB
  - TiDB 无状态，正常使用
- 修改 TiKV
  - ok ，需要重启
  - TiKV 是将 ip:port 注册到 pd etcd 信息中，如果重复，已有的必须是 tomestone ， 否则出现以下错误，地址被占用，无法生成新的 id

  ```log
  2018/03/20 15:50:14.436 util.rs:315: [ERROR] fail to request: Grpc(RpcFailure(RpcStatus { status: Unknown, details: Some("duplicated store address: id:1001 address:\"kv.p.cc:20160\" , already registered by id:1 address:\"kv.p.cc:20160\" ") }))
  2018/03/20 15:50:55.313 tikv-server.rs:263: [ERROR] failed to start node: Pd(Other(StringError("[src/pd/util.rs:323]: fail to request")))
  ```

### Etcdctl 工具

> PD 组件数据存储在 etcd 中，etcdctl 工具是管理 ectd 存储的工具；谨慎使用该工具，该工具可重置 etcd 存储从而导致 PD 数据丢失

- 获取 PD 内已注册所有字段

```bash
[root@DB etcd-v3.1.11-linux-amd64]# ./etcdctl  --endpoints=0.0.0.0:2379 get --prefix --keys-only /

/pd/6545739204143641259/leader

/pd/6545739204143641259/timestamp

/pd/cluster_id
```

- 获取 /pd 为首的所有字段；部分信息未转码成功而乱码

```bash
[root@DB etcd-v3.1.11-linux-amd64]# ./etcdctl  --endpoints=0.0.0.0:22379 get --prefix /pd

/pd/6545739204143641259/leader

pd1ȴ¦䇃͗http://pd.p.cc:2380"http://pd.p.cc:2379

/pd/6545739204143641259/timestamp
&»WI7

/pd/cluster_id
Zգ Tr«
```

- 当前仅 PD 启动
  - json 输出格式中，字符串通过 Base64 加密格式化

```json
[root@DB etcd-v3.1.11-linux-amd64]# ./etcdctl  --endpoints=0.0.0.0:2379 get --prefix=true /pd --write-out="json"

{
    "header": {
        "cluster_id": 8216837081137490000, 
        "member_id": 6313215679864995000, 
        "revision": 366, 
        "raft_term": 2
    }, 
    "kvs": [
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvbGVhZGVy", 
            "create_revision": 3, 
            "mod_revision": 3, 
            "version": 1, 
            "value": "CgNwZDEQx/SdpuSGw85XGhRodHRwOi8vcGQucC5jYzoyMjM4MCIUaHR0cDovL3BkLnAuY2M6MjIzNzk=", 
            "lease": 8811119877456664000
        }, 
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvdGltZXN0YW1w", 
            "create_revision": 4, 
            "mod_revision": 366, 
            "version": 363, 
            "value": "FSaDMWkxK6c="
        }, 
        {
            "key": "L3BkL2NsdXN0ZXJfaWQ=", 
            "create_revision": 2, 
            "mod_revision": 2, 
            "version": 1, 
            "value": "WtcjIBdUcqs="
        }
    ], 
    "count": 3
}
```

- 当前启动一个 PD ；两个 TiKV ，其中一个 Down 状态，另一个 UP 状态
  - json 输出格式中，字符串通过 Base64 加密格式化

```json
[root@DB etcd-v3.1.11-linux-amd64]# ./etcdctl  --endpoints=0.0.0.0:2379 get --prefix=true  /pd --write-out="json"

{
    "header": {
        "cluster_id": 8216837081137490000, 
        "member_id": 6313215679864995000, 
        "revision": 934, 
        "raft_term": 2
    }, 
    "kvs": [
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvYWxsb2NfaWQ=", 
            "create_revision": 693, 
            "mod_revision": 693, 
            "version": 1, 
            "value": "AAAAAAAAA+g="
        }, 
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvY29uZmln", 
            "create_revision": 697, 
            "mod_revision": 697, 
            "version": 1, 
            "value": "eyJjbGllbnQtdXJscyI6IiIsInBlZXItdXJscyI6IiIsImFkdmVydGlzZS1jbGllbnQtdXJscyI6IiIsImFkdmVydGlzZS1wZWVyLXVybHMiOiIiLCJuYW1lIjoiIiwiZGF0YS1kaXIiOiIiLCJpbml0aWFsLWNsdXN0ZXIiOiIiLCJpbml0aWFsLWNsdXN0ZXItc3RhdGUiOiIiLCJqb2luIjoiIiwibGVhc2UiOjAsImxvZyI6eyJsZXZlbCI6IiIsImZvcm1hdCI6IiIsImRpc2FibGUtdGltZXN0YW1wIjpmYWxzZSwiZmlsZSI6eyJmaWxlbmFtZSI6IiIsImxvZy1yb3RhdGUiOmZhbHNlLCJtYXgtc2l6ZSI6MCwibWF4LWRheXMiOjAsIm1heC1iYWNrdXBzIjowfX0sImxvZy1maWxlIjoiIiwibG9nLWxldmVsIjoiIiwidHNvLXNhdmUtaW50ZXJ2YWwiOiIwcyIsIm1ldHJpYyI6eyJqb2IiOiIiLCJhZGRyZXNzIjoiIiwiaW50ZXJ2YWwiOiIwcyJ9LCJzY2hlZHVsZSI6eyJtYXgtc25hcHNob3QtY291bnQiOjMsIm1heC1wZW5kaW5nLXBlZXItY291bnQiOjE2LCJtYXgtbWVyZ2UtcmVnaW9uLXNpemUiOjAsIm1heC1zdG9yZS1kb3duLXRpbWUiOiIxaDBtMHMiLCJsZWFkZXItc2NoZWR1bGUtbGltaXQiOjY0LCJyZWdpb24tc2NoZWR1bGUtbGltaXQiOjE2LCJyZXBsaWNhLXNjaGVkdWxlLWxpbWl0IjoyNCwibWVyZ2Utc2NoZWR1bGUtbGltaXQiOjIwLCJ0b2xlcmFudC1zaXplLXJhdGlvIjoyLjUsInNjaGVkdWxlcnMtdjIiOlt7InR5cGUiOiJiYWxhbmNlLXJlZ2lvbiIsImFyZ3MiOm51bGx9LHsidHlwZSI6ImJhbGFuY2UtbGVhZGVyIiwiYXJncyI6bnVsbH0seyJ0eXBlIjoiaG90LXJlZ2lvbiIsImFyZ3MiOm51bGx9LHsidHlwZSI6ImxhYmVsIiwiYXJncyI6bnVsbH1dfSwicmVwbGljYXRpb24iOnsibWF4LXJlcGxpY2FzIjoxLCJsb2NhdGlvbi1sYWJlbHMiOiIifSwibmFtZXNwYWNlIjp7fSwicXVvdGEtYmFja2VuZC1ieXRlcyI6IjAgQiIsImF1dG8tY29tcGFjdGlvbi1yZXRlbnRpb24iOjAsIlRpY2tJbnRlcnZhbCI6IjBzIiwiRWxlY3Rpb25JbnRlcnZhbCI6IjBzIiwic2VjdXJpdHkiOnsiY2FjZXJ0LXBhdGgiOiIiLCJjZXJ0LXBhdGgiOiIiLCJrZXktcGF0aCI6IiJ9LCJsYWJlbC1wcm9wZXJ0eSI6bnVsbCwiV2FybmluZ01zZ3MiOm51bGwsIm5hbWVzcGFjZS1jbGFzc2lmaWVyIjoiIn0="
        }, 
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvbGVhZGVy", 
            "create_revision": 3, 
            "mod_revision": 3, 
            "version": 1, 
            "value": "CgNwZDEQx/SdpuSGw85XGhRodHRwOi8vcGQucC5jYzoyMjM4MCIUaHR0cDovL3BkLnAuY2M6MjIzNzk=", 
            "lease": 8811119877456664000
        }, 
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvcmFmdA==", 
            "create_revision": 694, 
            "mod_revision": 694, 
            "version": 1, 
            "value": "CKvl0bqB5MjrWhAB"
        }, 
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvcmFmdC9yLzAwMDAwMDAwMDAwMDAwMDAwMDAy", 
            "create_revision": 694, 
            "mod_revision": 694, 
            "version": 1, 
            "value": "CAIiBAgBEAEqBAgDEAE="
        }, 
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvcmFmdC9zLzAwMDAwMDAwMDAwMDAwMDAwMDAx", 
            "create_revision": 694, 
            "mod_revision": 695, 
            "version": 2, 
            "value": "CAESEjE3Mi4xNi4xMC42NToyMjE2MA=="
        }, 
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvcmFmdC9zLzAwMDAwMDAwMDAwMDAwMDAwMDA0", 
            "create_revision": 856, 
            "mod_revision": 856, 
            "version": 1, 
            "value": "CAQSDWt2LnAuY2M6MjIxNjA="
        }, 
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvcmFmdC9zdGF0dXMvcmFmdF9ib290c3RyYXBfdGltZQ==", 
            "create_revision": 694, 
            "mod_revision": 694, 
            "version": 1, 
            "value": "FSaEFqfEqcU="
        }, 
        {
            "key": "L3BkLzY1NDU3MzkyMDQxNDM2NDEyNTkvdGltZXN0YW1w", 
            "create_revision": 4, 
            "mod_revision": 934, 
            "version": 926, 
            "value": "FSaEvjZofRE="
        }, 
        {
            "key": "L3BkL2NsdXN0ZXJfaWQ=", 
            "create_revision": 2, 
            "mod_revision": 2, 
            "version": 1, 
            "value": "WtcjIBdUcqs="
        }
    ], 
    "count": 10
}
```

- 获取 Store 状态
  - IP 与域名均可注册到 PD Cluster

```json
[root@DB etcd-v3.1.11-linux-amd64]# curl http://pd.p.cc:2379/pd/api/v1/stores
{
  "count": 2,
  "stores": [
    {
      "store": {
        "id": 1,
        "address": "172.16.10.65:20160", # IP 注册
        "state_name": "Down"
      },
      "status": {
        "leader_weight": 1,
        "region_weight": 1
      }
    },
    {
      "store": {
        "id": 4,
        "address": "kv.p.cc:20160", # 域名注册
        "state_name": "Up"
      },
      "status": {
        "capacity": "118 GiB",
        "available": "23 GiB",
        "leader_weight": 1,
        "region_weight": 1,
        "start_ts": "2018-04-18T19:44:51+08:00",
        "last_heartbeat_ts": "2018-04-18T19:45:51.567575991+08:00",
        "uptime": "1m0.567575991s"
      }
    }
  ]
}
```