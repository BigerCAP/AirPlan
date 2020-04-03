---
title: TiDB - Binlog 数据同步架构测试
date: 2017-10-08T00:00:00+08:00
lastmod: 2017-11-20T00:00:00+08:00
categories:
  - software
tags:
  - Linux
  - testing
  - deploy
  - K8S
  - kubernetes
---
## 0x00 缘起

> 无它，就是学习学习，然后装逼用

![无它，唯手熟尔](/global/helpertools.jpg)

## 0x01 准备

> K8S 有一些要求限制，不按照处理是无法启动滴  
> 以下内容随手复制的，非原创

- 关闭 swap

  ```bash
  sed -i 7,9s/0/1/g /usr/lib/sysctl.d/00-system.conf

  modprobe br_netfilter

  echo 'vm.swappiness = 0' >> /usr/lib/sysctl.d/00-system.conf

  swapoff -a

  sysctl -p /usr/lib/sysctl.d/00-system.conf
  ```

- 关闭防火墙

  ```bash
  # 关闭 selinux
  systemctl status firewalld
  systemctl status iptables
  ```

## 0x02 Docker-ce

> 之前安装 docker 的时候都是 yum install docker，当时应该安装是 docker-io ，这是最早的版本。docker 估计是为了商业化，后来改造成了 docker-ce 和 docker-ee 两个版本
> Docker 社区版（CE）：为了开发人员或小团队创建基于容器的应用, 与团队成员分享和自动化的开发管道。docker-ce 提供了简单的安装和快速的安装，以便可以立即开始开发。docker-ce 集成和优化，基础设施。（免费）
> Docker 企业版（EE）：专为企业的发展和 IT 团队建立谁。docker-ee 为企业提供最安全的容器平台，以应用为中心的平台。（付费）

- 修改国内 aliyun yum 源

  ```bash
  wget -O  /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  # 获取 aliyun yum 源

  yum install docker-ce-18.09.6*
  # TiDB operator 需要指定 docker 版本
  ```

- 查看 yum 是否安装成功

  ```bash
  [root@tmp49h ]# rpm -qa | grep docker
  docker-ce-cli-19.03.8-3.el7.x86_64
  docker-ce-18.09.6-3.el7.x86_64
  ```

- 选配 / 设置 docker-registry-mirrors 为国内地址

  ```bash
  tee /etc/docker/daemon.json <<-'EOF'
  {
  "registry-mirrors": ["https://kzflpq4b.mirror.aliyuncs.com"]
  }
  EOF
  ```

- 简单操作命令

  ```shell
  systemctl enable docker
  # 添加开机启动

  systemctl start docker
  # 使用 systemd 启动 docker

  systemctl status docker
  # 查看 doker 运行状态
  ```

- 查看 docker 版本信息

  ```bash
  [root@tmp49h ]# docker version
  Client: Docker Engine - Community
  Version:           19.03.8
  API version:       1.39 (downgraded from 1.40)
  Go version:        go1.12.17
  Git commit:        afacb8b
  Built:             Wed Mar 11 01:27:04 2020
  OS/Arch:           linux/amd64
  Experimental:      false

  Server: Docker Engine - Community
  Engine:
    Version:          18.09.6
    API version:      1.39 (minimum version 1.12)
    Go version:       go1.10.8
    Git commit:       481bc77
    Built:            Sat May  4 02:02:43 2019
    OS/Arch:          linux/amd64
    Experimental:     false
  ```

- 简单 docker 命令

  ```bash
  docker images

  docker pull xxxx:tag

  docker ps  / docker ps -a

  docker logs name

  docker rm -f name

  ```

## 0x03 K8S

> 安装简单，关键是如何规避科学上网

- 安装 kubectl
  - 默认读取 $HOME/.kube/config 文件来操作 K8S 集群

  ```shell
  curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.12.8/bin/linux/amd64/kubectl

  chmod 755 kubectl

  mv kubectl /use/bin/
  ```

- 安装 helm
  - 注意：脚本中把 helm 默认放在了 `/usr/local/bin/`，如果脚本执行失败记得检查下
  - PS ：可以把脚本下载到本地 helm.sh ，执行 sh -x helm.sh ，可以看到 debug 记录
  - PPS：该脚本多次执行会报错，自身不带清理回收

  ```bash
  curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
  ```

- 安装 kube 套件
  - 如果安装出现 gpg 无效，诡异的是执行下 yum makecache 就好了

  ```bash
  tee /etc/yum.repos.d/kubernets.repo << EOF
  [Kubernetes]
  name=kubernetes 
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
  gpgcheck=1
  gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg,https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg,https://mirrors.aliyun.com/kubernetes/yum/doc/apt-key.gpg
  EOF
  # 添加 yum 源

  yum makecache
  #创建yum元数据

  yum install -y kubelet kubeadm kubectl
  # 可选直接安装 kubectl ，也可以手动安装
  ```

### Init

> 这里有多种方案，均需要主机可以自由联网下载数据，比如：
>
> - Docker 代理
> - http / socket 代理
> - 指定 kubeadm 源
> - iptable 转发 k8s.grc.io 域名到其他国内镜像源域名 / 搭建 DNS 做 cname 转发也可以
> - 私自搭建 docker hub / ~~ 奇技淫巧~~ 可以用于内网机器搭建 k8s 环境
>
> 我是懒人，选择指定 kubeadm 源。会用到一个 `kubeadm config print init-default` 的命令

- 执行 kubeadm init 前建议先用 `kubeadm config images pull` 测试下是否可以获取镜像
  - 以下是失败案例，提示链接不到 `https://k8s.gcr.io/v2/` 仓库

  ```bash
  [root@tmp49h ]# kubeadm config images pull
  W0402 10:19:46.190059    3816 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]

  failed to pull image "k8s.gcr.io/kube-apiserver:v1.18.0": output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
  , error: exit status 1
  To see the stack trace of this error execute with --v=5 or higher
  ```

