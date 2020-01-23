---
title: 监控架构 - Blackbox_exporter 主动探测端口服务状态
date: 2017-06-03T00:00:00+08:00
lastmod: 2020-01-20T00:00:00+08:00
categories:
  - Monitoring
tags:
  - Deploy
  - AlertManager
  - Proemtheus
  - Blackbox
---
## 0x00

> 找个能监控组件端口是否存在的服务，并且可以将数据发送到 Prometheus ；blackbox export 是 Prometheus 家族一员，按照配置文件内容用于向目标主动探测

## 0x01 Deploy

- [blackbox_exporter](https://github.com/prometheus/blackbox_exporter) 是 Prometheus 官方提供的 exporter 之一，可以提供 http、dns、tcp、icmp 的监控数据采集

- Binary
  - [Release Download](https://github.com/prometheus/blackbox_exporter/releases)
  - `./bin/blackbox_exporter --web.listen-address=:9115 --log.level=info --config.file=conf/blackbox.yml`

- Docker

  ```bash
  docker pull prom/blackbox-exporter
  docker run -d -p 9115:9115 --name blackbox_exporter -v `pwd`:/config blackbox_exporter --config.file=/config/blackbox.yml
  ```

- blackbox.yml template
  - 通过 blackbox.yml 定义模块详细信息
  - 在 Prometheus 配置文件中引用该模块以及配置被监控目标主机

### blackbox.yml 配置文件

> 如无特殊需求，使用默认配置文件即可
> [blackbox-good.yml](https://github.com/prometheus/blackbox_exporter/blob/master/config/testdata/blackbox-good.yml) 官方配置文件案例

```YAML
modules:
  http_2xx:
    prober: http
    http:
      method: GET
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"
```

## 0x02 功能测试

- HTTP 测试
  - 定义 Request Header 信息
  - 判断 Http status / Http Respones Header / Http Body 内容
- TCP 测试
  - 业务组件端口状态监听
  - 应用层协议定义与监听
- ICMP 测试
  - 主机探活机制

### Ping 测试

- 阅读 [icmp.go](https://github.com/prometheus/blackbox_exporter/blob/master/prober/icmp.go) 了解 ICMP 模块工作
- 以下内容引用 [用 Prometheus 进行网络质量 Ping 监控](https://www.iamle.com/archives/2130.html)
- 相关代码块添加到 Prometheus 配置文件内

```YAML
  - job_name: 'ping_all'
    scrape_interval: 5s
    metrics_path: /probe
    params:
      module: [icmp]  #ping
    static_configs:
      - targets: ['219.150.32.132', '219.148.204.66', '183.59.4.178', '202.96.209.5', '202.100.4.15', '219.146.0.253', '222.173.113.3', '61.139.2.69', '61.128.114.133', '202.98.234.35', '202.100.64.68', '202.100.100.1', '221.232.129.35', '222.88.19.1', '222.190.127.1', '202.96.96.68', '202.101.103.54', '61.140.99.11', '202.96.161.1', '202.103.224.74']
        labels:
          group: '一线城市 - 电信网络监控'
      - targets: ['218.8.251.163', '218.107.51.1', '221.7.34.1', '112.65.244.1', '218.25.9.38', '124.89.76.214', '202.102.152.3', '202.102.128.68', '124.161.132.1', '221.7.1.20', '27.98.234.1', '218.104.111.122', '218.28.199.1', '221.6.4.66', '221.12.31.58', '218.104.136.149', '210.21.4.130', '210.21.196.6', '221.7.128.68', '221.199.15.5']
        labels:
          group: '一线城市 - 联通网络监控'
      - targets: ['211.137.241.34', '211.136.192.1', '218.203.160.194', '117.131.0.22', '211.137.32.178', '117.136.25.1', '211.137.191.26', '211.137.180.1', '211.137.106.1', '218.202.152.130', '211.139.73.34', '223.77.0.1', '211.138.16.1', '221.131.143.1', '211.140.10.2', '211.143.195.255', '183.238.55.1', '218.204.216.65', '211.138.245.180', '211.138.56.1']
        labels:
          group: '一线城市 - 移动网络监控'
      - targets: ['61.233.168.1', '61.234.113.1', '211.98.224.87', '222.44.44.1', '222.33.8.1', '61.232.201.76', '211.98.19.1', '61.232.49.1', '211.98.208.1', '61.233.199.129', '222.34.16.1', '61.232.206.1', '61.233.79.1', '211.98.43.1', '61.232.78.1', '61.232.93.193', '211.98.161.1', '61.235.99.1', '222.52.72.66', '211.138.56.1']
        labels:
          group: '一线城市 - 铁通网络监控'
      - targets: ['219.150.32.132', '219.153.32.1', '219.149.6.99', '220.189.218.45', '123.172.77.1', '202.103.124.34', '222.172.200.68', '202.100.64.68', '222.93.201.1', '222.191.255.1', '202.101.224.68', '222.85.179.1', '220.178.79.2', '59.49.12.97', '222.223.255.1', '219.148.172.97', '219.132.199.49', '14.120.0.1', '222.223.29.5', '222.173.222.1', '202.101.107.55', '222.74.58.29']
        labels:
          group: '二线城市 - 电信网络监控'
      - targets: ['60.24.4.1', '221.5.203.86', '202.96.86.18', '221.12.33.227', '202.99.96.68', '58.20.127.238', '221.3.131.11', '124.152.0.1', '221.3.131.11', '221.6.96.1', '220.248.192.13', '221.13.28.234', '211.91.88.129', '202.99.192.66', '202.99.160.68', '202.99.224.68', '58.252.65.1', '221.4.141.254', '60.2.17.49', '202.102.154.3', '36.249.127.25', '202.99.224.8']
        labels:
          group: '二线城市 - 联通网络监控'
      - targets: ['211.137.160.5', '218.201.17.2', '211.140.192.1', '211.140.35.1', '211.141.66.209', '211.138.225.2', '111.17.191.25', '218.203.160.195', '211.103.55.51', '211.103.114.1', '183.217.255.254', '211.139.2.18', '211.138.180.2', '211.142.12.2', '111.11.85.1', '211.138.91.2', '218.204.173.197', '183.233.223.25', '211.143.60.1', '120.192.167.2', '218.207.133.3', '120.193.138.1']
        labels:
          group: '二线城市 - 移动网络监控'
      - targets: ['61.234.70.2', '211.98.114.1', '61.232.157.1', '61.234.186.1', '61.235.225.1', '211.98.71.1', '211.98.72.7', '123.81.2.9', '211.98.44.65', '222.45.86.1', '222.49.82.1', '211.98.108.1', '222.48.116.17', '211.98.149.117', '61.235.145.1', '222.39.142.1', '110.211.253.9', '222.50.104.1', '61.234.76.1', '61.232.53.113', '61.234.196.185', '222.39.156.1']
        labels:
          group: '二线城市 - 铁通网络监控'
      - targets: ['166.111.8.28', '202.112.26.34', '202.114.240.6', '202.116.160.33', '211.65.207.170', '218.26.226.100', '222.221.252.162', '202.38.64.1', '202.201.48.1', '202.118.176.1', '121.193.138.1', '222.23.245.1', '58.195.136.3', '202.118.1.28', '59.77.229.1', '121.48.177.1', '121.250.214.1', '59.75.177.1', '59.75.112.254']
        labels:
          group: '教育网络监控'
    relabel_configs:
      - source_labels: [__address__]
        regex: (.*)(:80)?
        target_label: __param_target
        replacement: ${1}
      - source_labels: [__param_target]
        regex: (.*)
        target_label: ping
        replacement: ${1}
      - source_labels: []
        regex: .*
        target_label: __address__
        replacement: 172.16.10.65:9115  # Blackbox exporter.
```

### TCP 测试

- 阅读 [tcp_test.go](https://github.com/prometheus/blackbox_exporter/blob/master/prober/tcp_test.go) 了解 TCP 工作
- 监听 TiDB TiKV PD 的业务端口地址，用来判断服务是否在线
- 相关代码块添加到 Prometheus 文件内

```YAML
- job_name: "port_test"
  scrape_interval: 30s
  metrics_path: /probe
  params:
    module: [tcp_connect]
  static_configs:
  - targets: ['172.16.10.64:2379', '172.16.10.65:2379', '172.16.10.66:2379']
    labels:
      group: 'pd'
  - targets: ['172.16.10.64:4000', '172.16.10.65:4000', '172.16.10.66:4000']
    labels:
      group: 'tidb'
  - targets: ['172.16.10.67:20160', '172.16.10.68:20160', '172.16.10.69:20160']
    labels:
      group: 'tikv'
  - targets: ['172.16.10.64:9100', '172.16.10.65:9100', '172.16.10.66:9100','172.16.10.67:9100', '172.16.10.68:9100', '172.16.10.69:9100']
    labels:
      group: 'node-export'
  - targets: ['172.16.10.67:3000', '172.16.10.67:9091', '172.16.10.67:9090']
    labels:
      group: 'monitor'
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 172.16.10.65:9115  # Blackbox 地址
```

### HTTP 测试

- [http_test.go](https://github.com/prometheus/blackbox_exporter/blob/master/prober/http_test.go)
- 相关代码块添加到 Prometheus 文件内

```YAML
  - job_name: 'http_2xx'
    scrape_interval: 5s
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - 'www.baidu.com'
        - 'www.pingcap.com'
        - 'http://172.16.10.65:2379/metrics'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 172.16.10.65:9115
```

## 0x03 告警测试

- 通过 `http://172.16.10.65:9115/probe?target=prometheus.io&module=http_2xx&debug=true` 方式可观察监听过程

  ```LOG
  Logs for the probe:
  ts=2018-04-13T03:47:36.354717876Z caller=main.go:116 module=http_2xx target=prometheus.io level=info msg="Beginning probe" probe=http timeout_seconds=9.5
  ts=2018-04-13T03:47:36.354917698Z caller=utils.go:42 module=http_2xx target=prometheus.io level=info msg="Resolving target address" preferred_ip_protocol=ip6
  ts=2018-04-13T03:47:36.403356427Z caller=utils.go:65 module=http_2xx target=prometheus.io level=info msg="Resolved target address" ip=2400:cb00:2048:1::6818:783c
  ts=2018-04-13T03:47:36.40345284Z caller=http.go:282 module=http_2xx target=prometheus.io level=info msg="Making HTTP request" url=http://[2400:cb00:2048:1::6818:783c] host=prometheus.io
  ts=2018-04-13T03:47:36.414825723Z caller=http.go:297 module=http_2xx target=prometheus.io level=error msg="Error for HTTP request" err="Get http://[2400:cb00:2048:1::6818:783c]: dial tcp [2400:cb00:2048:1::6818:783c]:80: connect: network is unreachable"
  ts=2018-04-13T03:47:36.426234543Z caller=http.go:354 module=http_2xx target=prometheus.io level=info msg="Response timings for roundtrip" roundtrip=0 start=2018-04-13T11:47:36.414648186+08:00 dnsDone=2018-04-13T11:47:36.414648186+08:00 connectDone=2018-04-13T11:47:36.414794718+08:00 gotConn=0001-01-01T00:00:00Z responseStart=0001-01-01T00:00:00Z end=2018-04-13T11:47:36.414814678+08:00
  ts=2018-04-13T03:47:36.426339526Z caller=main.go:129 module=http_2xx target=prometheus.io level=error msg="Probe failed" duration_seconds=0.071537196

  Metrics that would have been returned:
  # HELP probe_dns_lookup_time_seconds Returns the time taken for probe dns lookup in seconds
  # TYPE probe_dns_lookup_time_seconds gauge
  probe_dns_lookup_time_seconds 0.048474718
  # HELP probe_duration_seconds Returns how long the probe took to complete in seconds
  # TYPE probe_duration_seconds gauge
  probe_duration_seconds 0.071537196
  # HELP probe_failed_due_to_regex Indicates if probe failed due to regex
  # TYPE probe_failed_due_to_regex gauge
  probe_failed_due_to_regex 0
  # HELP probe_http_content_length Length of http content response
  # TYPE probe_http_content_length gauge
  probe_http_content_length 0
  # HELP probe_http_duration_seconds Duration of http request by phase, summed over all redirects
  # TYPE probe_http_duration_seconds gauge
  probe_http_duration_seconds{phase="connect"} 0
  probe_http_duration_seconds{phase="processing"} 0
  probe_http_duration_seconds{phase="resolve"} 0.048474718
  probe_http_duration_seconds{phase="tls"} 0
  probe_http_duration_seconds{phase="transfer"} 0
  # HELP probe_http_redirects The number of redirects
  # TYPE probe_http_redirects gauge
  probe_http_redirects 0
  # HELP probe_http_ssl Indicates if SSL was used for the final redirect
  # TYPE probe_http_ssl gauge
  probe_http_ssl 0
  # HELP probe_http_status_code Response HTTP status code
  # TYPE probe_http_status_code gauge
  probe_http_status_code 0
  # HELP probe_http_version Returns the version of HTTP of the probe response
  # TYPE probe_http_version gauge
  probe_http_version 0
  # HELP probe_ip_protocol Specifies whether probe ip protocol is IP4 or IP6
  # TYPE probe_ip_protocol gauge
  probe_ip_protocol 6
  # HELP probe_success Displays whether or not the probe was a success
  # TYPE probe_success gauge
  probe_success 0

  Module configuration:
  prober: http
  http:
    method: GET
  ```

### 告警测试规则

- 判断 probe_success metric 结果值，达到告警效果
  - 成功 == 1
  - 失败 == 0

- 创建 `port.rules.yml` 文件，写入以下告警规则

```yaml
  - alert: TiDB_is_Down
    expr: probe_success{group="tidb"} == 0
    for: 3m
    labels:
      env: test-cluster
      level: port
      expr: probe_success{group="tidb"} == 0
    annotations:
      description: 'alert:{{ $labels.expr }} instance:
        {{ $labels.instance }} values: {{ $value }}'
      value: '{{ $value }}'
      summary: TiDB_is_Down

  - alert: TiKV_is_Down
    expr: probe_success{group="tikv"} == 0
    for: 3m
    labels:
      env: test-cluster
      level: port
      expr: probe_success{group="tikv"} == 0
    annotations:
      description: 'alert:{{ $labels.expr }} instance:
        {{ $labels.instance }} values: {{ $value }}'
      value: '{{ $value }}'
      summary: TiKV_is_Down

  - alert: PD_is_Down
    expr: probe_success{group="pd"} == 0
    for: 3m
    labels:
      env: test-cluster
      level: port
      expr: probe_success{group="pd"} == 0
    annotations:
      description: 'alert:{{ $labels.expr }} instance:
        {{ $labels.instance }} values: {{ $value }}'
      value: '{{ $value }}'
      summary: PD_is_Down
```

### 上报 Prometheus

- Prometheus 配置文件中添加 `port.rules.yml` 字段，修改完成后重启生效
  - 如果 Prometheus 已经引入 Alertmanger 服务，Prometheus 监控到端口 down 掉后会触发告警

```yaml
rule_files:
  - 'node.rules.yml'
  - 'blacker.rules.yml'
  - 'bypass.rules.yml'
  - 'pd.rules.yml'
  - 'tidb.rules.yml'
  - 'tikv.rules.yml'
  - 'binlog.rules.yml'
  - 'port.rules.yml' # 端口监测告警规则文件
```

### Grafana 展示

- 部署 Grafana 服务
  - 下载 [Grafana](https://grafana.com/grafana/download)
  - 部署 [Grafana](http://docs.grafana.org/installation/rpm/)
- 利用以下 metrics 构建页面
  - probe_success
  - probe_duration_seconds
