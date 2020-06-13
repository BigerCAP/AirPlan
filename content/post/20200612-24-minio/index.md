---
title: 软件测试 - S3 协议与 minio
date: 2020-06-12T00:00:00+08:00
lastmod: 2020-06-12T00:00:00+08:00
categories:
  - software
tags:
  - AirPlan
  - minio
---
## 0x00 S3

S3（Amazon Simple Storage Service (Amazon S3)）着重于简易性和稳健性的最小功能集【具体信息跳转到 [亚马逊 S3](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/dev/Introduction.html) 技术介绍页面】：

- 创建存储桶 – 创建和命名存储数据的存储桶。存储桶是 Amazon S3 中用于数据存储的基础容器
- 存储数据 – 在存储桶中存储无限量的数据。可将所需数量的对象上传到 Amazon S3 存储桶。每个对象可包含最多 5 TB 的数据。使用开发人员分配的唯一键值存储和检索每个对象
- 下载数据 – 下载您的数据或允许其他人进行下载。随时下载您的数据或允许其他人进行下载
- 权限 – 对于要在您的 Amazon S3 存储桶中上传或下载数据的其他人员，您可以授予其访问权限或拒绝其访问。将上传和下载的许可授予三种类型的用户。身份验证机制可帮助确保数据安全，以防未授权访问
- 标准接口 – 使用基于标准的 REST 和 SOAP 接口，它们可与任何 Internet 开发工具包搭配使用

> 个人使用

- 集中式标准接口：S3:// + backup + 文件路径 + AK/SK
- 精确 ACL 访问控制（主要是配合 AWS IAM 场景好用，minio 是按照 ak/sk 做的）
- KV 方式存储对象元数据信息，提供 sql 或类似接口查询
- 多媒体在线访问：如 图片、视频、音频等

## 0x01 MINio

[minio](https://min.io/) 开源（Apache License v2.0）、兼容 S3 协议、部署运维轻量化、高性能的对象存储服务。文档可以阅读 [minio docs](https://docs.min.io/docs/minio-quickstart-guide.html)，[Github MinIO](https://github.com/minio/minio)

安装部署支持 amd64、arm 、Windows 系统架构，支持 docker、binary 等安装方式。

> 测试 TiDB -- BR 时需要用到一个集中式存储服务，可以是 NFS 或者 S3 服务。

参考 [Minio Docs](https://docs.min.io/docs/minio-docker-quickstart-guide.html) 内容：

```shell
docker run -p 99:9000 \
    -v /data3/tmpuser/minio:/data \
    --name s3minio -d minio/minio server /data    \
    -e "MINIO_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE" \
    -e "MINIO_SECRET_KEY=wJalrXUtnFEMIK7MDENGbPxRfiCYEXAMPLEKEY"
```

docker 启动参数中不使用 `MINIO_ACCESS_KEY` 和 `MINIO_SECRET_KEY` 的情况下，AK/SK 默认为 `minioadmin/minioadmin`；启动后可以通过 `docker log 容器ID` 查看到运行信息。

## 0x02 分布式

minio 支持分布式部署，目前还未测试成功，持续在报错。。。