- 那么接来就要使用备用方案处理
  - 此处使用的修改数据源
  - 具体详情见 [FAQ case-1](#case-1)
  - 然后执行 `kubeadm init --config ./config` 命令

- 安装记录
  - 失败的过程 [FAQ case-4](#case-4)
  - 成功的过程 [FAQ case-5](#case-5)

- 记得记录安装成功的输出内容，如下
  - 尤其是最底部的 kubeadm join 信息，该 token 时效 24 小时
  - 通过 kubeadm token list 可以看到信息

  ```log
  Your Kubernetes control-plane has initialized successfully!

  To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

  You should now deploy a pod network to the cluster.
  Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

  Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join 10.0.1.5:6443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:9a4b14630871dd9c8c6cd19787a305ef1188337375682085459d5d5bcd2d2b52
  ```

  ```bash
  [root@tmp49h ~]# kubeadm token list
  TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
  abcdef.0123456789abcdef   6h          2020-04-03T12:05:16Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
  ```

### Flannel

- k8s 网络插件 flannel 安装，安装没遇见问题，
  - 该步骤需要下载镜像，所以网络上可能会耗时

  ```bash
  [root@tmp49h ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  podsecuritypolicy.policy/psp.flannel.unprivileged created
  clusterrole.rbac.authorization.k8s.io/flannel created
  clusterrolebinding.rbac.authorization.k8s.io/flannel created
  serviceaccount/flannel created
  configmap/kube-flannel-cfg created
  daemonset.apps/kube-flannel-ds-amd64 created
  daemonset.apps/kube-flannel-ds-arm64 created
  daemonset.apps/kube-flannel-ds-arm created
  daemonset.apps/kube-flannel-ds-ppc64le created
  daemonset.apps/kube-flannel-ds-s390x created
  ```

### Testing

> 测试是否可用

- 执行几个查询命令
  - `kubectl get node`
  - `kubectl get namespace`
  - `kubectl get pods --namespace=kube-system -o wide`

  ```bash
  [root@tmp49h ~]# kubectl get node
  NAME     STATUS   ROLES    AGE     VERSION
  tmp49h   Ready    master   7m21s   v1.18.0

  [root@tmp49h ~]#  kubectl get namespace
  NAME              STATUS   AGE
  default           Active   7m26s
  kube-node-lease   Active   7m28s
  kube-public       Active   7m28s
  kube-system       Active   7m28s

  [root@tmp49h ~]# kubectl get pods --namespace=kube-system -o wide
  NAME                             READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
  coredns-6fbfdf9657-54gp9         1/1     Running   0          7m29s   192.240.0.2   tmp49h   <none>           <none>
  coredns-6fbfdf9657-hwpb2         1/1     Running   0          7m29s   192.240.0.3   tmp49h   <none>           <none>
  etcd-tmp49h                      1/1     Running   0          7m43s   10.0.1.5      tmp49h   <none>           <none>
  kube-apiserver-tmp49h            1/1     Running   0          7m43s   10.0.1.5      tmp49h   <none>           <none>
  kube-controller-manager-tmp49h   1/1     Running   0          7m43s   10.0.1.5      tmp49h   <none>           <none>
  kube-flannel-ds-amd64-9bs72      1/1     Running   0          55s     10.0.1.5      tmp49h   <none>           <none>
  kube-proxy-zdln9                 1/1     Running   0          7m29s   10.0.1.5      tmp49h   <none>           <none>
  kube-scheduler-tmp49h            1/1     Running   0          7m43s   10.0.1.5      tmp49h   <none>           <none>
  ```

## 0x04 Nodes

- 准备环境 / 按照 [0x01](#0x01-%e5%87%86%e5%a4%87) 做一遍
  - 重点检查下有木有 ip_vs_sh、ip_vs_wrr、ip_vs_rr、ip_vs 的这几个模块，木有的话尝试用 modprobe 加载

  ```bash
  [root@tmp49h ~]# lsmod |grep ip_
  ip_vs_sh               12688  0
  ip_vs_wrr              12697  0
  ip_vs_rr               12600  0
  ip_vs                 145497  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
  nf_conntrack          139224  7 ip_vs,nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4
  ip_tables              27126  4 iptable_security,iptable_filter,iptable_mangle,iptable_nat
  libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
  ```

- **按照 [0x02](#0x02-docker-ce) 安装 docker-ce 服务**

- 接下来是获取必备镜像
  - `gcr.azk8s.cn/google_containers/kube-proxy:v1.18.0`
  - `quay.io/coreos/flannel:v0.12.0-amd64`
  - `gcr.azk8s.cn/google_containers/pause:3.2`

- 下载镜像方式与 k8s init 方式类似
  - `kubeadm config print init-defaults` 修改源，详情见[FAQ case-1](#case-1)
  - `kubeadm --config ./config config images pull` 下载镜像
  - `docker pull quay.io/coreos/flannel:v0.12.0-amd64` ，**这个要单独下**

- 检查 docker 镜像

  ```bash
  [root@atman-50 atman]# docker images
  REPOSITORY                                               TAG                 IMAGE ID            CREATED             SIZE
  gcr.azk8s.cn/google_containers/kube-proxy                v1.18.0             43940c34f24f        7 days ago          117MB
  gcr.azk8s.cn/google_containers/kube-apiserver            v1.18.0             74060cea7f70        7 days ago          173MB
  gcr.azk8s.cn/google_containers/kube-controller-manager   v1.18.0             d3e55153f52f        7 days ago          162MB
  gcr.azk8s.cn/google_containers/kube-scheduler            v1.18.0             a31f78c7c8ce        7 days ago          95.3MB
  quay.io/coreos/flannel                                   v0.12.0-amd64       4e9f801d2217        2 weeks ago         52.8MB
  gcr.azk8s.cn/google_containers/pause                     3.2                 80d28bedfe5d        6 weeks ago         683kB
  gcr.azk8s.cn/google_containers/coredns                   1.6.7               67da37a9a360        2 months ago        43.8MB
  gcr.azk8s.cn/google_containers/etcd                      3.4.3-0             303ce5db0e90        5 months ago        288MB
  ```

### Join

- 执行 kubeadm join 命令，token 过期后见 [FAQ case-6](#case-6)

  ```bash
  [root@atman-50 atman]# kubeadm join 10.0.1.5:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:9a4b14630871dd9c8c6cd19787a305ef1188337375682085459d5d5bcd2d2b52
  W0402 12:43:22.333783    3043 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
  [preflight] Running pre-flight checks
    [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
  [preflight] Reading configuration from the cluster...
  [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
  [kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
  [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
  [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
  [kubelet-start] Starting the kubelet
  [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

  This node has joined the cluster:
  * Certificate signing request was sent to apiserver and a response was received.
  * The Kubelet was informed of the new secure connection details.

  Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
  ```

### Remove

- 删除 node

  ```bash
  kubectl drain tmp50h --delete-local-data --force --ignore-daemonsets
  # 设置被目标节点为维护模式【atman-50 是节点名称】，通过 kubectl get node 获取

  kubectl delete node atman-50
  # 删除节点
  ```

## 0x05 FAQ

> 记录下测试期间遇见的大大小小的各种坑

### case 0

> 机器配置太低！

- 心痛，安装 k8s 至少需要 2 个 cpu

  ```log
  W0402 10:03:26.595866    3844 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
  [init] Using Kubernetes version: v1.18.0
  [preflight] Running pre-flight checks
    [WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
    [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
  error execution phase preflight: [preflight] Some fatal errors occurred:
    [ERROR NumCPU]: the number of available CPUs 1 is less than the required 2
  [preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
  To see the stack trace of this error execute with --v=5 or higher
  ```

### case 1

> kubeadm 是个神器  
> 命令行参数不能覆盖配置文件参数，由于没有提前阅读文档，后面有多个坑

- kubeadm 配置文件预览
  - 具体可以看 k8s 官网 [kubeadm-init](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/)【中文内容】

  ```bash
  [root@tmp49h ~]# kubeadm config print init-defaults
  W0403 05:19:28.449746   23654 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]

  apiVersion: kubeadm.k8s.io/v1beta2
  bootstrapTokens:
  - groups:
    - system:bootstrappers:kubeadm:default-node-token
    token: abcdef.0123456789abcdef  // token 密钥，注意修改
    ttl: 24h0m0s
    usages:
    - signing
    - authentication
  kind: InitConfiguration
  localAPIEndpoint:
    advertiseAddress: 1.2.3.4  // 如果是多个 master 节点，填写 master LB IP，init 的时候需要这个 IP
    bindPort: 6443
  nodeRegistration:
    criSocket: /var/run/dockershim.sock
    name: tmp49h
    taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
  ---
  apiServer:
    timeoutForControlPlane: 4m0s
  apiVersion: kubeadm.k8s.io/v1beta2
  certificatesDir: /etc/kubernetes/pki
  clusterName: kubernetes
  controllerManager: {}
  dns:
    type: CoreDNS
  etcd:
    local:
      dataDir: /var/lib/etcd
  imageRepository: k8s.gcr.io  // kubeadm 使用的源，可以修改为 gcr.azk8s.cn/google_containers
  kind: ClusterConfiguration
  kubernetesVersion: v1.18.0
  networking:
    dnsDomain: cluster.local
    podSubnet: 192.240.0.0/16    // POD 预分配网段地址不可以与 node 主机地址重复
    serviceSubnet: 192.96.0.0/12 // 不知道干啥的
  scheduler: {}
  ```

- 命令行参数与配置文件内容冲突，无法覆盖

  ```bahs
  [root@tmp49h ~]# kubeadm --config .kube/config init --kubernetes-version=stable-1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12
  can not mix '--config' with arguments [kubernetes-version pod-network-cidr service-cidr]
  To see the stack trace of this error execute with --v=5 or higher
  ```

- advertiseAddress 参数配置错误
  - 配置文件中指定 api.advertiseAddress 为 vip 使客户端通过 vip 来访问，指定 etcd.endpoints 使用我们已经搭建好的 etcd 集群存储集群数据，指定 imageRepository 使用我们自己的镜像仓库
  - `https://www.niuhp.com/k8s/init-master1.html`

  ```bash
  I0402 11:54:28.334273   45830 round_trippers.go:443] GET https://1.2.3.4:6443/healthz?timeout=10s  in 10000 milliseconds
  I0402 11:54:38.835316   45830 round_trippers.go:443] GET https://1.2.3.4:6443/healthz?timeout=10s  in 10000 milliseconds
  I0402 11:54:49.334834   45830 round_trippers.go:443] GET https://1.2.3.4:6443/healthz?timeout=10s  in 10000 milliseconds
  [kubelet-check] Initial timeout of 40s passed.
  I0402 11:54:59.834973   45830 round_trippers.go:443] GET https://1.2.3.4:6443/healthz?timeout=10s  in 10000 milliseconds
  ```

### case 2

- 因为频繁修改 config 文件，且 advertiseAddress 修改后，kubeadm 并不会覆盖已有的残留信息( k8s 在 /etc/kubernetes 下面生成的文件，提示文件内容与最新的 config 不一致)
  - 建议使用 kubeadm reset 重制

  ```bash
  a kubeconfig file "/etc/kubernetes/kubelet.conf" exists already but has got the wrong API Server URL
  k8s.io/kubernetes/cmd/kubeadm/app/phases/kubeconfig.validateKubeConfig

  I0402 11:56:34.170449   46888 loader.go:375] Config loaded from file:  /etc/kubernetes/admin.conf
  a kubeconfig file "/etc/kubernetes/admin.conf" exists already but has got the wrong API Server URL

  I0402 12:00:15.020692   49830 loader.go:375] Config loaded from file:  /etc/kubernetes/controller-manager.conf
  failed to find CurrentContext in Contexts of the kubeconfig file /etc/kubernetes/controller-manager.conf
  k8s.io/kubernetes/cmd/kubeadm/app/phases/kubeconfig.validateKubeConfig
  ```

- 执行 kubeadm reset 重制环境信息
  - 提示重制了所有信息

  ```bash
  [root@tmp49h atman]# kubeadm reset
  [reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
  [reset] Are you sure you want to proceed? [y/N]: y
  [preflight] Running pre-flight checks
  W0402 10:42:29.820727    7017 removeetcdmember.go:79] [reset] No kubeadm config, using etcd pod spec to get data directory
  [reset] No etcd config found. Assuming external etcd
  [reset] Please, manually reset etcd to prevent further issues
  [reset] Stopping the kubelet service
  [reset] Unmounting mounted directories in "/var/lib/kubelet"
  W0402 10:42:29.836562    7017 cleanupnode.go:99] [reset] Failed to evaluate the "/var/lib/kubelet" directory. Skipping its unmount and cleanup: lstat /var/lib/kubelet: no such file or directory
  [reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
  [reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
  [reset] Deleting contents of stateful directories: [/var/lib/dockershim /var/run/kubernetes /var/lib/cni]

  The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

  The reset process does not reset or clean up iptables rules or IPVS tables.
  If you wish to reset iptables, you must do so manually by using the "iptables" command.

  If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
  to reset your system's IPVS tables.

  The reset process does not clean your kubeconfig files and you must remove them manually.
  Please, check the contents of the $HOME/.kube/config file.
  ```

### case 3

- 安装成功后，在 master 节点直接执行了 kubeadm join，提示信息被占用
  - 执行了 kubectl get node 才理解 master 本身就算一个节点，并且已经默认加入到了集群
  - 所以加开了一台虚拟机测试 kubeadm join

  ```shell
  [root@tmp49h ~]# kubeadm join 10.0.1.5:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:9a4b14630871dd9c8c6cd19787a305ef1188337375682085459d5d5bcd2d2b52
  W0402 12:14:46.722266   54428 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
  [preflight] Running pre-flight checks
    [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
  error execution phase preflight: [preflight] Some fatal errors occurred:
    [ERROR DirAvailable--etc-kubernetes-manifests]: /etc/kubernetes/manifests is not empty
    [ERROR FileAvailable--etc-kubernetes-kubelet.conf]: /etc/kubernetes/kubelet.conf already exists
    [ERROR Port-10250]: Port 10250 is in use
    [ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
  [preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
  To see the stack trace of this error execute with --v=5 or higher
  ```

### case 4

- 贴个安装失败的日志
  - 该过程使用默认日志等级，有了错误啥也看不出来

  ```log
  [root@tmp49h ~]# kubeadm --config .kube/config init 
  W0402 11:30:17.970803   38671 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
  [init] Using Kubernetes version: v1.18.0
  [preflight] Running pre-flight checks
    [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
  [preflight] Pulling images required for setting up a Kubernetes cluster
  [preflight] This might take a minute or two, depending on the speed of your internet connection
  [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
  [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
  [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
  [kubelet-start] Starting the kubelet
  [certs] Using certificateDir folder "/etc/kubernetes/pki"
  [certs] Generating "ca" certificate and key
  [certs] Generating "apiserver" certificate and key
  [certs] apiserver serving cert is signed for DNS names [tmp49h kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 1.2.3.4]
  [certs] Generating "apiserver-kubelet-client" certificate and key
  [certs] Generating "front-proxy-ca" certificate and key
  [certs] Generating "front-proxy-client" certificate and key
  [certs] Generating "etcd/ca" certificate and key
  [certs] Generating "etcd/server" certificate and key
  [certs] etcd/server serving cert is signed for DNS names [tmp49h localhost] and IPs [1.2.3.4 127.0.0.1 ::1]
  [certs] Generating "etcd/peer" certificate and key
  [certs] etcd/peer serving cert is signed for DNS names [tmp49h localhost] and IPs [1.2.3.4 127.0.0.1 ::1]
  [certs] Generating "etcd/healthcheck-client" certificate and key
  [certs] Generating "apiserver-etcd-client" certificate and key
  [certs] Generating "sa" key and public key
  [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
  [kubeconfig] Writing "admin.conf" kubeconfig file
  [kubeconfig] Writing "kubelet.conf" kubeconfig file
  [kubeconfig] Writing "controller-manager.conf" kubeconfig file
  [kubeconfig] Writing "scheduler.conf" kubeconfig file
  [control-plane] Using manifest folder "/etc/kubernetes/manifests"
  [control-plane] Creating static Pod manifest for "kube-apiserver"
  [control-plane] Creating static Pod manifest for "kube-controller-manager"
  W0402 11:30:26.193263   38671 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
  [control-plane] Creating static Pod manifest for "kube-scheduler"
  W0402 11:30:26.195753   38671 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
  [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
  [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
  [kubelet-check] Initial timeout of 40s passed.

    Unfortunately, an error has occurred:
      timed out waiting for the condition

    This error is likely caused by:
      - The kubelet is not running
      - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

    If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
      - 'systemctl status kubelet'
      - 'journalctl -xeu kubelet'

    Additionally, a control plane component may have crashed or exited when started by the container runtime.
    To troubleshoot, list all containers using your preferred container runtimes CLI.

    Here is one example how you may list all Kubernetes containers running in docker:
      - 'docker ps -a | grep kube | grep -v pause'
      Once you have found the failing container, you can inspect its logs with:
      - 'docker logs CONTAINERID'

  error execution phase wait-control-plane: couldn't initialize a Kubernetes cluster
  To see the stack trace of this error execute with --v=5 or higher
  ```

### case 5

- 贴个安装成功的过程
  - 该过程使用了 -v=6 日志等级输出

  ```log
  [root@tmp49h ~]# kubeadm --config .kube/config init --ignore-preflight-errors=all --v=6
  I0402 12:04:52.044950   52108 initconfiguration.go:200] loading configuration from ".kube/config"
  I0402 12:04:52.048675   52108 interface.go:400] Looking for default routes with IPv4 addresses
  I0402 12:04:52.048691   52108 interface.go:405] Default route transits interface "eth0"
  I0402 12:04:52.049188   52108 interface.go:208] Interface eth0 is up
  I0402 12:04:52.049282   52108 interface.go:256] Interface "eth0" has 2 addresses :[10.0.1.5/24 fe80::217:faff:fe00:a2b3/64].
  I0402 12:04:52.049310   52108 interface.go:223] Checking addr  10.0.1.5/24.
  I0402 12:04:52.049323   52108 interface.go:230] IP found 10.0.1.5
  I0402 12:04:52.049337   52108 interface.go:262] Found valid IPv4 address 10.0.1.5 for interface "eth0".
  I0402 12:04:52.049348   52108 interface.go:411] Found active IP 10.0.1.5 
  W0402 12:04:52.049539   52108 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
  [init] Using Kubernetes version: v1.18.0
  [preflight] Running pre-flight checks
  I0402 12:04:52.049892   52108 checks.go:577] validating Kubernetes and kubeadm version
  I0402 12:04:52.049924   52108 checks.go:166] validating if the firewall is enabled and active
  I0402 12:04:52.076294   52108 checks.go:201] validating availability of port 6443
  I0402 12:04:52.078115   52108 checks.go:201] validating availability of port 10259
  I0402 12:04:52.078177   52108 checks.go:201] validating availability of port 10257
  I0402 12:04:52.078232   52108 checks.go:286] validating the existence of file /etc/kubernetes/manifests/kube-apiserver.yaml
  I0402 12:04:52.078252   52108 checks.go:286] validating the existence of file /etc/kubernetes/manifests/kube-controller-manager.yaml
  I0402 12:04:52.078265   52108 checks.go:286] validating the existence of file /etc/kubernetes/manifests/kube-scheduler.yaml
  I0402 12:04:52.078277   52108 checks.go:286] validating the existence of file /etc/kubernetes/manifests/etcd.yaml
  I0402 12:04:52.078298   52108 checks.go:432] validating if the connectivity type is via proxy or direct
  I0402 12:04:52.078353   52108 checks.go:471] validating http connectivity to first IP address in the CIDR
  I0402 12:04:52.078381   52108 checks.go:471] validating http connectivity to first IP address in the CIDR
  I0402 12:04:52.078402   52108 checks.go:102] validating the container runtime
  I0402 12:04:52.224441   52108 checks.go:128] validating if the service is enabled and active
    [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
  I0402 12:04:52.388976   52108 checks.go:335] validating the contents of file /proc/sys/net/bridge/bridge-nf-call-iptables
  I0402 12:04:52.389091   52108 checks.go:335] validating the contents of file /proc/sys/net/ipv4/ip_forward
  I0402 12:04:52.389159   52108 checks.go:649] validating whether swap is enabled or not
  I0402 12:04:52.389480   52108 checks.go:376] validating the presence of executable conntrack
  I0402 12:04:52.389920   52108 checks.go:376] validating the presence of executable ip
  I0402 12:04:52.390250   52108 checks.go:376] validating the presence of executable iptables
  I0402 12:04:52.390276   52108 checks.go:376] validating the presence of executable mount
  I0402 12:04:52.390532   52108 checks.go:376] validating the presence of executable nsenter
  I0402 12:04:52.390558   52108 checks.go:376] validating the presence of executable ebtables
  I0402 12:04:52.390574   52108 checks.go:376] validating the presence of executable ethtool
  I0402 12:04:52.390596   52108 checks.go:376] validating the presence of executable socat
  I0402 12:04:52.390614   52108 checks.go:376] validating the presence of executable tc
  I0402 12:04:52.390629   52108 checks.go:376] validating the presence of executable touch
  I0402 12:04:52.390648   52108 checks.go:520] running all checks
  I0402 12:04:52.544118   52108 checks.go:406] checking whether the given node name is reachable using net.LookupHost
  I0402 12:04:52.544592   52108 checks.go:618] validating kubelet version
  I0402 12:04:52.649444   52108 checks.go:128] validating if the service is enabled and active
  I0402 12:04:52.661720   52108 checks.go:201] validating availability of port 10250
  I0402 12:04:52.661983   52108 checks.go:201] validating availability of port 2379
  I0402 12:04:52.662091   52108 checks.go:201] validating availability of port 2380
  I0402 12:04:52.662153   52108 checks.go:249] validating the existence and emptiness of directory /var/lib/etcd
  [preflight] Pulling images required for setting up a Kubernetes cluster
  [preflight] This might take a minute or two, depending on the speed of your internet connection
  [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
  I0402 12:04:52.732912   52108 checks.go:838] image exists: gcr.azk8s.cn/google_containers/kube-apiserver:v1.18.0
  I0402 12:04:52.803307   52108 checks.go:838] image exists: gcr.azk8s.cn/google_containers/kube-controller-manager:v1.18.0
  I0402 12:04:52.874891   52108 checks.go:838] image exists: gcr.azk8s.cn/google_containers/kube-scheduler:v1.18.0
  I0402 12:04:52.953082   52108 checks.go:838] image exists: gcr.azk8s.cn/google_containers/kube-proxy:v1.18.0
  I0402 12:04:53.028397   52108 checks.go:838] image exists: gcr.azk8s.cn/google_containers/pause:3.2
  I0402 12:04:53.105915   52108 checks.go:838] image exists: gcr.azk8s.cn/google_containers/etcd:3.4.3-0
  I0402 12:04:53.179742   52108 checks.go:838] image exists: gcr.azk8s.cn/google_containers/coredns:1.6.7
  I0402 12:04:53.179794   52108 kubelet.go:64] Stopping the kubelet
  [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
  [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
  [kubelet-start] Starting the kubelet
  [certs] Using certificateDir folder "/etc/kubernetes/pki"
  I0402 12:04:53.517439   52108 certs.go:103] creating a new certificate authority for ca
  [certs] Generating "ca" certificate and key
  [certs] Generating "apiserver" certificate and key
  [certs] apiserver serving cert is signed for DNS names [tmp49h kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [192.96.0.1 10.0.1.5]
  [certs] Generating "apiserver-kubelet-client" certificate and key
  I0402 12:04:54.676701   52108 certs.go:103] creating a new certificate authority for front-proxy-ca
  [certs] Generating "front-proxy-ca" certificate and key
  [certs] Generating "front-proxy-client" certificate and key
  I0402 12:04:56.537529   52108 certs.go:103] creating a new certificate authority for etcd-ca
  [certs] Generating "etcd/ca" certificate and key
  [certs] Generating "etcd/server" certificate and key
  [certs] etcd/server serving cert is signed for DNS names [tmp49h localhost] and IPs [10.0.1.5 127.0.0.1 ::1]
  [certs] Generating "etcd/peer" certificate and key
  [certs] etcd/peer serving cert is signed for DNS names [tmp49h localhost] and IPs [10.0.1.5 127.0.0.1 ::1]
  [certs] Generating "etcd/healthcheck-client" certificate and key
  [certs] Generating "apiserver-etcd-client" certificate and key
  I0402 12:04:58.028007   52108 certs.go:69] creating new public/private key files for signing service account users
  [certs] Generating "sa" key and public key
  [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
  I0402 12:04:58.417808   52108 kubeconfig.go:79] creating kubeconfig file for admin.conf
  [kubeconfig] Writing "admin.conf" kubeconfig file
  I0402 12:04:59.079377   52108 kubeconfig.go:79] creating kubeconfig file for kubelet.conf
  [kubeconfig] Writing "kubelet.conf" kubeconfig file
  I0402 12:04:59.330894   52108 kubeconfig.go:79] creating kubeconfig file for controller-manager.conf
  [kubeconfig] Writing "controller-manager.conf" kubeconfig file
  I0402 12:04:59.559569   52108 kubeconfig.go:79] creating kubeconfig file for scheduler.conf
  [kubeconfig] Writing "scheduler.conf" kubeconfig file
  [control-plane] Using manifest folder "/etc/kubernetes/manifests"
  [control-plane] Creating static Pod manifest for "kube-apiserver"
  I0402 12:04:59.782909   52108 manifests.go:91] [control-plane] getting StaticPodSpecs
  I0402 12:04:59.783878   52108 manifests.go:104] [control-plane] adding volume "ca-certs" for component "kube-apiserver"
  I0402 12:04:59.783895   52108 manifests.go:104] [control-plane] adding volume "etc-pki" for component "kube-apiserver"
  I0402 12:04:59.783905   52108 manifests.go:104] [control-plane] adding volume "k8s-certs" for component "kube-apiserver"
  I0402 12:04:59.795764   52108 manifests.go:121] [control-plane] wrote static Pod manifest for component "kube-apiserver" to "/etc/kubernetes/manifests/kube-apiserver.yaml"
  [control-plane] Creating static Pod manifest for "kube-controller-manager"
  I0402 12:04:59.795793   52108 manifests.go:91] [control-plane] getting StaticPodSpecs
  W0402 12:04:59.795959   52108 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
  I0402 12:04:59.796292   52108 manifests.go:104] [control-plane] adding volume "ca-certs" for component "kube-controller-manager"
  I0402 12:04:59.796308   52108 manifests.go:104] [control-plane] adding volume "etc-pki" for component "kube-controller-manager"
  I0402 12:04:59.796318   52108 manifests.go:104] [control-plane] adding volume "flexvolume-dir" for component "kube-controller-manager"
  I0402 12:04:59.796327   52108 manifests.go:104] [control-plane] adding volume "k8s-certs" for component "kube-controller-manager"
  I0402 12:04:59.796336   52108 manifests.go:104] [control-plane] adding volume "kubeconfig" for component "kube-controller-manager"
  I0402 12:04:59.797418   52108 manifests.go:121] [control-plane] wrote static Pod manifest for component "kube-controller-manager" to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
  [control-plane] Creating static Pod manifest for "kube-scheduler"
  I0402 12:04:59.797444   52108 manifests.go:91] [control-plane] getting StaticPodSpecs
  W0402 12:04:59.797514   52108 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
  I0402 12:04:59.797782   52108 manifests.go:104] [control-plane] adding volume "kubeconfig" for component "kube-scheduler"
  I0402 12:04:59.798954   52108 manifests.go:121] [control-plane] wrote static Pod manifest for component "kube-scheduler" to "/etc/kubernetes/manifests/kube-scheduler.yaml"
  [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
  I0402 12:04:59.800928   52108 local.go:72] [etcd] wrote Static Pod manifest for a local etcd member to "/etc/kubernetes/manifests/etcd.yaml"
  I0402 12:04:59.800958   52108 waitcontrolplane.go:87] [wait-control-plane] Waiting for the API server to be healthy
  I0402 12:04:59.802317   52108 loader.go:375] Config loaded from file:  /etc/kubernetes/admin.conf
  [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
  I0402 12:04:59.804411   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s  in 0 milliseconds
  I0402 12:05:00.305351   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s  in 0 milliseconds
  I0402 12:05:00.805367   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s  in 0 milliseconds
  I0402 12:05:01.305315   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s  in 0 milliseconds
  I0402 12:05:06.305168   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s  in 0 milliseconds
  I0402 12:05:06.809393   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s  in 2 milliseconds
  I0402 12:05:13.635622   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s 500 Internal Server Error in 6330 milliseconds
  I0402 12:05:13.807469   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s 500 Internal Server Error in 2 milliseconds
  I0402 12:05:14.307830   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s 500 Internal Server Error in 2 milliseconds
  I0402 12:05:14.809268   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s 500 Internal Server Error in 4 milliseconds
  I0402 12:05:15.307555   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s 500 Internal Server Error in 2 milliseconds
  I0402 12:05:15.809277   52108 round_trippers.go:443] GET https://10.0.1.5:6443/healthz?timeout=10s 200 OK in 4 milliseconds
  [apiclient] All control plane components are healthy after 16.005791 seconds
  I0402 12:05:15.809404   52108 uploadconfig.go:108] [upload-config] Uploading the kubeadm ClusterConfiguration to a ConfigMap
  [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
  I0402 12:05:15.815779   52108 round_trippers.go:443] POST https://10.0.1.5:6443/api/v1/namespaces/kube-system/configmaps?timeout=10s 201 Created in 4 milliseconds
  I0402 12:05:15.824007   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/roles?timeout=10s 201 Created in 7 milliseconds
  I0402 12:05:15.829199   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/rolebindings?timeout=10s 201 Created in 4 milliseconds
  I0402 12:05:15.829943   52108 uploadconfig.go:122] [upload-config] Uploading the kubelet component config to a ConfigMap
  [kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
  I0402 12:05:15.834057   52108 round_trippers.go:443] POST https://10.0.1.5:6443/api/v1/namespaces/kube-system/configmaps?timeout=10s 201 Created in 2 milliseconds
  I0402 12:05:15.841508   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/roles?timeout=10s 201 Created in 7 milliseconds
  I0402 12:05:15.845856   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/rolebindings?timeout=10s 201 Created in 4 milliseconds
  I0402 12:05:15.845997   52108 uploadconfig.go:127] [upload-config] Preserving the CRISocket information for the control-plane node
  I0402 12:05:15.846066   52108 patchnode.go:30] [patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "tmp49h" as an annotation
  I0402 12:05:16.352672   52108 round_trippers.go:443] GET https://10.0.1.5:6443/api/v1/nodes/tmp49h?timeout=10s 200 OK in 6 milliseconds
  I0402 12:05:16.364925   52108 round_trippers.go:443] PATCH https://10.0.1.5:6443/api/v1/nodes/tmp49h?timeout=10s 200 OK in 5 milliseconds
  [upload-certs] Skipping phase. Please see --upload-certs
  [mark-control-plane] Marking the node tmp49h as control-plane by adding the label "node-role.kubernetes.io/master=''"
  [mark-control-plane] Marking the node tmp49h as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
  I0402 12:05:16.868754   52108 round_trippers.go:443] GET https://10.0.1.5:6443/api/v1/nodes/tmp49h?timeout=10s 200 OK in 2 milliseconds
  I0402 12:05:16.877566   52108 round_trippers.go:443] PATCH https://10.0.1.5:6443/api/v1/nodes/tmp49h?timeout=10s 200 OK in 7 milliseconds
  [bootstrap-token] Using token: abcdef.0123456789abcdef
  [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
  I0402 12:05:16.884804   52108 round_trippers.go:443] GET https://10.0.1.5:6443/api/v1/namespaces/kube-system/secrets/bootstrap-token-abcdef?timeout=10s 404 Not Found in 6 milliseconds
  I0402 12:05:16.893841   52108 round_trippers.go:443] POST https://10.0.1.5:6443/api/v1/namespaces/kube-system/secrets?timeout=10s 201 Created in 8 milliseconds
  [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
  I0402 12:05:16.908160   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/clusterroles?timeout=10s 201 Created in 13 milliseconds
  I0402 12:05:16.914868   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/clusterrolebindings?timeout=10s 201 Created in 5 milliseconds
  [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
  I0402 12:05:16.922119   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/clusterrolebindings?timeout=10s 201 Created in 6 milliseconds
  [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
  I0402 12:05:16.926440   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/clusterrolebindings?timeout=10s 201 Created in 4 milliseconds
  [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
  I0402 12:05:16.935959   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/clusterrolebindings?timeout=10s 201 Created in 9 milliseconds
  [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
  I0402 12:05:16.936118   52108 clusterinfo.go:45] [bootstrap-token] loading admin kubeconfig
  I0402 12:05:16.937438   52108 loader.go:375] Config loaded from file:  /etc/kubernetes/admin.conf
  I0402 12:05:16.937464   52108 clusterinfo.go:53] [bootstrap-token] copying the cluster from admin.conf to the bootstrap kubeconfig
  I0402 12:05:16.937878   52108 clusterinfo.go:65] [bootstrap-token] creating/updating ConfigMap in kube-public namespace
  I0402 12:05:16.944203   52108 round_trippers.go:443] POST https://10.0.1.5:6443/api/v1/namespaces/kube-public/configmaps?timeout=10s 201 Created in 6 milliseconds
  I0402 12:05:16.944386   52108 clusterinfo.go:79] creating the RBAC rules for exposing the cluster-info ConfigMap in the kube-public namespace
  I0402 12:05:16.952131   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-public/roles?timeout=10s 201 Created in 7 milliseconds
  I0402 12:05:16.960717   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-public/rolebindings?timeout=10s 201 Created in 8 milliseconds
  I0402 12:05:16.962014   52108 kubeletfinalize.go:88] [kubelet-finalize] Assuming that kubelet client certificate rotation is enabled: found "/var/lib/kubelet/pki/kubelet-client-current.pem"
  [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
  I0402 12:05:16.962805   52108 loader.go:375] Config loaded from file:  /etc/kubernetes/kubelet.conf
  I0402 12:05:16.964191   52108 kubeletfinalize.go:132] [kubelet-finalize] Restarting the kubelet to enable client certificate rotation
  I0402 12:05:17.144392   52108 round_trippers.go:443] GET https://10.0.1.5:6443/apis/apps/v1/namespaces/kube-system/deployments?labelSelector=k8s-app%3Dkube-dns 200 OK in 14 milliseconds
  I0402 12:05:17.160533   52108 round_trippers.go:443] GET https://10.0.1.5:6443/api/v1/namespaces/kube-system/configmaps/kube-dns?timeout=10s 404 Not Found in 7 milliseconds
  I0402 12:05:17.167540   52108 round_trippers.go:443] GET https://10.0.1.5:6443/api/v1/namespaces/kube-system/configmaps/coredns?timeout=10s 404 Not Found in 5 milliseconds
  I0402 12:05:17.177800   52108 round_trippers.go:443] POST https://10.0.1.5:6443/api/v1/namespaces/kube-system/configmaps?timeout=10s 201 Created in 9 milliseconds
  I0402 12:05:17.183843   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/clusterroles?timeout=10s 201 Created in 5 milliseconds
  I0402 12:05:17.192713   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/clusterrolebindings?timeout=10s 201 Created in 8 milliseconds
  I0402 12:05:17.204923   52108 round_trippers.go:443] POST https://10.0.1.5:6443/api/v1/namespaces/kube-system/serviceaccounts?timeout=10s 201 Created in 7 milliseconds
  I0402 12:05:17.241253   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/apps/v1/namespaces/kube-system/deployments?timeout=10s 201 Created in 25 milliseconds
  I0402 12:05:17.259595   52108 round_trippers.go:443] POST https://10.0.1.5:6443/api/v1/namespaces/kube-system/services?timeout=10s 201 Created in 15 milliseconds
  [addons] Applied essential addon: CoreDNS
  I0402 12:05:17.269186   52108 round_trippers.go:443] POST https://10.0.1.5:6443/api/v1/namespaces/kube-system/serviceaccounts?timeout=10s 201 Created in 9 milliseconds
  I0402 12:05:17.279073   52108 round_trippers.go:443] POST https://10.0.1.5:6443/api/v1/namespaces/kube-system/configmaps?timeout=10s 201 Created in 7 milliseconds
  I0402 12:05:17.303603   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/apps/v1/namespaces/kube-system/daemonsets?timeout=10s 201 Created in 15 milliseconds
  I0402 12:05:17.312854   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/clusterrolebindings?timeout=10s 201 Created in 8 milliseconds
  I0402 12:05:17.318643   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/roles?timeout=10s 201 Created in 5 milliseconds
  I0402 12:05:17.355049   52108 round_trippers.go:443] POST https://10.0.1.5:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/rolebindings?timeout=10s 201 Created in 36 milliseconds
  [addons] Applied essential addon: kube-proxy
  I0402 12:05:17.357075   52108 loader.go:375] Config loaded from file:  /etc/kubernetes/admin.conf
  I0402 12:05:17.357930   52108 loader.go:375] Config loaded from file:  /etc/kubernetes/admin.conf

  Your Kubernetes control-plane has initialized successfully!

  To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

  You should now deploy a pod network to the cluster.
  Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

  Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join 10.0.1.5:6443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:9a4b14630871dd9c8c6cd19787a305ef1188337375682085459d5d5bcd2d2b52
  ```

### case 6

- kubeadm join 使用的 token 默认有效期 24 小时
  - 通过 `kubeadm token list` 查看
  - 过期后通过 `kubeadm token create` 创建
- `--discovery-token-ca-cert-hash` 丢失处理方式，然后用新的token和ca-hash加入集群
  - `openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2&gt;/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'`

### case 7

- 这错误还没查到是啥意思
  - 以为是根据主机名无法解析，本地添加了 /etc/hosts 也没用

  ```log
  4月 02 11:46:47 tmp49h kubelet[42431]: E0402 11:46:47.388209   42431 kubelet.go:2267] node "tmp49h" not found
  4月 02 11:46:47 tmp49h kubelet[42431]: I0402 11:46:47.455578   42431 kubelet_node_status.go:294] Setting node annotation to enable volume controller attach/detach
  4月 02 11:46:47 tmp49h kubelet[42431]: I0402 11:46:47.456129   42431 kubelet_node_status.go:294] Setting node annotation to enable volume controller attach/detach
  4月 02 11:46:47 tmp49h kubelet[42431]: I0402 11:46:47.458294   42431 topology_manager.go:219] [topologymanager] RemoveContainer - Container ID: 3633a94c2ab917bf6e58ed4e851ecaffbeed5bf13f9dc8a843100cd0da5d40f4
  4月 02 11:46:47 tmp49h kubelet[42431]: I0402 11:46:47.459307   42431 topology_manager.go:219] [topologymanager] RemoveContainer - Container ID: 9bc11dafd96de2fad4223d92b8a5871f785841f84a03b31899294a944ebc4b03
  4月 02 11:46:47 tmp49h kubelet[42431]: E0402 11:46:47.459989   42431 pod_workers.go:191] Error syncing pod 49c6953d8c6d010ee4fd5a17c89d4f21 ("etcd-tmp49h_kube-system(49c6953d8c6d010ee4fd5a17c89d4f21)"), skipping: failed to "StartContainer" for "etcd" with CrashLoopBackOff: "back-off 40s restarting failed container=etcd pod=etcd-tmp49h_kube-system(49c6953d8c6d010ee4fd5a17c89d4f21)"
  4月 02 11:50:43 tmp49h kubelet[43440]: E0402 11:50:43.508686   43440 pod_workers.go:191] Error syncing pod 69f26225068c8c6c0e2cfe490854d0b9 ("kube-apiserver-tmp49h_kube-system(69f26225068c8c6c0e2cfe490854d0b9)"), skipping: failed to "StartContainer" for "kube-apiserver" with CrashLoopBackOff: "back-off 20s restarting failed container=kube-apiserver pod=kube-apiserver-tmp49h_kube-system(69f26225068c8c6c0e2cfe490854d0b9)"
  4月 02 11:50:43 tmp49h kubelet[43440]: E0402 11:50:43.573473   43440 kubelet.go:2267] node "tmp49h" not found
  4月 02 11:50:44 tmp49h kubelet[43440]: I0402 11:50:44.875613   43440 kubelet_node_status.go:294] Setting node annotation to enable volume controller attach/detach
  4月 02 11:50:44 tmp49h kubelet[43440]: I0402 11:50:44.879380   43440 topology_manager.go:219] [topologymanager] RemoveContainer - Container ID: 0562b4ea5f837a8e89ad447860be51c652a0665389ce1980359f4c5bf12c890f
  4月 02 11:50:44 tmp49h kubelet[43440]: E0402 11:50:44.879901   43440 pod_workers.go:191] Error syncing pod 49c6953d8c6d010ee4fd5a17c89d4f21 ("etcd-tmp49h_kube-system(49c6953d8c6d010ee4fd5a17c89d4f21)"), skipping: failed to "StartContainer" for "etcd" with CrashLoopBackOff: "back-off 40s restarting failed container=etcd pod=etcd-tmp49h_kube-system(49c6953d8c6d010ee4fd5a17c89d4f21)"
  4月 02 11:50:45 tmp49h kubelet[43440]: E0402 11:50:45.330608   43440 controller.go:136] failed to ensure node lease exists, will retry in 3.2s, error: Get https://1.2.3.4:6443/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/tmp49h?timeout=10s: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
  4月 02 11:50:47 tmp49h kubelet[43440]: W0402 11:50:47.133078   43440 cni.go:237] Unable to update cni config: no networks found in /etc/cni/net.d

  4月 02 11:50:47 tmp49h kubelet[43440]: E0402 11:50:47.568483   43440 kubelet.go:2187] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
  ```
