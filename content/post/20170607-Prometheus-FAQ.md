---
title: 监控架构 - TIDB 监控运维 FAQ
date: 2017-06-07T00:00:00+08:00
lastmod: 2020-01-23T00:00:00+08:00
categories:
  - Monitoring
tags:
  - Prometheus
  - Grafana
  - TiDB
---
## 0x00 唠叨

> 遇见了、记下了，后面或许会遇见；经历过的事件只记下了标签、这里或许是个解答题
> 部分问题会因使用版本或场景解决方案不一，仅供参考 如有雷同 纯属我剽；

## 0x01 Prometheus 问题

### 升级 Prometheus

> 大致时间 2017Q3 当时 Prometheus 出了 2.0，tidb-ansible 默认部署 1.8 的版本，2.0 版本有一些我也不知道的优化（迷信新版本肯定有提升）；然后尝试在 1.8 上升级新版本

Prometheus 升级方案比较简单，和 TiDB 相同都是由 golang 语言开发和静态编译的 binary，替换新的 binary 文件然后重启服务即可生效。然我没看官方文档的亏，替换后升级失败；向 GitHub repo 学习后发现 1.8 和 2.0 底层 openTSDB（官方自称，*资料查询是 levelDB 进化而来* ） 时序数据库不兼容了，也就是 2.0 版本无法读取和写入 1.8 版本生成的数据。哎…… 数据不过渡啊、更换目录后折腾发现配置文件还有部分不兼容

  ```LOG
  time="2017-08-08T15:38:30+08:00" level=info msg="Starting prometheus (version=1.5.2, branch=master, revision=bd1182d29f462c39544f94cc822830e1c64cf55b)" source="main.go:75"
  time="2017-08-08T15:38:30+08:00" level=info msg="Build context (go=go1.7.5, user=root@a8af9200f95d, date=20170210-14:41:22)" source="main.go:76"
  time="2017-08-08T15:38:30+08:00" level=info msg="Loading configuration file /tidb-data/tidb/deploy/conf/prometheus.yml" source="main.go:248"
  time="2017-08-08T15:38:30+08:00" level=error msg="Error opening memory series storage: found existing files in storage path that do not look like storage files compatible with this version of Prometheus; please delete the files in the storage path or choose a different storage path" source="main.go:182"
  ```

- TiDB ansible 部署历史版本
  - Prometheus 1.6
  - Prometheus 1.8
    - 1.8 与 2.0 数据不兼容
  - Prometheus 2.0 配置文件与 1.8 版本部分不兼容
    - 新增 Prometheus [tsdb-admin-apis](https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis) 功能，该功能在 devops 平台中有较好的扩展性
    - 告警配置文件 alert.rule 格式该为 yml

### 监控数据异常

有一套数据量比较大的 TiDB 集群，大概 80 \* TiKV、10 \* TiDB、3 \* PD 部署在 60 台物理机上，某天巡检发现监控数据出现了些问题；不定期出现几分钟的无数据状态。查看日志发现下面日志信息：

  ```LOG
  time="2017-10-24T10:08:30+08:00" level=warning msg="Storage has entered rushed mode." chunksToPersist=107071 maxChunksToPersist=524288 maxMemoryChunks=10485
  76 memoryChunks=1139421 source="storage.go:1660" urgencyScore=0.8663654327392578

  time="2017-10-24T10:05:39+08:00" level=info msg="Storage has left rushed mode." chunksToPersist=108398 maxChunksToPersist=524288 maxMemoryChunks=1048576 mem
  oryChunks=1121866 source="storage.go:1647" urgencyScore=0.6989479064941406

  time="2017-10-24T10:05:45+08:00" level=warning msg="Storage has entered rushed mode." chunksToPersist=112409 maxChunksToPersist=524288 maxMemoryChunks=10485
  76 memoryChunks=1268143 source="storage.go:1660" urgencyScore=1
  ```

界面展示为数据短暂断崖式下降,日志中打印 `Storage has entered rushed mode.` 信息；通过这些信息在网上冲浪，发现 GitHub Prometheus 有一些类似的问题

