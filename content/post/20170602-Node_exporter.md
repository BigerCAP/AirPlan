---
title: 监控架构 - Node_exporter 系统监控组件
date: 2017-06-02T00:00:00+08:00
lastmod: 2020-01-20T00:00:00+08:00
categories:
  - Monitoring
tags:
  - Node_exporter
  - Prometheus
---
## 0x00 起源

> 学习 node export 存的

## 0x01 Collector

- 通过 `http://IP:9100/metrics` 查看当前主机监控信息
- Node export 通过[Collector](https://github.com/prometheus/node_exporter/tree/master/collector)代码块收集系统性能指标

> 以下内容引用 Prometeus/Node\_export [README](https://github.com/prometheus/node_exporter/blob/master/README.md)
> 每个操作系统上的 Collectors 都有不同的支持。下表列出了所有现有 Collectors 和支持的系统。

通过 `--collector.<name>` 参数启用相应的 Collectors  
通过 `--no-collector.<name>` 参数禁用默认已经启动的 Collectors

### 默认开启的监控类型

Name     | Description | OS
---------|-------------|----
arp | Exposes ARP statistics from `/proc/net/arp`. | Linux
bcache | Exposes bcache statistics from `/sys/fs/bcache/`. | Linux
bonding | Exposes the number of configured and active slaves of Linux bonding interfaces. | Linux
conntrack | Shows conntrack statistics (does nothing if no `/proc/sys/net/netfilter/` present). | Linux
cpu | Exposes CPU statistics | Darwin, Dragonfly, FreeBSD, Linux
diskstats | Exposes disk I/O statistics. | Darwin, Linux
edac | Exposes error detection and correction statistics. | Linux
entropy | Exposes available entropy. | Linux
exec | Exposes execution statistics. | Dragonfly, FreeBSD
filefd | Exposes file descriptor statistics from `/proc/sys/fs/file-nr`. | Linux
filesystem | Exposes filesystem statistics, such as disk space used. | Darwin, Dragonfly, FreeBSD, Linux, OpenBSD
hwmon | Expose hardware monitoring and sensor data from `/sys/class/hwmon/`. | Linux
infiniband | Exposes network statistics specific to InfiniBand and Intel OmniPath configurations. | Linux
ipvs | Exposes IPVS status from `/proc/net/ip_vs` and stats from `/proc/net/ip_vs_stats`. | Linux
loadavg | Exposes load average. | Darwin, Dragonfly, FreeBSD, Linux, NetBSD, OpenBSD, Solaris
mdadm | Exposes statistics about devices in `/proc/mdstat` (does nothing if no `/proc/mdstat` present). | Linux
meminfo | Exposes memory statistics. | Darwin, Dragonfly, FreeBSD, Linux, OpenBSD
netdev | Exposes network interface statistics such as bytes transferred. | Darwin, Dragonfly, FreeBSD, Linux, OpenBSD
netstat | Exposes network statistics from `/proc/net/netstat`. This is the same information as `netstat -s`. | Linux
nfs | Exposes NFS client statistics from `/proc/net/rpc/nfs`. This is the same information as `nfsstat -c`. | Linux
nfsd | Exposes NFS kernel server statistics from `/proc/net/rpc/nfsd`. This is the same information as `nfsstat -s`. | Linux
sockstat | Exposes various statistics from `/proc/net/sockstat`. | Linux
stat | Exposes various statistics from `/proc/stat`. This includes boot time, forks and interrupts. | Linux
textfile | Exposes statistics read from local disk. The `--collector.textfile.directory` flag must be set. | _any_
time | Exposes the current system time. | _any_
timex | Exposes selected adjtimex(2) system call stats. | Linux
uname | Exposes system information as provided by the uname system call. | Linux
vmstat | Exposes statistics from `/proc/vmstat`. | Linux
wifi | Exposes WiFi device and station statistics. | Linux
xfs | Exposes XFS runtime statistics. | Linux (kernel 4.4+)
zfs | Exposes [ZFS](http://open-zfs.org/) performance statistics. | [Linux](http://zfsonlinux.org/)

### 默认关闭的监控类型

Name     | Description | OS
---------|-------------|----
buddyinfo | Exposes statistics of memory fragments as reported by /proc/buddyinfo. | Linux
devstat | Exposes device statistics | Dragonfly, FreeBSD
drbd | Exposes Distributed Replicated Block Device statistics (to version 8.4) | Linux
interrupts | Exposes detailed interrupts statistics. | Linux, OpenBSD
ksmd | Exposes kernel and system statistics from `/sys/kernel/mm/ksm`. | Linux
logind | Exposes session counts from [logind](http://www.freedesktop.org/wiki/Software/systemd/logind/). | Linux
meminfo\_numa | Exposes memory statistics from `/proc/meminfo_numa`. | Linux
mountstats | Exposes filesystem statistics from `/proc/self/mountstats`. Exposes detailed NFS client statistics. | Linux
ntp | Exposes local NTP daemon health to check [time](./docs/TIME.md) | _any_
qdisc | Exposes [queuing discipline](https://en.wikipedia.org/wiki/Network_scheduler#Linux_kernel) statistics | Linux
runit | Exposes service status from [runit](http://smarden.org/runit/). | _any_
supervisord | Exposes service status from [supervisord](http://supervisord.org/). | _any_
systemd | Exposes service and system status from [systemd](http://www.freedesktop.org/wiki/Software/systemd/). | Linux
tcpstat | Exposes TCP connection status information from `/proc/net/tcp` and `/proc/net/tcp6`. (Warning: the current version has potential performance issues in high load situations.) | Linux

### 命令行参数

```bash
usage: node_exporter [<flags>]

Flags:
  -h, --help                    Show context-sensitive help (also try --help-long and --help-man).
      --collector.diskstats.ignored-devices="^(ram|loop|fd|(h|s|v|xv)d[a-z]|nvme\\d+n\\d+p)\\d+$"
                                Regexp of devices to ignore for diskstats.
      --collector.filesystem.ignored-mount-points="^/(sys|proc|dev)($|/)"
                                Regexp of mount points to ignore for filesystem collector.
      --collector.filesystem.ignored-fs-types="^(sys|proc|auto)fs$"
                                Regexp of filesystem types to ignore for filesystem collector.
      --collector.megacli.command="megacli"
                                Command to run megacli.
      --collector.netdev.ignored-devices="^$"
                                Regexp of net devices to ignore for netdev collector.
      --collector.ntp.server="127.0.0.1"
                                NTP server to use for ntp collector
      --collector.ntp.protocol-version=4
                                NTP protocol version
      --collector.ntp.server-is-local
                                Certify that collector.ntp.server address is the same local host as this collector.
      --collector.ntp.ip-ttl=1  IP TTL to use while sending NTP query
      --collector.ntp.max-distance=3.46608s
                                Max accumulated distance to the root
      --collector.ntp.local-offset-tolerance=1ms
                                Offset between local clock and local ntpd time to tolerate
      --path.procfs="/proc"     procfs mountpoint.
      --path.sysfs="/sys"       sysfs mountpoint.
      --collector.qdisc.fixtures=""
                                test fixtures to use for qdisc collector end-to-end testing
      --collector.runit.servicedir="/etc/service"
                                Path to runit service directory.
      --collector.supervisord.url="http://localhost:9001/RPC2"
                                XML RPC endpoint.
      --collector.systemd.unit-whitelist=".+"
                                Regexp of systemd units to whitelist. Units must both match whitelist and not match blacklist to be included.
      --collector.systemd.unit-blacklist=".+\\.scope"
                                Regexp of systemd units to blacklist. Units must both match whitelist and not match blacklist to be included.
      --collector.systemd.private
                                Establish a private, direct connection to systemd without dbus.
      --collector.textfile.directory=""
                                Directory to read text files with metrics from.
      --collector.wifi.fixtures=""
                                test fixtures to use for wifi collector metrics
      --collector.arp           Enable the arp collector (default: enabled).
      --collector.bcache        Enable the bcache collector (default: enabled).
      --collector.bonding       Enable the bonding collector (default: disabled).
      --collector.buddyinfo     Enable the buddyinfo collector (default: disabled).
      --collector.conntrack     Enable the conntrack collector (default: enabled).
      --collector.cpu           Enable the cpu collector (default: enabled).
      --collector.diskstats     Enable the diskstats collector (default: enabled).
      --collector.drbd          Enable the drbd collector (default: disabled).
      --collector.edac          Enable the edac collector (default: enabled).
      --collector.entropy       Enable the entropy collector (default: enabled).
      --collector.filefd        Enable the filefd collector (default: enabled).
      --collector.filesystem    Enable the filesystem collector (default: enabled).
      --collector.gmond         Enable the gmond collector (default: disabled).
      --collector.hwmon         Enable the hwmon collector (default: enabled).
      --collector.infiniband    Enable the infiniband collector (default: enabled).
      --collector.interrupts    Enable the interrupts collector (default: disabled).
      --collector.ipvs          Enable the ipvs collector (default: enabled).
      --collector.ksmd          Enable the ksmd collector (default: disabled).
      --collector.loadavg       Enable the loadavg collector (default: enabled).
      --collector.logind        Enable the logind collector (default: disabled).
      --collector.mdadm         Enable the mdadm collector (default: enabled).
      --collector.megacli       Enable the megacli collector (default: disabled).
      --collector.meminfo       Enable the meminfo collector (default: enabled).
      --collector.meminfo_numa  Enable the meminfo_numa collector (default: disabled).
      --collector.mountstats    Enable the mountstats collector (default: disabled).
      --collector.netdev        Enable the netdev collector (default: enabled).
      --collector.netstat       Enable the netstat collector (default: enabled).
      --collector.nfs           Enable the nfs collector (default: disabled).
      --collector.ntp           Enable the ntp collector (default: disabled).
      --collector.qdisc         Enable the qdisc collector (default: disabled).
      --collector.runit         Enable the runit collector (default: disabled).
      --collector.sockstat      Enable the sockstat collector (default: enabled).
      --collector.stat          Enable the stat collector (default: enabled).
      --collector.supervisord   Enable the supervisord collector (default: disabled).
      --collector.systemd       Enable the systemd collector (default: disabled).
      --collector.tcpstat       Enable the tcpstat collector (default: disabled).
      --collector.textfile      Enable the textfile collector (default: enabled).
      --collector.time          Enable the time collector (default: enabled).
      --collector.uname         Enable the uname collector (default: enabled).
      --collector.vmstat        Enable the vmstat collector (default: enabled).
      --collector.wifi          Enable the wifi collector (default: enabled).
      --collector.xfs           Enable the xfs collector (default: enabled).
      --collector.zfs           Enable the zfs collector (default: enabled).
      --collector.timex         Enable the timex collector (default: enabled).
      --web.listen-address=":9100"
                                Address on which to expose metrics and web interface.
      --web.telemetry-path="/metrics"
                                Path under which to expose metrics.
      --log.level="info"        Only log messages with the given severity or above. Valid levels: [debug, info, warn, error, fatal]
      --log.format="logger:stderr"
                                Set the log target and format. Example: "logger:syslog?appname=bob&local=7" or "logger:stdout?json=true"
      --version                 Show application version.
```

### 关闭指定 collector 模块

- 使用 [prometheus.NewRegistry](https://github.com/prometheus/node_exporter/blob/master/node_exporter.go#L46) 判断并注册相关模块
  - 引用 [--no-collector](https://github.com/prometheus/node_exporter/blob/cf3edadcbbd7064a7671e2c1467de90298faef66/end-to-end-test.sh#L92) 参数

#### 关闭指定模块启动

```bash
[root@jeff bin]$ ./node_exporter --no-collector.arp --no-collector.bcache --no-collector.conntrack --no-collector.cpu --no-collector.diskstats --no-collector.edac --no-collector.entropy --no-collector.filefd --no-collector.filesystem  --no-collector.hwmon  --no-collector.infiniband  --no-collector.ipvs  --no-collector.loadavg --no-collector.mdadm  --no-collector.meminfo --no-collector.netdev --no-collector.netstat  --no-collector.sockstat  --no-collector.stat  --no-collector.textfile  --no-collector.time  --no-collector.timex  --no-collector.uname  --no-collector.vmstat  --no-collector.wifi  --no-collector.xfs  --no-collector.zfs
INFO[0000] Starting node_exporter (version=0.15.2, branch=HEAD, revision=98bc64930d34878b84a0f87dfe6e1a6da61e532d)  source="node_exporter.go:43"
INFO[0000] Build context (go=go1.9.2, user=root@d5c4792c921f, date=20171205-14:50:53)  source="node_exporter.go:44"
INFO[0000] Enabled collectors:                           source="node_exporter.go:50"
INFO[0000] Listening on :9100                            source="node_exporter.go:76"
```

#### 默认启动组件

```bash
[root@jeff bin]$ ./node_exporter
INFO[0000] Starting node_exporter (version=0.15.2, branch=HEAD, revision=98bc64930d34878b84a0f87dfe6e1a6da61e532d)  source="node_exporter.go:43"
INFO[0000] Build context (go=go1.9.2, user=root@d5c4792c921f, date=20171205-14:50:53)  source="node_exporter.go:44"
INFO[0000] No directory specified, see --collector.textfile.directory  source="textfile.go:57"
INFO[0000] Enabled collectors:                           source="node_exporter.go:50"
INFO[0000]  - hwmon                                      source="node_exporter.go:52"
INFO[0000]  - diskstats                                  source="node_exporter.go:52"
INFO[0000]  - infiniband                                 source="node_exporter.go:52"
INFO[0000]  - time                                       source="node_exporter.go:52"
INFO[0000]  - mdadm                                      source="node_exporter.go:52"
INFO[0000]  - netdev                                     source="node_exporter.go:52"
INFO[0000]  - stat                                       source="node_exporter.go:52"
INFO[0000]  - arp                                        source="node_exporter.go:52"
INFO[0000]  - vmstat                                     source="node_exporter.go:52"
INFO[0000]  - meminfo                                    source="node_exporter.go:52"
INFO[0000]  - xfs                                        source="node_exporter.go:52"
INFO[0000]  - conntrack                                  source="node_exporter.go:52"
INFO[0000]  - bcache                                     source="node_exporter.go:52"
INFO[0000]  - loadavg                                    source="node_exporter.go:52"
INFO[0000]  - textfile                                   source="node_exporter.go:52"
INFO[0000]  - sockstat                                   source="node_exporter.go:52"
INFO[0000]  - wifi                                       source="node_exporter.go:52"
INFO[0000]  - uname                                      source="node_exporter.go:52"
INFO[0000]  - zfs                                        source="node_exporter.go:52"
INFO[0000]  - ipvs                                       source="node_exporter.go:52"
INFO[0000]  - entropy                                    source="node_exporter.go:52"
INFO[0000]  - netstat                                    source="node_exporter.go:52"
INFO[0000]  - filesystem                                 source="node_exporter.go:52"
INFO[0000]  - filefd                                     source="node_exporter.go:52"
INFO[0000]  - edac                                       source="node_exporter.go:52"
INFO[0000]  - timex                                      source="node_exporter.go:52"
INFO[0000]  - cpu                                        source="node_exporter.go:52"
INFO[0000] Listening on :9100                            source="node_exporter.go:76"
```

## 0x02 采集分析主机指标

- 以下部分内容引用 [GPE 监控预警系统-node_exporter](https://my.oschina.net/neverforget/blog/1613272)

### 文本格式

在讨论 Exporter 之前，有必要先介绍一下 Prometheus 文本数据格式，因为一个 Exporter 本质上就是将收集的数据，转化为对应的文本格式，并提供 http 请求。

Exporter 收集的数据转化的文本内容以行 (\n) 为单位，空行将被忽略, 文本内容最后一行为空行。

### 注释

文本内容，如果以 # 开头通常表示注释

以 # HELP 开头表示 metric 帮助说明  
以 # TYPE 开头表示定义 metric 类型，包含 counter, gauge, histogram, summary, 和 untyped 类型  
其他表示一般注释，供阅读使用，将被 Prometheus 忽略  

### 采样数据

内容如果不以 # 开头，表示采样数据。它通常紧挨着类型定义行，满足以下格式([primaryExpr parses a primary expression.](https://github.com/prometheus/prometheus/blob/master/promql/parse.go))：

```go
metric_name [
  "{" label_name "=" `"` label_value `"` { "," label_name "=" `"` label_value `"` } [ "," ] "}"
] value [ timestamp ]
```

- 使用 `curl http://127.0.0.1:9100/metrics` 获取信息如下(部分数据)：
  - 需要特别注意的是，假设采样数据 metric 叫做 x, 如果 x 是 histogram 或 summary 类型必需满足以下条件：
    - 采样数据的总和应表示为 x_sum
    - 采样数据的总量应表示为 x_count
    - summary 类型的采样数据的 quantile 应表示为 x{quantile="y"}
    - histogram 类型的采样分区统计数据将表示为 x_bucket{le="y"}
    - histogram 类型的采样必须包含 x_bucket{le="+Inf"}, 它的值等于 x_count 的值
    - summary 和 historam 中 quantile 和 le 必需按从小到大顺序排列


```bash
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 8
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.9.2"} 1
# HELP go_memstats_alloc_bytes Number of bytes allocated and still in use.
# TYPE go_memstats_alloc_bytes gauge
go_memstats_alloc_bytes 1.229584e+06
# HELP go_memstats_alloc_bytes_total Total number of bytes allocated, even if freed.
# TYPE go_memstats_alloc_bytes_total counter
go_memstats_alloc_bytes_total 1.229584e+06
# HELP go_memstats_buck_hash_sys_bytes Number of bytes used by the profiling bucket hash table.
# TYPE go_memstats_buck_hash_sys_bytes gauge
go_memstats_buck_hash_sys_bytes 1.443396e+06
# HELP go_memstats_frees_total Total number of frees.
# TYPE go_memstats_frees_total counter
go_memstats_frees_total 455
# HELP go_memstats_gc_cpu_fraction The fraction of this program's available CPU time used by the GC since the program started.
# TYPE go_memstats_gc_cpu_fraction gauge
go_memstats_gc_cpu_fraction 0
# HELP go_memstats_gc_sys_bytes Number of bytes used for garbage collection system metadata.
# TYPE go_memstats_gc_sys_bytes gauge
go_memstats_gc_sys_bytes 169984
# HELP go_memstats_heap_alloc_bytes Number of heap bytes allocated and still in use.
# TYPE go_memstats_heap_alloc_bytes gauge
go_memstats_heap_alloc_bytes 1.229584e+06
# HELP go_memstats_heap_idle_bytes Number of heap bytes waiting to be used.
# TYPE go_memstats_heap_idle_bytes gauge
go_memstats_heap_idle_bytes 622592
# HELP go_memstats_heap_inuse_bytes Number of heap bytes that are in use.
# TYPE go_memstats_heap_inuse_bytes gauge
go_memstats_heap_inuse_bytes 2.12992e+06
# HELP go_memstats_heap_objects Number of allocated objects.
# TYPE go_memstats_heap_objects gauge
go_memstats_heap_objects 8912
# HELP go_memstats_heap_released_bytes Number of heap bytes released to OS.
# TYPE go_memstats_heap_released_bytes gauge
go_memstats_heap_released_bytes 0
# HELP go_memstats_heap_sys_bytes Number of heap bytes obtained from system.
# TYPE go_memstats_heap_sys_bytes gauge
go_memstats_heap_sys_bytes 2.752512e+06
# HELP go_memstats_last_gc_time_seconds Number of seconds since 1970 of last garbage collection.
# TYPE go_memstats_last_gc_time_seconds gauge
go_memstats_last_gc_time_seconds 0
# HELP go_memstats_lookups_total Total number of pointer lookups.
# TYPE go_memstats_lookups_total counter
go_memstats_lookups_total 17
# HELP go_memstats_mallocs_total Total number of mallocs.
# TYPE go_memstats_mallocs_total counter
go_memstats_mallocs_total 9367
# HELP go_memstats_mcache_inuse_bytes Number of bytes in use by mcache structures.
# TYPE go_memstats_mcache_inuse_bytes gauge
go_memstats_mcache_inuse_bytes 6944
# HELP go_memstats_mcache_sys_bytes Number of bytes used for mcache structures obtained from system.
# TYPE go_memstats_mcache_sys_bytes gauge
go_memstats_mcache_sys_bytes 16384
# HELP go_memstats_mspan_inuse_bytes Number of bytes in use by mspan structures.
# TYPE go_memstats_mspan_inuse_bytes gauge
go_memstats_mspan_inuse_bytes 31616
# HELP go_memstats_mspan_sys_bytes Number of bytes used for mspan structures obtained from system.
# TYPE go_memstats_mspan_sys_bytes gauge
go_memstats_mspan_sys_bytes 32768
# HELP go_memstats_next_gc_bytes Number of heap bytes when next garbage collection will take place.
# TYPE go_memstats_next_gc_bytes gauge
go_memstats_next_gc_bytes 4.473924e+06
# HELP go_memstats_other_sys_bytes Number of bytes used for other system allocations.
# TYPE go_memstats_other_sys_bytes gauge
go_memstats_other_sys_bytes 797364
# HELP go_memstats_stack_inuse_bytes Number of bytes in use by the stack allocator.
# TYPE go_memstats_stack_inuse_bytes gauge
go_memstats_stack_inuse_bytes 393216
# HELP go_memstats_stack_sys_bytes Number of bytes obtained from system for stack allocator.
# TYPE go_memstats_stack_sys_bytes gauge
go_memstats_stack_sys_bytes 393216
# HELP go_memstats_sys_bytes Number of bytes obtained from system.
# TYPE go_memstats_sys_bytes gauge
go_memstats_sys_bytes 5.605624e+06
# HELP go_threads Number of OS threads created.
# TYPE go_threads gauge
go_threads 7
# HELP http_request_duration_microseconds The HTTP request latencies in microseconds.
# TYPE http_request_duration_microseconds summary
http_request_duration_microseconds{handler="prometheus",quantile="0.5"} NaN
http_request_duration_microseconds{handler="prometheus",quantile="0.9"} NaN
http_request_duration_microseconds{handler="prometheus",quantile="0.99"} NaN
http_request_duration_microseconds_sum{handler="prometheus"} 0
http_request_duration_microseconds_count{handler="prometheus"} 0
# HELP http_request_size_bytes The HTTP request sizes in bytes.
# TYPE http_request_size_bytes summary
http_request_size_bytes{handler="prometheus",quantile="0.5"} NaN
http_request_size_bytes{handler="prometheus",quantile="0.9"} NaN
http_request_size_bytes{handler="prometheus",quantile="0.99"} NaN
http_request_size_bytes_sum{handler="prometheus"} 0
http_request_size_bytes_count{handler="prometheus"} 0
# HELP http_response_size_bytes The HTTP response sizes in bytes.
# TYPE http_response_size_bytes summary
http_response_size_bytes{handler="prometheus",quantile="0.5"} NaN
http_response_size_bytes{handler="prometheus",quantile="0.9"} NaN
http_response_size_bytes{handler="prometheus",quantile="0.99"} NaN
http_response_size_bytes_sum{handler="prometheus"} 0
http_response_size_bytes_count{handler="prometheus"} 0
# HELP node_exporter_build_info A metric with a constant '1' value labeled by version, revision, branch, and goversion from which node_exporter was built.
# TYPE node_exporter_build_info gauge
node_exporter_build_info{branch="HEAD",goversion="go1.9.2",revision="98bc64930d34878b84a0f87dfe6e1a6da61e532d",version="0.15.2"} 1
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 0.01
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1e+06
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 7
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 6.127616e+06
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.52403869128e+09
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 2.0353024e+08
```
