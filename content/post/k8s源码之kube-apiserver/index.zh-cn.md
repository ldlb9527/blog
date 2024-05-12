+++
author = "旅店老板"
title = "k8s源码学习之kube-apiserver"
date = "2024-03-04"
description = "k8s源码学习之kube-apiserver"
tags = [
	"k8s",
]
categories = [
    "k8s",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "k8s.jpg"
mermaid = true
+++
## Kubernetes组件
![k8s](components-of-kubernetes.svg "k8s")
* **kube-apiserver**：是Kubernetes控制平面的组件，负责对外开放Kubernetes API,接受处理http请求，是Kubernetes控制平面的前端。
* **etcd**:是一款分布式键值存储存储中间件，用于Kubernetes的所有集群数据的后台数据库。
* **kube-scheduler**：调度器是Kubernetes控制平面的组件，负责监听新创建、未指定运行节点的Pods，并选择最合适的节点让Pod运行。
>kube-scheduler调度策略因素包含单个Pod及Pods集合的资源需求、软硬件约束、亲和性规则等
* **kube-controller-manager**：控制器管理器是Kubernetes控制平面的组件， 负责运行控制器。
* **cloud-controller-manager**：云控制器管理器是Kubernetes控制平面的组件，负责与云平台交互。(本地环境运行k8s不需要启动该组件)
* **node**:节点组件，负责维护Pod的运行环境。
* **kubelet**：在集群的每个node上运行，从kube-apiserver组件接收新的或修改的PodSpecs，并确保Pod及其容器在期望规范下运行，同时向kube-apiserver汇报主机的运行状况。
* **kube-proxy**：是集群中每个节点上运行的网络代理，维护节点上的一些网络规则， 这些网络规则会允许从集群内部或外部的网络会话与Pod进行网络通信。
* **容器运行时(Container Runtime)**:用于支持许多容器运行环境。
### kube-apiserver
* kube-apiserver是整个集群的入口，提供REST API接口，支持同时提供https(6443端口)和http(8080端口),8080是非安全接口，不做任何认证授权机制,默认是关闭的。


* kube-apiserver的REST API格式是通过GVR实现，即group、version、resource。知道GVR，唯一的http路径就确定了，路径格式为`/api/{group}/{version}/namespaces/{namespace}/{resource}`  
通常我们通过**kubectl**来访问apiserver，追加`--v=7`可查看发送的具体请求，如下所示：
```shell
[root@master ~]# kubectl get daemonsets --v=7
I0410 17:01:45.706385   13197 loader.go:375] Config loaded from file:  /root/.kube/config
I0410 17:01:45.716737   13197 round_trippers.go:420] GET https://10.0.8.10:6443/apis/apps/v1/namespaces/default/daemonsets?limit=500
I0410 17:01:45.716750   13197 round_trippers.go:427] Request Headers:
I0410 17:01:45.716754   13197 round_trippers.go:431]     Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json
I0410 17:01:45.716759   13197 round_trippers.go:431]     User-Agent: kubectl/v1.18.2 (linux/amd64) kubernetes/52c56ce
I0410 17:01:45.722889   13197 round_trippers.go:446] Response Status: 200 OK in 6 millisecond
```
完整的http路径为：/apis/apps/v1/namespaces/default/daemonsets，Group为apps，还包括networking、extensions等，细心的发现部分资源是没有Group的，比如pod：
```shell
[root@master ~]# kubectl get pod --v=7
I0410 16:37:29.887597   29621 loader.go:375] Config loaded from file:  /root/.kube/config
I0410 16:37:29.908778   29621 round_trippers.go:420] GET https://10.0.8.10:6443/api/v1/namespaces/default/pods?limit=500
I0410 16:37:29.908797   29621 round_trippers.go:427] Request Headers:
I0410 16:37:29.908803   29621 round_trippers.go:431]     Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json
I0410 16:37:29.908818   29621 round_trippers.go:431]     User-Agent: kubectl/v1.18.2 (linux/amd64) kubernetes/52c56ce
I0410 16:37:29.923879   29621 round_trippers.go:446] Response Status: 200 OK in 15 milliseconds
```
完整的http路径为：/api/v1/namespaces/default/pods，这是因为这部分资源是在core包下，具体可通过`kubectl api-resources`和`kubectl api-versions`查看
>**为什么core包下的资源在http路径中没有core这个组？**  
> 
>事实上core、apps、networking、extensions等都属于同一层级的包名。  
>猜测k8s在最初的时候只有core包名，http路径中还没有Group的概念，后面引入GVR后http路径中引入对应的Group，core包为了兼容以前的版本的http路径，就没有引入core这个Group