- 相关 Github Prometheus issue
  - https://github.com/prometheus/prometheus/issues/2542
  - https://github.com/coreos/prometheus-operator/issues/304
  - https://github.com/prometheus/prometheus/issues/2936
  - https://github.com/prometheus/prometheus/issues/2222

根据一顿胡聊聊理解，大致说从各节点上获取到的数据无法在短时间内刷写到磁盘上，然后导致临时存放在内存中的数据超时丢弃了。辣么还要研究下怎么这么个现象……

继续探索 Prometheus 的 storage 相关的参数，看有什么方案

- Prometheus [内存参数 & storage 参数](http://www.jianshu.com/p/36f72490a2a0)介绍

未完待续

### 启动速度慢

- Prometheus 启动后，发现网页无法打开，查看日志后，发现 Prometheus 启动后一直在 GC
  - Prometheus 2.0 与 1.0.8 底层存储格式不一致，这个在 2.0 后续版本上有一些优化，但数据量超过 500G 启动速度还是直线下降
  - 一般是修改参数后重启进程，官方文档提示可以使用 kill -SIGHUP PID 用来重新加载配置文件、或者使用 2.1 版本上的 admin api 功能

  ```bash
  [tidb@Jeff scripts]$ ./run_prometheus.sh
  level=info ts=2018-04-17T03:04:31.495865947Z caller=main.go:220 msg="Starting Prometheus" version="(version=2.2.0, branch=HEAD, revision=f63e7db4cbdb616337ca877b306b9b96f7f4e381)"
  level=info ts=2018-04-17T03:04:31.495932795Z caller=main.go:221 build_context="(go=go1.10, user=root@52af9f66ce71, date=20180308-16:40:42)"
  level=info ts=2018-04-17T03:04:31.495952002Z caller=main.go:222 host_details="(Linux 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 Jeff (none))"
  level=info ts=2018-04-17T03:04:31.495966987Z caller=main.go:223 fd_limits="(soft=1000000, hard=1000000)"
  level=info ts=2018-04-17T03:04:31.498497082Z caller=main.go:504 msg="Starting TSDB ..."
  level=info ts=2018-04-17T03:04:31.498536225Z caller=web.go:382 component=web msg="Start listening for connections" address=:9090
  level=info ts=2018-04-17T03:05:52.153499505Z caller=main.go:514 msg="TSDB started"
  level=info ts=2018-04-17T03:05:52.153580197Z caller=main.go:588 msg="Loading configuration file" filename=/data1/deploy/conf/prometheus.yml
  level=info ts=2018-04-17T03:05:52.177873304Z caller=main.go:491 msg="Server is ready to receive web requests."
  level=info ts=2018-04-17T03:05:57.320867461Z caller=compact.go:394 component=tsdb msg="compact blocks" count=1 mint=1523923200000 maxt=1523930400000
  level=info ts=2018-04-17T03:06:03.658563096Z caller=head.go:348 component=tsdb msg="head GC completed" duration=227.952838ms
  level=info ts=2018-04-17T03:06:06.403176186Z caller=head.go:357 component=tsdb msg="WAL truncation completed" duration=2.74453888s
  level=info ts=2018-04-17T03:06:06.666573558Z caller=compact.go:394 component=tsdb msg="compact blocks" count=3 mint=1523901600000 maxt=1523923200000
  ```

- 查看 metrics 数据目录大小

    ```bash
    du -sh *

    89G prometheus2.0.0.data.metrics
    ```

- 查看 Prometheus 具体版本信息
  - 查看官方 changelog 信息，在 Prometheus 2.0.2 对启动速度有优化

  ```bash
  [tidb@Jeff deploy]$ ./prometheus --version
  prometheus, version 2.2.0 (branch: HEAD, revision: f63e7db4cbdb616337ca877b306b9b96f7f4e381)
    build user:       root@52af9f66ce71
    build date:       20180308-16:40:42
    go version:       go1.10
  ```

### 监控数据清理

- 节点下线后，metric 由残留数据
  - 第一种方案：
    - 停止 Prometheus 与 Push Gateway 服务，然后删除 Prometheus 的所有监控数据，然后启动两个组件。
  - 第二种方案：
    - 人工修改 metric ，按个过滤掉所有已经被下线的节点信息
    - histogram_quantile(0.99, sum(rate(tidb_server_handle_query_duration_seconds_bucket{instance!="6a5114505e5f_4000"}[1m])) by (le, instance))
  - 第三种方案：
    - 目前只能通过 http api 删除历史数据，不能删除 instance 信息
    - 且版本必须高于 2.1.0 以上，官方文档 `https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis`
      - `curl -XPOST -g 'http://172.16.10.65:29090/api/v1/admin/tsdb/delete_series?match[]=tidb_server_handle_query_duration_seconds_bucket'`
      - `curl -XPOST -g 'http://172.16.10.65:29090/api/v1/admin/tsdb/clean_tombstones'`

## 0x02 Grafana 问题

### Grafana 忘记密码

#### 重新部署

> 一般情况 Grafana 公司不会重度使用监控自带的鉴权系统，所以仅做监控展示的 Grafana 的 sqllite 里并木有神马数据。重新部署一套监控然后添加 Prometheus 数据源是最简单的方式。
> 比较烦人的是那些 dashboard 需要重新导入，而 tidb-ansible 会自动化导入那些默认的 dashboard、那些后期修改的就没办法了
> 适合执行力度快的同学

- 删除 Grafana 目录下数据，dashboard 二次自定义将会丢失
- 执行 `ansible-playbook deploy.yml --tags=grafana` 部署
- 执行 `ansible-playbook start.yml --tags=grafana` 启动

#### 暴力重置 Grafana db

> 该方式与上述方案结果相同，会丢失 dashboard 信息

1. 停止 Grafana
2. 找到 `grafana.ini` 文件，并将 grafana.ini 文件中 admin_password 修改为 admin
3. 删除掉 `grafana.db` 文件；文件默认在 `{{deploy_dir}}/data.grafana/grafana.db`
4. 重启 Grafana 服务
5. 可选步骤 // 通过 tidb-ansible 脚本导入 dashboard
   - 只有当前 Grafana 是通过 tidb-ansible 部署时才可使用
   - 执行 `ansible-playbook start.yml --tags=grafana` 导入
6. 可选步骤 // 手动导入 dashboard
   - 登陆 grafana ui ，添加监控数据源，如 Prometheus
   - 登陆 grafana ui ，选择 import dashboard

#### Grafana ctl 工具

- [官方文档](https://grafana.com/docs/administration/cli/#reset-admin-password) 链接，此方法仅适用于 4.1 以上版本
- 首先找到 grafana db 文件
- 直接修改 grafana db
  - grafana-cli admin reset-admin-password ...
  - grafana-cli admin reset-admin-password --homepath "/usr/share/grafana" newpass

#### sqllite 修改数据库

使用 sql lite 工具修改 Grafana.db ; 执行以下步骤会强制将 Grafana 管理员密码重置为 admin

1. 停止 Grafana 服务
2. 通过 grafana.ini 配置文件找到 grafana.db 实际目录
     - 通过 `find / -name "grafana.db"` 查找，需要确保是 grafana.ini 文件中引用的
     - ini 文件默认在 `{{deploy_dir}}/opt/grafana/conf/grafana.ini`
     - db 文件默认在 `{{deploy_dir}}/data.grafana/grafana.db`
3. 通过 sqllite cliet 链接数据库修改密码

    ```sql
    # sqlite3 {{deploy_dir}}/data.grafana/grafana.db

    sqlite> select salt, password from user;
    pyaUhfDzYg|54c7d1ce2eeaa6000bd84407d0f8ab4663dfa575e0a326bc70dc5cab4b864f6677b21879dbf5e33427c88f9160f744b625bf

    sqlite> update user set password = '59acf18b94d7eb0694c61e60ce44c110c7a683ac6a8f09580d626f90f4a242000746579358d77dd9e570e83fa24faa88a8a6', salt = 'F3FAxVm33R' where login = 'admin';

    sqlite> .exit
    ```
