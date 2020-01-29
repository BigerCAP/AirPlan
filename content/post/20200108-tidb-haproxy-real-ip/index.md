---
title: TiDB & Haproxy / show processlist View real IP
date: 2020-01-08T00:00:00+08:00
lastmod: 2020-01-10T00:00:00+08:00
categories:
  - TiDB
  - Software
tags:
  - TiDB
  - Haproxy
  - real server ip
  - load balance
---
## 0x00 - 背景

高可用性（英语：high availability，缩写为 HA）测试 failpoint 场景，需要在服务前面添加 load balance 组件；比如 F5、LVS、Haproxy 等支持 TCP 协议的网络分发负载工具。

之前有篇文档介绍 TiDB & Haproxy [HAproxy – Load Balance Service](/post/20170610-HAproxy "docs.tidb.cc HAproxy – Load Balance Service https://ap.tidb.cc/post/20170610-haproxy/") 使用姿势 / 但随后在运维期间遇见一些小问题；即 TiDB 部署在 Haproxy 后面无法通过 show processlist 查看 client 真实 IP 地址信息，每次 show processlist 看到的都是 Haproxy 主机信息。

> Haproxy Proxy Architecture Overview

![Proxy Architecture Overview](https://www.percona.com/blog/wp-content/uploads/2015/10/cluster3.png)

## 0x01 - 场景复现

|机器 |IP | 备注|
|----|:----|----|
| A | 10.0.1.4 | 部署 Haproxy、TiDB 4000 与 TiDB 4001
| A | 10.0.1.4 | 外网 IP 为 138.x.x.x
| B | 42.x.x.x | 客户端所在机器

通过 [HAproxy – Load Balance Service](/post/20170610-HAproxy "docs.tidb.cc HAproxy – Load Balance Service https://ap.tidb.cc/post/20170610-haproxy/")文档中安装 haproxy ； yum install haproxy 并使用文档中的配置文件。

Haproxy 配置文件栗子如下：

```c
global
    log /dev/log local0
    maxconn 65536
    ulimit-n 131086
    quiet
    pidfile /var/run/haproxy.pid

defaults
    option httplog
    option redispatch
    option nolinger
    option dontlognull
    retries 3
    contimeout 5000
    clitimeout 50000
    srvtimeout 50000
    log 127.0.0.1 local3

listen testip
    bind :3308
    mode tcp
    balance roundrobin
    server tidb2 10.0.1.4:4001 weight 1
```

使用 `./tidb-server -path /tmp/nodb -P 4001` 方式在本地启动一个 mocktikv TiDB-Server 服务；注：TiDB 启动后自带的 root 用户无法远程登陆，需要新建带密码用户才可在远程机器登陆。

在 B 机器执行 `mysql -h 138.x.x.x -u st -P 4001 -p` 执行结果如下

```sql
mysql> show processlist;
+------+------+--------------+------+---------+------+-------+------------------+
| Id   | User | Host         | db   | Command | Time | State | Info             |
+------+------+--------------+------+---------+------+-------+------------------+
|    1 | st   | 42.X.X.X |      | Query   |    0 | 2     | show processlist |
+------+------+--------------+------+---------+------+-------+------------------+
1 row in set (0.00 sec)
```

在 B 机器执行 `mysql -h 138.x.x.x -u st -P 3308 -p` 执行结果如下；通过 Haproxy 负载均衡连接数据库，在数据库无法区分客户端机器 IP 地址是多少；

```sql
mysql> show processlist;
+------+------+--------------+------+---------+------+-------+------------------+
| Id   | User | Host         | db   | Command | Time | State | Info             |
+------+------+--------------+------+---------+------+-------+------------------+
|    1 | st   | 42.X.X.X |      | Sleep   |  597 | 2     |                  |
|    2 | st   | 10.0.1.4     |      | Query   |    0 | 2     | show processlist |
+------+------+--------------+------+---------+------+-------+------------------+
2 rows in set (0.00 sec)
```

此处通过 Haproxy 连接到数据库后，Host 字段显示 IP 地址为 Haproxy 所在主机地址。当 client 与 real server（TiDB-server）数量比较大的时候，无法准确判断数据流量来源；不方便运维管理。

## 0x02 - 探索与优化

经过一番搜索和学习，duck duck 过后看到 Percona 的一篇基于 [Percona XtraDB & Haproxy](https://www.percona.com/blog/2015/10/15/proxy-protocol-percona-xtradb-cluster-quick-guide/ "Proxy Protocol and Percona XtraDB Cluster: A Quick Guide") 架构的文档；文档中介绍使用 Proxy Protocol Netowrk 产品功能来修复以上问题，此处需要先看下 TiDB-Server 是否支持类似的功能

在 [TiDB-server 配置文件](https://github.com/pingcap/tidb/blob/master/config/config.toml.example) 看到以下参数信息；从注释内容字面理解上与 Percona xtraDB 中的 Proxy Protocol 是类似的。

```toml
[proxy-protocol]
# PROXY protocol acceptable client networks.
# Empty string means disable PROXY protocol, * means all networks.
networks = ""

# PROXY protocol header read timeout, unit is second
header-timeout = 5
```

需要查到相关 PR 确认下代码块或者提交人，方便后续出问题做些咨询；通过 [sourcegraph.com](https://sourcegraph.com/github.com/pingcap/tidb) 网站搜寻当时这段配置文件的作者；中间过程省略，最终查到 issue 地址为 [TiDB issue #3757](https://github.com/pingcap/tidb/pull/3757 "2017 年 7 月 14 日提交的代码，2017 年 11 月 26 日合并的 PR / TiDB 1.0 GA 后的功能")。

> PR 重要内容如下:

```text
Add PROXY protocol V1 and V2 support.
usage: tidb-server --proxy-protocol-networks "*" --proxy-protocol-header-timeout 5
Add --proxy-protocol-networks command parameter for PROXY protocol enable or disable. If you want to limit HAProxy server IP range, you can set --proxy-protocol-networks parameter to a CIDRs and split by ",". For example:
tidb-server --proxy-protocol-networks "192.168.1.0/24,192.168.2.0/24"
For more information about PROXY protocol please refer https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt
```

根据内容表述，使用 `--proxy-protocol-networks "*"` 启用 Proxy 协议，支持 v1 and v2 版本；如果需要限制 TiDB Proxy 数据源，可以使用 `--proxy-protocol-networks "192.168.1.0/24,192.168.2.0/24"` CIDRs *「无类别域间路由（Classless Inter-Domain Routing、CIDR）是一个用于给用户分配IP地址以及在互联网上有效地路由IP数据包的对IP地址进行归类的方法」* 声明，并使用 **逗号** 分割。

> 同时还给出了 Haproxy 配置文件栗子

```c
global
    log /dev/log local0
    maxconn 65536
    ulimit-n 131086
    quiet
    pidfile /var/run/haproxy.pid

defaults
    option httplog
    option redispatch
    option nolinger
    option dontlognull
    retries 3
    contimeout 5000
    clitimeout 50000
    srvtimeout 50000
    log 10.0.1.4 local3

listen web
    bind :3307
    mode tcp
    balance roundrobin
    server tidb1 10.0.1.4:4000 send-proxy weight 1
```

Haproxy 配置文件中 global 与 defaults 与其他一般无二；重点在 real server 中的 **send-proxy** 参数信息；

按照 PR 内容启动了新的 TiDB 进程：`./tidb-server --proxy-protocol-networks "10.0.1.0/24"`

在 B 机器执行 `mysql -h 138.X.X.X -u te -P 4000 -p` 登陆主机，并执行 `select sleep(100)` 用于区分链接； 查看结果如下：

```sql
mysql> show processlist;
+------+------+--------------+------+---------+------+-------+------------------+
| Id   | User | Host         | db   | Command | Time | State | Info             |
+------+------+--------------+------+---------+------+-------+------------------+
|    3 | te   | 42.X.X.X |      | Query   |    0 | 2     | show processlist |
|    4 | te   | 42.X.X.X |      | Query   |    6 | 2     | select sleep(100) |
+------+------+--------------+------+---------+------+-------+------------------+
1 row in set (0.00 sec)
```

在 B 机器执行 `mysql -h 138.X.X.X -u te -P 3307 -p` 查看结果如下：

```sql
mysql> show processlist;
+------+------+--------------+------+---------+------+-------+-------------------+
| Id   | User | Host         | db   | Command | Time | State | Info              |
+------+------+--------------+------+---------+------+-------+-------------------+
|    4 | te   | 42.X.X.X |      | Query   |    6 | 15    | select sleep(100) |
|    5 | te   | 42.X.X.X |      | Query   |    0 | 2     | show processlist  |
+------+------+--------------+------+---------+------+-------+-------------------+
2 rows in set (0.00 sec)
```

由此发现，该功能使用姿势暂无问题，同时解决了上述 show processlist Host 字段无法看到 real sercer IP 的问题。

## 0x03 - 没有彩蛋

PR 最底部还有一些关于 Proxy Protocol 的讨论

> - Proxy Protocol 扩展阅读
>   - 扩展阅读 [MariaDB 源码分析 Proxy Protocol](http://mysql.taobao.org/monthly/2019/01/07/ "aliyun RDS 团队文档")
>   - 扩展 issue [blacktear23/go-proxyprotocol/pull/1](https://github.com/blacktear23/go-proxyprotocol/pull/1 "该 issue 不在 PingCAP/TiDB repo 下，由该作者自行控制，可能会被删除。")

```markdown
1. most proxy protocol implementations such as MariaDB, Percona server, apache(via mod_remoteip), allow user to specify a list of cider blocks（`proxy-protocol-networks` for MariaDB and Percona server，`RemoteIPProxyProtocolExceptions ` for apache), IP in the list MUST use proxy protocol and IP not in the list MUST NOT use proxy protocol(but they can still connect without proxy protocol). some implementations, like nginx and TiDB, forces all IP addresses to use proxy protocol when enables proxy protocol options.
2. the spec says that we "MUST not try to guess whether the protocol header is presentor not", to prevent security issues, but there's no guessing when we provide a list and config IPs in the list to MUST use and IPs not in the list to MUST NOT use(but still able to connect). There's no security issues doing this way and it actually meet the spec requirement.
3. allowing certain IP blocks to connect directly without haproxy will make certain tasks much easier to configure, such as:
    * monitoring individual TiDB nodes
    * make a certain TiDB node to do complex OLAP tasks without influence other OLTP nodes.
    * control individual TiDB nodes via script tasks in the local machine(such as a crontab script at each TiDB node

I think make IPs not in AllowedIPs to connect directly without proxy protocol is:

1. very easy to implement. only need to modify two lines of code
2. has some benefits
3. meets the spec

So I think its good to do this way.
```

- 以上内容重点内容是：
  - PR #3757 ｜ CIDRs 列表中所有 IP 地址仅能通过 Proxy 协议链接，如非 Proxy 协议链接将会被拒绝；但 CIDRs 列表外的 IP 地址依旧可以通过非 Proxy 协议链接（#3757 前半部分 PR 中的实现，TiDB 启用 Proxy 后所有流量都会走这个检查）。
    - 原始 Proxy protocol 在使用 CIDRs 后不支持白名单以外的 IP 连接，SteamedFish 作者代码部分 PR 代码块可以做到 **非 CIDRs 列表内的 IP** 支持以普通方式链接 TiDB ，且改动代码量足够的小
    - 通过检测 IP 地址是否存在与 CIDRs 列表内，而不是尝试获取服务端返回的 hedaer 信息（proxy v1 header 是 5 字节，v2 版本是 12 字节）；同时在处理 CIDRs 列表外的 IP 时，可以省略探测服务器协议状态。
    - 允许非白名单外的 IP 连接 TiDB 后，可以对该 TiDB 做特殊处理，如：监控节点、处理 OLAP 业务、执行人工查询等

### 场景一

> haproxy 使用以下配置文件

```c
listen web
    bind :3307
    mode tcp
    balance roundrobin
    server tidb1 127.0.0.1:4000 send-proxy weight 1
```

TiDB 使用命令启动 `./tidb-server --host 10.0.1.4 --proxy-protocol-networks "10.0.2.0/24"`

使用 `mysql -h 138.X -u te -P 3307 -p` 登陆时，提示 `ERROR 2013 (HY000): Lost connection to MySQL server at 'reading authorization packet', system error: 0` ；原因是 Haproxy real server IP 地址与 TiDB-server listen IP 地址不相等。

使用 `mysql -h 138.X.X.X -u te -P 4000 -p` 登陆未出现异常；Proxy Protocol  CIDRs 列表外非 Proxy 协议数据可以直接连接 TiDB-server(10.0.1.4 与 138.x.x.x 在防火墙做了端口映射)。

### 场景二

> haproxy 使用以下配置文件

```c
listen web
    bind :3307
    mode tcp
    balance roundrobin
    server tidb1 10.0.1.4:4000 send-proxy weight 1
```

TiDB 使用该模式启动 `./tidb-server --host 10.0.1.4 --proxy-protocol-networks "10.0.1.0/24"`

使用 `mysql -h 10.0.1.4 -u root -P 4000` 登陆时，提示 `ERROR 2013 (HY000): Lost connection to MySQL server at 'reading authorization packet', system error: 0` ；
原因是 Proxy networks 中的 CIDRs IP 地址包含了目前 mysql client 登陆地址。

使用 `mysql -h 10.0.1.4 -u root -P 3307` 可以登陆成功，Proxy protocol 已经被 Haproxy 做了包转发解析。
使用 `mysql -h 138.x.x.x -u root -P 4000` 可以登陆成功，因为外网 IP 不在 proxy protocols 的白名单范围内。

满足 Proxy 协议规则，CIDRs 列表内的 IP 地址必须使用 Proxy 协议链接，否则拒绝提供服务。

### 场景三

> haproxy 使用以下配置文件

```c
listen web
    bind :3307
    mode tcp
    balance roundrobin
    server tidb1 10.0.1.4:4000 send-proxy weight 1
```

TiDB 使用该模式启动 `./tidb-server --host 10.0.1.4`

使用命令经过 Haproxy 服务登陆 TiDB `mysql -h  127.0.0.1 -u root -P 3307`，返回错误 `ERROR 2013 (HY000): Lost connection to MySQL server at 'reading authorization packet', system error: 0`
后端主机不允许 Haproxy 建立链接，因 CIDRs IP 范围内的主机（ only support｜ 仅支持）proxy 协议的客户端链接。

### 场景四

> haproxy 使用以下配置文件

```c
listen web
    bind :3308
    mode tcp
    balance roundrobin
    server tidb1 10.0.1.4:4001 weight 1
```

TiDB 使用该模式启动 `./tidb-server --proxy-protocol-networks "10.0.1.0/24" -P 4001`

使用该命令经过 Haproxy 登陆 `mysql -h  127.0.0.1 -u root -P 3308`  TiDB 服务；返回错误  `ERROR 2013 (HY000): Lost connection to MySQL server at 'reading initial communication packet', system error: 0` ；查看 tidb.log 日志信息，发现是尝试判断 proxy 协议状态时失败；默认读取 proxy header 信息超时时间 5s

```c
[2020/01/09 22:02:59.274 +08:00] [ERROR] [server.go:321] ["PROXY protocol failed"] [error="Header read timeout"] [stack="github.com/pingcap/tidb/server.(*Server).Run\n\t/home/jenkins/workspace/build_tidb_master/go/src/github.com/pingcap/tidb/server/server.go:321\nmain.runServer\n\t/home/jenkins/workspace/build_tidb_master/go/src/github.com/pingcap/tidb/tidb-server/main.go:568\nmain.main\n\t/home/jenkins/workspace/build_tidb_master/go/src/github.com/pingcap/tidb/tidb-server/main.go:174\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:200"]
```

### 测试总结

|场景| Haproxy | TiDB | 备注
----|----|----|----
| 一 | send haproxy 10.0.1.4 | proxy protocols 10.0.2.0 / 24 | 链接失败，Haproxy CIP 不符合 CIDRs IP 范围
| 二 | send haproxy 10.0.1.4 | proxy protocols 10.0.1.0 / 24 | 正常工作，但无法使用 mysql -h 10.0.1.4 -P 4000 与 TiDB 直接建立链接
| 三 | no send proxy 10.0.1.4 | proxy protocols 10.0.1.0 / 24 | 无法建立链接，不符合 CIDRs IP 范围，与上一条场景类似
| 四 | send haproxy 10.0.1.4 | no proxy | 获取 proxy header 失败，建立链接失败

场景三与场景四中， proxy & CIDRs 结合 TiDB #3757 PR 信息；
先判断 IP 是否在 Proxy CIDRs 中，如果存在直接按照 Proxy 方式链接；当 IP 存在 CIDRs 中但客户端不存在 Proxy Header 消息包，会出现 header read timeout ，因为建立 Proxy 链接需要 Proxy Header 信息判断 v1 或者 v2 版本，以便传递给后端初始化链接。

## 0x04 赞

> 点赞 [PR #3757](https://github.com/pingcap/tidb/pull/3757) 作者 [Rain Li](https://github.com/blacktear23) & [SteamedFish](https://github.com/SteamedFish)
> // 人文结案
> Proxy Protocols 功能总结：引入了 proxy protocol 来解决 client ip 透传的问题。
> proxy 节点获取到 client ip 后，将 client ip 包装在 proxy protocol 报文中发给 real server，real server 解析到了 proxy protocol 报文，再用报文中的 IP 替换 Proxy 节点的 IP

![UML Proxy Protocols / TiDB / Docs.tidb.cc](http://www.plantuml.com/plantuml/png/SoWkIImgAStDuNBEoKpDAr7GjLFmI2meog-eLB1IIFOCKB1LC3JGCz0pr3FHAKRXMXaR6vXpGHM3z8LakZXPAIIYQKh6r8Hka8c1WG4NI3V2bASTIvvDMwi0LFTinlgd4vOzdRD2mTdJ9QXOuScEjU_tz3nTEmCezAAd-UdiBK_RMXytDA4Q0D89KOFGzcHlsvCTlL3H0B2hFL8JKrAB59xiN_YiSVsJFREUh-rykg_rnRRMPzEteHduhAVpUUjogBwd6zeK8046TkBi_Szw5y723IW2qmeJ0eOafey912VKV2i5mW48Ypk4M_iNFkliVjOnuMdNV4p9pkKl5lPmEQJcfG3Z7G00 "Proxy protocols & TiDB / Docs.tidb.cc")
