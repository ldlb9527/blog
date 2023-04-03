+++

author = "旅店老板"
title = "k8s的亲和性与反亲和性"
date = "2022-10-02                                                                                                                                                                                                                                                                                                                                            "
description = "k8s的亲和性与反亲和性"
tags = [
	"亲和性",
]
categories = [
    "kubernetes",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "k8s.jpg"
mermaid = true
+++

k8s环境：
```shell
[root@master ~]# kubectl get node
NAME     STATUS   ROLES    AGE    VERSION
master   Ready    master   363d   v1.18.2
node1    Ready    <none>   363d   v1.18.2
node2    Ready    <none>   363d   v1.18.2
node3    Ready    <none>   363d   v1.18.2
```
共有四个节点，master、node1、node2、node3，本文都在该环境下进行测试
## 背景
Scheduler调度器做为Kubernetes三大核心组件之一，承载着整个集群资源的调度功能，其根据特定调度算法和策略，将Pod调度到最优工作节点上，从而更合理与充分的利用集群计算资源。


节点亲和性就是在调度器中的NodeAffinity实现的
## nodeSelector
**nodeSelector**是节点调度最简单的方式，可以实现将Pod调度到带有对应标签的目标节点上。 
* 打标签  
`kubectl label nodes <node-name> <label-key>e=<label-value>`,通过如下命令给node2节点打上s=node2的标签：
```shell
kubectl label nodes node2 s=node2 
```
结果如下:
```shell
[root@master ~]# kubectl get node --show-labels
NAME     STATUS   ROLES    AGE    VERSION   LABELS
master   Ready    master   363d   v1.18.2   ...
node1    Ready    <none>   363d   v1.18.2   ...
node2    Ready    <none>   363d   v1.18.2   ...,s=node2
node3    Ready    <none>   363d   v1.18.2   ...
```
* 创建资源  
创建`pod-node2.yaml`资源，该yaml通过nodeSelector将pod调度到node2节点上,yaml文件内容如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    s: node2
```
使用如下命令创建资源：
```shell
kubectl apply -f pod-node2.yaml
```
* 检测结果
```shell
[root@master k8s-sch]# kubectl get pod nginx -owide
NAME    READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
nginx   2/2     Running   0          99s   10.244.1.144   node2   <none>           <none>
```
pod已被正确调度到node2节点上
## 亲和性与反亲和性
* 亲和性与反亲和性提供一种比nodeSelector有更强控制力的调度方式。

### 节点亲和性
节点亲和性概念上类似于nodeSelector，根据节点上的标签来约束Pod可以调度到哪些节点上，但语法调度控制力更强。节点亲和性有两种：
* **requiredDuringSchedulingIgnoredDuringExecution**： (硬亲和)调度器只有在规则被满足时才能执行调度
* **preferredDuringSchedulingIgnoredDuringExecution**： (软亲和)调度器尝试寻找满足规则的节点，如果匹配不到，调度器仍会调度

首先我们先给node2和node3节点分别打上region=east、region=west，使用如下命令：
```shell
[root@master k8s-sch]# kubectl label nodes node2 region=east && kubectl label nodes node3 region=west
node/node2 labeled
node/node3 labeled
```
你可以使用`spec.affinity.nodeAffinity`字段来设置节点亲和性,定义`nodeAffinity.yaml`资源文件，内容如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: region
                operator: In
                values:
                  - east
                  - west
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: s
                operator: In
                values:
                  - node2
  containers:
    - name: node-affinity
      image: nginx
```
* **operator**：逻辑操作符，取值有In、NotIn、Exists、DoesNotExist、Gt、Lt
* **weight**：节点亲和性权重，取值为1-100

该yaml描述了Pod调度的节点**必须有**region=east或region=west标签，该节点**最好有**s=node2的标签
>**注意：**  
> 如果同时指定了`nodeSelector` 和 `nodeAffinity`，两者必须都要满足规则，才能将Pod调度到节点上。
>
> 如果指定了多个与`nodeAffinity`类型关联的`nodeSelectorTerms`，只要其中一个`nodeSelectorTerms`满足规则，Pod就可以被调度到节点上。
> 
> 如果指定了多个与同一`nodeSelectorTerms`关联的`matchExpressions`， 则只有当所有`matchExpressions`都满足规则时，Pod才可以被调度到节点上。
***
### Pod亲和性
如果节点(机架、地理区域等)上已经运行了一个或多个满足规则的Pod，则这个pod应该运行在这个节点(机架、地理区域等)上

Pod亲和性有两种：
* **requiredDuringSchedulingIgnoredDuringExecution**： (硬亲和)调度器只有在规则被满足时才能执行调度，IgnoredDuringExecution是说明节点标签发生变化，已在节点运行的Pod不受影响
* **preferredDuringSchedulingIgnoredDuringExecution**： (软亲和)调度器尝试寻找满足规则的节点，如果匹配不到，调度器仍会调度

在实际中，我们可能将多个Pod调度到同一地理区域，因为它们的通信非常频繁，可以使用`spec.affinity.podAffinity`字段来设置Pod亲和性  

创建`pods-one.yaml`资源文件，内容如下：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity
spec:
  replicas: 2
  selector:
    matchLabels:
      app: affinity
  template:
    metadata:
      labels:
        app: affinity
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - affinity
              topologyKey: region
      containers:
        - name: pod-affinity
          image: nginx
          imagePullPolicy: Always  
```
>**topologyKey作用**:  
>pod亲和性和反亲和性调度要求pod运行于**同一位置**和不能运行于**同一位置**,这里的同一位置通过topologyKey来定义，topologyKey的值为node的标签名称  
>
> 比如node2节点的region=east和node3节点的region=west或有更多节点有这两个标签,当topologyKey的值为region时，pod就围绕着east拓扑和west拓扑来调度，相同拓扑下的node就为同一位置
> 
> 如果topologyKey的值为kubernetes.io/hostname时，**同一位置**就意味着同一节点，不同节点为不同位置，因为该标签的值通常是不同的

该资源文件定义了一个Deployment，有两个副本，每个副本有app=affinity标签，Pod亲和性表示Pod会调度到具有region=X标签的节点上，并且该节点上至少运行了一个app=affinity的pod
使用`kubectl apply -f pods-one.yaml`命令发布该资源，预期两个Pod会同时调度到node2或node3节点
```shell
[root@master k8s-sch]# kubectl get pod -owide
NAME                              READY   STATUS    RESTARTS   AGE    IP             NODE    NOMINATED NODE   READINESS GATES
affinity-6f65798795-7gjvn         2/2     Running   0          39m    10.244.3.85    node3   <none>           <none>
affinity-6f65798795-wqqxq         2/2     Running   0          39m    10.244.3.84    node3   <none>           <none>
```
无论是增加副本数还是删除Deployment重建，所有pod都会处于同一节点:node2或node3(这两个节点有region键的标签)
***
### Pod反亲和性
如果节点(机架、地理区域等)上已经运行了一个或多个满足规则的Pod，则这个pod不应该运行在这个节点(机架、地理区域等)上

在实际中，我们可能将一组Pod调度到不同的节点上，因为它们是IO密集型或者CPU密集型。可以使用`spec.affinity.podAntiAffinity`字段来设置Pod反亲和性

创建`pods-all.yaml`资源文件，内容如下：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: antiaffinity
spec:
  replicas: 4
  selector:
    matchLabels:
      app: antiaffinity
  template:
    metadata:
      labels:
        app: antiaffinity
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - antiaffinity
              topologyKey: kubernetes.io/hostname
      containers:
        - name: pod-antiaffinity
          image: nginx
          imagePullPolicy: Always  
```
topologyKey的值为kubernetes.io/hostname，定义副本数量为4(4个node)，使用`kubectl apply -f pods-all.yaml`命令发布资源，我们会得到4个pod分布在不同节点上，结果如下：
```shell
[root@master k8s-sch]# kubectl get pod -owide
NAME                              READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
antiaffinity-5b9865945f-2z4nz     2/2     Running   0          36s     10.244.1.145   node2    <none>           <none>
antiaffinity-5b9865945f-8jjmc     2/2     Running   0          36s     10.244.0.29    master   <none>           <none>
antiaffinity-5b9865945f-8xl8m     2/2     Running   0          36s     10.244.2.146   node1    <none>           <none>
antiaffinity-5b9865945f-k2c2g     2/2     Running   0          36s     10.244.3.86    node3    <none>           <none>
```
4个pod分别位于master、node1、node2、node3四个节点上，与预期一致

调整副本数为5(超过节点数)会发生什么呢?
```shell
[root@master k8s-sch]# kubectl get pod -owide
NAME                              READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
antiaffinity-5b9865945f-559v4     2/2     Running   0          35s     10.244.0.30    master   <none>           <none>
antiaffinity-5b9865945f-7th59     2/2     Running   0          35s     10.244.2.147   node1    <none>           <none>
antiaffinity-5b9865945f-hdvwz     2/2     Running   0          35s     10.244.3.87    node3    <none>           <none>
antiaffinity-5b9865945f-kqsl8     0/2     Pending   0          35s     <none>         <none>   <none>           <none>
antiaffinity-5b9865945f-vzdt4     2/2     Running   0          35s     10.244.1.146   node2    <none>           <none>
```
其中一个pod为Pending状态，无法被调度，查看event如下：
```shell
[root@master k8s-sch]# kubectl get events
LAST SEEN   TYPE      REASON              OBJECT                               MESSAGE
<unknown>   Warning   FailedScheduling    pod/antiaffinity-5b9865945f-7th59    0/4 nodes are available: 4 node(s) didn't match pod affinity/anti-affinity, 4 node(s) didn't satisfy existing pods anti-affinity rules.
```
>**注意：**  
> Pod亲和性和反亲和性都需要相当的计算量，因此会在大规模集群中显著降低调度速度。不建议在包含数百个节点的集群中使用这类设置。
> 
> Pod反亲和性需要节点上存在一致性的标签。如果部分或所有节点不存在指定的标签，调度行为可能不符合预期