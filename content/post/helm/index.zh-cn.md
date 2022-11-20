+++

author = "旅店老板"
title = "时序数据库InfluxDB初体验"
date = "2022-11-18"
description = "InfluxDB"
tags = [
	"influxdb",
]
categories = [
    "influxdb",
]
series = [""]
aliases = ["migrate-from-jekyl"]
mermaid = true

+++
# 前言
**Helm**是CNCF毕业项目，是K8S的包管理器，就像apt/yum/homebrew这些作为linux的包管理器一样，学了helm你需要`一个 Kubernetes 集群`，`安装和配置了Helm客户端`。以下是helm对kubernetes的版本支持列表：

| **Helm 版本** | **支持的 Kubernetes 版本* |
| :-------------: | :-------------------------: |
| 3.9.x         | 1.24.x - 1.21.x           |
| 3.8.x         | 1.23.x - 1.20.x           |
| 3.7.x         | 1.22.x - 1.19.x           |
| 3.6.x         | 1.21.x - 1.18.x           |
| 3.5.x         | 1.20.x - 1.17.x           |
| 3.4.x         | 1.19.x - 1.16.x           |
| 3.3.x         | 1.18.x - 1.15.x           |
| 3.2.x         | 1.18.x - 1.15.x           |
| 3.1.x         | 1.17.x - 1.14.x           |
| 3.0.x         | 1.16.x - 1.13.x           |
| 2.16.x        | 1.16.x - 1.15.x           |
| 2.15.x        | 1.15.x - 1.14.x           |
| 2.14.x        | 1.14.x - 1.13.x           |
| 2.13.x        | 1.13.x - 1.12.x           |
| 2.12.x        | 1.12.x - 1.11.x           |
| 2.11.x        | 1.11.x - 1.10.x           |
| 2.10.x        | 1.10.x - 1.9.x            |
| 2.9.x         | 1.10.x - 1.9.x            |



# 基本概念
## Chart

*Chart* 代表着 Helm 包。它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。你可以把它看作是 Homebrew formula，Apt dpkg，或 Yum RPM 在Kubernetes 中的等价物。

## Repository

*Repository（仓库）* 是用来存放和共享 charts 的地方，类似yum包仓库。

## Release

*Release* 是运行在 Kubernetes 集群中的 chart 的实例。一个 chart 通常可以在同一个集群中安装多次。每一次安装都会创建一个新的 *release*。以 MySQL chart为例，如果你想在你的集群中运行两个数据库，你可以安装该chart两次，类似于镜像与容器的关系。
