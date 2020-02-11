---
title: 监控架构 - 专业打小报告好多年
date: 2017-06-04T00:00:00+08:00
lastmod: 2020-01-20T00:00:00+08:00
categories:
  - Monitoring
tags:
  - Deploy
  - AlertManager
  - Proemtheus
---
## 骑士

- [AlertManager](https://prometheus.io/docs/alerting/configuration/) 配置以及工具使用说明
- [AlertManager](https://github.com/pingcap/tidb-ansible/blob/master/conf/alertmanager.yml) 配置文件模板

## 交叉

- 在 Prometheus 配置文件更新以下代码块(prometheus 2.0 +)，修改完成后重启后生效
  - Prometheus 配置文件地址 `{{deploy_dir}}/conf/prometheus.yml`

  ```yaml
  alerting:
    alertmanagers:
    - static_configs:
      - targets:
        - '172.16.10.65:39093' # AlertManager 地址与端口
  ```

## 模范

- 告警发送
  - Slack：API 通过 [Slack-API](https://api.slack.com/incoming-webhooks) 获取
    - 需要 Slack 账号
  - E-mail：通过邮件方式发送告警，选用 db-alert-email 接收器
    - 需要 SMTP 服务器信息
  - WebHook：通过 SMS 或者 syslog 方式发送告警选用 webhook-pulgin 接收器
    - 需要提供 syslog binary 工具接收
    - 发送到 syslog binary 中的告警等级为 port ，由 route 中的 `level: port` 参数控制。
- 告警场景
  - 注意版本问题 / 发送次数
  - 同一个集群环境，向多方同时发送告警
    - 多个 slack api + 多个邮箱地址 + SMS / 语音告警 = 告警平台 (伪)
- 小问题
  - 邮件模板不美观
    - 邮件模板内连接不准确
  - 邮件单独告警，而非一封邮件
  - alertmanager 组件如果挂掉，那么就没办法发告警了，前面所有链路中断

```yaml
global:
  # Slack API 信息
  slack_api_url: 'https://hooks.slack.com/services/T04A1PYPM/B2EJ1BSBG/12313424342342344'

  # Email SMTP 服务器信息
  smtp_smarthost: 'smtp.ym.126.com:25'
  smtp_from: 'jump@tidb.cc'
  smtp_auth_username: 'jump@tidb.cc'
  smtp_auth_password: 'password'
  smtp_auth_secret: null
  smtp_auth_identity: ""
  smtp_require_tls: true  # 邮箱服务是否开启 TLS

route:                          # 声明 根路由节点
  receiver: "alert-default"     # 如果没有接收器规则匹配到这条告警，默认发送到 `alert-default`，该 alert-default 必须存在，否则报错退出
  group_by: ['env','instance','type','group','job']   # 按照该 labels 分组用于分组压缩短时间内产生的大量告警
  group_wait:      30s
  group_interval:  3m
  repeat_interval: 3m
  routes:                      # 声明 子路由
  - match:                     # 规则匹配
    env: test-cluster          # 集群环境名字，路由接收到后分配给接收器，接收器读取规则后发送至 slack_config
    level: waring              # 匹配 alert.rule 中 level: waring labels 属性，匹配成功后发送到相应接收器
  receiver: alert-slack        # receiver 关键词，全局唯一，不能重复
  continue: true               # 默认告警匹配成功第一个 receivers 会退出匹配，开启 continue 参数后会继续匹配 receivers 列表，直到再无 receivers 时或者下一个 receivers 中 continue fasle 的时候才会退出 (continue default false)
  - match_re:                  # 支持正则匹配
    env: test-cluster          # 客户集群环境名字，路由接收到后分配给接收器，接收器读取规则后发送至 slack_config
  receiver: alert-email-global # 指定接收器唯一名字
  continue: true
  - match_re:                  # 支持正则匹配
    level: emergency           # 匹配 alert.rule 中 level: emergency labels 属性，匹配成功后发送到相应接收器
    env: test-cluster          # 客户集群环境名字，路由接收到后分配给接收器，接收器读取规则后发送至 slack_config
    severity: ^([a-zA-Z0-9]*)$ # 正则匹配
  receiver: alert-webhook      # 指定接收器唯一名字
  continue: true


receivers:                        # 接收器根

# Slack template

- name: 'alert-default'           # 接收器唯一名字，如有重复，告警停用。与 `route` 节点规则关联使用。
  slack_configs:
  - channel: '#alerts-for-test'   # 指定发送至 slack channel
    username: 'office-alert'        # 指定发送者 username
    icon_emoji: ':pingcap:'         # 指定发送者头像, 可在 slack 表情包获取 emoji 信息
    title:   '{{.CommonLabels.alertname}}'
    text:    'in {{.CommonLabels.env}}:   {{ .GroupLabels.alertname }}  {{ .CommonAnnotations.description }}    http://office.tidb.cc/alerts'

- name: 'alert-slack-user'                  # 接收器唯一名字，如有重复，告警停用。与 `route` 节点规则关联使用。
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/T04AQPYPM/B2EJ1BSBG/zczczczczcdsadafsvdvsf'
    channel: '@user'                          # 指定发送至 slack 用户
    username: '{{.CommonLabels.env}}-alert'   # 指定发送者 username (昵称) , 此处格式为 ` 集群名称 - alert`
    icon_emoji: ':{{.CommonLabels.env}}:'     # 指定发送者头像, 此处格式为 `: 集群名称:`, 提前在 slack 定义该图片
    title:   '{{.CommonLabels.alertname}}'
    text:    'in {{.CommonLabels.env}}:   {{ .GroupLabels.alertname }}  {{ .CommonAnnotations.description }}    http://office.tidb.cc/alerts'

# Email template

- name: 'alert-email-global'
  email_configs:
  - send_resolved: true         # 告警回执，监控下由 Alert 返回为 OK 状态时，会发送一条 OK 状态的回执
    to: 'alert-global@tidb.cc'  # 接收邮箱地址
    from: 'jump@tidb.cc'        # 发送邮箱地址

- name: alert-email-user
  email_configs:
  - send_resolved: false
    to: alert-user@tidb.cc
    from: jump@tidb.cc
    smarthost: smtp.ym.126.com:25   # 单独设置 smtp 信息
    auth_username: jump@tidb.cc
    auth_password: <secret>
    auth_secret: null
    auth_identity: ""
    headers:
      From: jump@tidb.cc
      Subject: '{{template"email.default.subject".}}'
      To: alert-user@tidb.cc
    html: '{{template"email.default.html".}}'
    require_tls: true         # 默认开启 TLS 协议

# webhook-pulgin

- name: 'webhook-pulgin'
  webhook_configs:
  - send_resolved: true
    url: 'http://172.16.10.65:28082/v1/alertmanager'  # 该 URL 是一个小工具，用来解析 alertmanager json 告警数据，然后发往目的地
```

### 管理

- 通过 `http://IP:9093/` 打开网页，可通过网页管理告警
  - 通过页面可以 silence 一条正在持续发送的告警

### 维护

- 告警测试
  - `curl -H "Content-Type: application/json" -d ‘[{"labels":{"alertname":"TestAlert1"}}]’ http://180.76.154.185:9093/api/v1/alerts`
- 启动 停止管理
  - 自动化部署的 AlertManager，启动停止脚本位于 `{{deploy_dir}}/scripts` 目录中
  - 手动部署的，参考 [通过 systemd 守护 syncer 进程启动](../Docs/180323-Systemd-Syncer) 文档

## 友人

### 组件部署

- 使用 2018 年 3 月 22 日以后的 [TiDB-Ansible](https://github.com/pingcap/tidb-ansible/blob/master/deploy.yml) 可以自动部署 AlertManager 组件
- 也可按照参考 [通过 systemd 守护 syncer 进程启动](/post/20180323-Systemd-Syncer/ "ap.tidb.cc syncer systemd") 方案手动部署 AlertManager 组件
  - AlertManager 下载 [传送门](https://prometheus.io/download/) ，AlertManager 项目库 [传送门](https://github.com/prometheus/alertmanager)
