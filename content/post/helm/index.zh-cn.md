+++

author = "旅店老板"
title = "helm从入门到精通"
date = "2022-11-18"
description = "Helm"
tags = [
	"helm",
]
categories = [
    "helm",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "helm.png"
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
***
# 基本概念
## Chart
*Chart* 代表着 Helm 包。它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。你可以把它看作是 Homebrew formula，Apt dpkg，或 Yum RPM 在Kubernetes 中的等价物。
## Repository
*Repository（仓库）* 是用来存放和共享 charts 的地方，类似yum包仓库。
## Release
*Release* 是运行在 Kubernetes 集群中的 chart 的实例。一个 chart 通常可以在同一个集群中安装多次。每一次安装都会创建一个新的 *release*。以 MySQL chart为例，如果你想在你的集群中运行两个数据库，你可以安装该chart两次，类似于镜像与容器的关系。
# 命令学习
## helm search
helm可以用来从两种来源中进行搜索  
* `helm search hub` 从 [Artifact Hub](https://artifacthub.io/) 中查找并列出 helm charts。 Artifact Hub中存放了大量不同的仓库
  通过`helm search hub mysql`从[Artifact Hub](https://artifacthub.io/)搜索mysql相关的chart
```shell
[root@master ~]# helm search hub mysql
URL                                               	CHART VERSION	APP VERSION            	DESCRIPTION                                       
https://artifacthub.io/packages/helm/stakater/m...	1.0.6        	                       	mysql chart that runs on kubernetes               
https://artifacthub.io/packages/helm/ygqygq2/mysql	4.5.3        	5.7.26                 	Chart to create a Highly available MySQL cluster  
https://artifacthub.io/packages/helm/saber/mysql  	8.8.21       	8.0.27                 	Chart to create a Highly available MySQL cluster  
https://artifacthub.io/packages/helm/kubesphere...	1.0.2        	5.7.33                 	High Availability MySQL Cluster, Open Source.     

```
* `helm search repo` 从你添加（使用 helm repo add）到本地 helm 客户端中的仓库中进行查找。该命令基于本地数据进行搜索，无需连接互联网。使用`helm repo list`命令查看添加的仓库
```shell
# 添加常用仓库
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
helm repo add aliyuncs https://apphub.aliyuncs.com
helm repo add stable http://mirror.azure.cn/kubernetes/charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
从添加的本地仓中库搜索mysql相关的chart,搜索使用模糊字符串匹配算法，下面搜索中出现了`mysqldump`
```shell
[root@master ~]# helm search repo mysql
NAME                                          	CHART VERSION	APP VERSION	DESCRIPTION                                       
aliyun/mysql                                  	0.3.5        	           	Fast, reliable, scalable, and easy to use open-...
aliyuncs/mysql                                	6.8.0        	8.0.19     	Chart to create a Highly available MySQL cluster  
aliyuncs/mysqldump                            	2.6.0        	2.4.1      	A Helm chart to help backup MySQL databases usi...
aliyuncs/mysqlha                              	1.0.0        	5.7.13     	MySQL cluster with a single master and zero or ...
aliyuncs/prometheus-mysql-exporter            	0.5.2        	v0.11.0    	A Helm chart for prometheus mysql exporter with...
bitnami/mysql                                 	8.9.0        	8.0.28     	MySQL is a fast, reliable, scalable, and easy t...

```
## helm install

