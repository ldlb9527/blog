+++
author = "旅店老板"
title = "k8s源码之kube-scheduler"
date = "2024-03-02"
description = "从k8s的基本知识到源码的学习"
tags = [
	"istio",
]
categories = [
    "istio",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "golang.png"
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
## kube-scheduler
### 启动调度器
启动入口在`cmd/kube-scheduler/scheduler.go`的main()函数，保留主干代码如下:
```go
func main() {
	command := app.NewSchedulerCommand()
	
	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```
核心函数就两行:
1. `command := app.NewSchedulerCommand()`创建一个`*cobra.Command`对象
2. `command.Execute()`会调用`*cobra.Command`对象的那个Run字段
***
`NewSchedulerCommand()`函数保留主干代码如下:
```go
func NewSchedulerCommand(registryOptions ...Option) *cobra.Command {
	opts, err := options.NewOptions()
	if err != nil {
		klog.Fatalf("unable to initialize command options: %v", err)
	}

	cmd := &cobra.Command{
		Use: "kube-scheduler",
		Long: ``,
		Run: func(cmd *cobra.Command, args []string) {
			if err := runCommand(cmd, args, opts, registryOptions...); err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
		},
	}
	return cmd
}
```
核心方法大概也是两行：
1. `options.NewOptions()`创建了`*Options`对象，是apiserver的连接配置信息和调度的一些策略配置
2. cobra最终调用的是`runCommand(cmd, args, opts, registryOptions...)`函数，传入了`*Options`对象
***
`runCommand()`函数保留主干代码如下:
```go
func runCommand(cmd *cobra.Command, args []string, opts *options.Options, registryOptions ...Option) error {
	c, err := opts.Config()
	if err != nil {
		return err
	}

	cc := c.Complete()
	return Run(ctx, cc, registryOptions...)
}
```
* `opts.Config()`获取的配置有client、PodInformer等， 获取的配置通过`c.Complete()`封装到CompletedConfig结构体的成员变量中
* `runCommand()`函数最终调用了`Run()` 函数
***
`Run()`函数保留主干代码如下:
```go
func Run(ctx context.Context, cc schedulerserverconfig.CompletedConfig, outOfTreeRegistryOptions ...Option) error {
	// Create the scheduler.
	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory,
		cc.PodInformer,
		recorderFactory,
		ctx.Done(),
		scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
		scheduler.WithAlgorithmSource(cc.ComponentConfig.AlgorithmSource),
		scheduler.WithPreemptionDisabled(cc.ComponentConfig.DisablePreemption),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithBindTimeoutSeconds(cc.ComponentConfig.BindTimeoutSeconds),
		scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
		scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
		scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
		scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
	)
	if err != nil {
		return err
	}
	
	// Start up the healthz server.
	if cc.InsecureServing != nil {
		separateMetrics := cc.InsecureMetricsServing != nil
		handler := buildHandlerChain(newHealthzHandler(&cc.ComponentConfig, separateMetrics, checks...), nil, nil)
		if err := cc.InsecureServing.Serve(handler, 0, ctx.Done()); err != nil {
			return fmt.Errorf("failed to start healthz server: %v", err)
		}
	}
	if cc.InsecureMetricsServing != nil {
		handler := buildHandlerChain(newMetricsHandler(&cc.ComponentConfig), nil, nil)
		if err := cc.InsecureMetricsServing.Serve(handler, 0, ctx.Done()); err != nil {
			return fmt.Errorf("failed to start metrics server: %v", err)
		}
	}
	if cc.SecureServing != nil {
		handler := buildHandlerChain(newHealthzHandler(&cc.ComponentConfig, false, checks...), cc.Authentication.Authenticator, cc.Authorization.Authorizer)
		// TODO: handle stoppedCh returned by c.SecureServing.Serve
		if _, err := cc.SecureServing.Serve(handler, 0, ctx.Done()); err != nil {
			// fail early for secure handlers, removing the old error loop from above
			return fmt.Errorf("failed to start secure server: %v", err)
		}
	}

	// Start all informers.
	go cc.PodInformer.Informer().Run(ctx.Done())
	cc.InformerFactory.Start(ctx.Done())

	// Wait for all caches to sync before scheduling.
	cc.InformerFactory.WaitForCacheSync(ctx.Done())

	// If leader election is enabled, runCommand via LeaderElector until done and exit.
	if cc.LeaderElection != nil {
		cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
			OnStartedLeading: sched.Run,
			OnStoppedLeading: func() {
				klog.Fatalf("leaderelection lost")
			},
		}
		leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}

		leaderElector.Run(ctx)

		return fmt.Errorf("lost lease")
	}

	// Leader election is disabled, so runCommand inline until done.
	sched.Run(ctx)
	return fmt.Errorf("finished without leader elect")
}
```
`Run()`函数为启动调度器的核心函数
* `scheduler.New()`函数创建了一个调度器sched，最开始传入的配置在该函数得到使用，如`cc.Client`、`cc.InformerFactory`、`cc.PodInformer`  
* 下面是`scheduler.New()`函数的部分逻辑，提供了两种设置预选和优选算法并初始化**Scheduler**的方式，默认情况下：当`source.Provider != nil`调用`createFromProvider()`函数生成**Scheduler**，
自定义情况下：`source.Policy != nil`调用`initPolicyFromXXX()`,有两种方式加载配置，一种是File，另一种是ConfigMap。如果**Provider**和**Policy**都不为空的情况，按switch的逻辑自定义策略会覆盖默认策略

```go
func New() (*Scheduler, error) {
	//...
	var sched *Scheduler
	source := options.schedulerAlgorithmSource
	
	switch {
	case source.Provider != nil:
		
		// Create the config from a named algorithm provider.
		sc, err := configurator.createFromProvider(*source.Provider)
		sched = sc
		
	case source.Policy != nil:
		
		// Create the config from a user specified policy source.
		policy := &schedulerapi.Policy{}
		switch {
		case source.Policy.File != nil:
			if err := initPolicyFromFile(source.Policy.File.Path, policy); err != nil {
				return nil, err
			}
		case source.Policy.ConfigMap != nil:
			if err := initPolicyFromConfigMap(client, source.Policy.ConfigMap, policy); err != nil {
				return nil, err
			}
		}
		sched = sc
		
	default:
		return nil, fmt.Errorf("unsupported algorithm source: %v", source)
	}
	return sched, nil
}
```
* `newHealthzHandler`和`newHealthzHandler`分别提供了`/healthz`和`/metrics`结构的具体处理逻辑，用于健康检测和指标监控
* 启动所有informer，`cc.PodInformer.Informer().Run(ctx.Done())`、`cc.InformerFactory.Start(ctx.Done())`
* `cc.LeaderElection != nil`表示调度器配置了高可用,用选举模式启动调度器，`OnStartedLeading`的值设置为`sched.Run`,`leaderElector.Run(ctx)`中会调用该函数
* 如果调度器是非集群模式，会跳过if中的逻辑，直接执行调度器的`Run()`函数,即`sched.Run(ctx)`,调度器的`Run()`函数中是具体的调度逻辑，只有异常情况才会退出，该函数位于`pkg/scheduler/scheduler.go`中
***
### 调度框架
我们来复习一下，k8s调度器的流程是从apiserver获取未被调度(nodeName字段为空)的pods,通过预选(过滤掉不合条件的node，比如内存、CPU资源)、优选(通过算法为node打分)为pod寻找一个最合适的node后，向apiserver发送一个post请求设置nodeName的值

上一节中已经调试到cobra最终执行到`sched.Run(ctx)`，我们先学习一下`Scheduler`结构体和它的`Run()`函数
* `Scheduler`结构体,位于`pkg/scheduler/scheduler.go`
```go
// 调度程序监视新的未调度的pods. 它试图找到节点，并将绑定写回apiserver。
type Scheduler struct {
	// 通过SchedulerCache做出的改变将被NodeLister和Algorithm观察到
	SchedulerCache internalcache.Cache

	Algorithm core.ScheduleAlgorithm
	// PodConditionUpdater is used only in case of scheduling errors. If we succeed
	// with scheduling, PodScheduled condition will be updated in apiserver in /bind
	// handler so that binding and setting PodCondition it is atomic.
	podConditionUpdater podConditionUpdater
	// PodPreemptor is used to evict pods and update 'NominatedNode' field of
	// the preemptor pod.
	podPreemptor podPreemptor

	// 应该是一个阻塞直到下一个Pod存在的函数。之所以不使用channel结构，
	// 是因为调度pod可能需要一些时间，k8s不希望pod位于通道中变得陈旧
	NextPod func() *framework.PodInfo

	// 在出现错误的时候被调用。如果有错误，它会传递有问题的pod信息，和错误
	Error func(*framework.PodInfo, error)

	// 通过关闭它来停止scheduler
	StopEverything <-chan struct{}

	// VolumeBinder handles PVC/PV binding for the pod.
	VolumeBinder scheduling.SchedulerVolumeBinder

	// Disable pod preemption or not.
	DisablePreemption bool

	// 保存着正在准备被调度的pod列表
	SchedulingQueue internalqueue.SchedulingQueue

	// 调度的策略
	Profiles profile.Map

	scheduledPodsHasSynced func() bool
}
```
***
* 调度器Scheduler的`Run()`函数
```go
//开始watching和scheduling。它等待缓存同步，然后开始调度并阻止，直到上下文完成。
func (sched *Scheduler) Run(ctx context.Context) {
	if !cache.WaitForCacheSync(ctx.Done(), sched.scheduledPodsHasSynced) {
		return
	}
	sched.SchedulingQueue.Run()
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
	sched.SchedulingQueue.Close()
}
```
* 核心逻辑是`wait.UntilWithContext(ctx, sched.scheduleOne, 0)`,入参有三个：`ctx`、`sched.scheduleOne`、`0`,进入方法内部具体逻辑如下:
```go

// BackoffUntil循环直到停止通道关闭，在BackoffManager给定的每个持续时间内运行。
// 如果滑动为真，则在f次运行后计算周期。如果它是假的，那么period包括f的运行时间。
func BackoffUntil(f func(), backoff BackoffManager, sliding bool, stopCh <-chan struct{}) {
	var t clock.Timer
	for {
		select {
		case <-stopCh:
			return
		default:
		}

		if !sliding {
			t = backoff.Backoff()
		}

		func() {
			defer runtime.HandleCrash()
			f()
		}()

		if sliding {
			t = backoff.Backoff()
		}

		// NOTE: b/c there is no priority selection in golang
		// it is possible for this to race, meaning we could
		// trigger t.C and stopCh, and t.C select falls through.
		// In order to mitigate we re-check stopCh at the beginning
		// of every loop to prevent extra executions of f().
		select {
		case <-stopCh:
			return
		case <-t.C():
		}
	}
}
```
该方法的具体逻辑
`BackoffUntil(f func(), backoff BackoffManager, sliding bool, stopCh <-chan struct{})`函数有4个入参：
* f: 具体的执行逻辑，这里为`sched.scheduleOne`
* backoff: 定时任务的结构体对象,duration传入的为0，jitter传入的也为0.0
```go
type jitteredBackoffManagerImpl struct {
	clock        clock.Clock
	duration     time.Duration   //定时任务的时间间隔
	jitter       float64         //如果大于0.0，间隔时间变为duration到 duration + maxFactor*duration的随机值
	backoffTimer clock.Timer
}
```
* sliding：如果为false，间隔时间包含f的执行时间，如果为true，不包含f的执行时间，也就是f执行完后才开始计时，这里外部传入的true
* `case <-stopCh:`这个分支读取channel的值，在没有写入或关闭channel时不会执行该分支，stopCh即为接受停止信号的channel
>**总结：**
> 
>调度器Scheduler的`Run()`函数中的`wait.UntilWithContext(ctx, sched.scheduleOne, 0)`的逻辑为循环调用`sched.scheduleOne`,两次调用之间的间隔时间为0
* `scheduleOne`  
`scheduleOne`为单个pod执行整个调度工作流，根据上面定时任务的设置情况，该函数是顺序执行的，执行完成后立即再次执行，没有并发情况
>**注意：**  
>`scheduleOne`执行完成是指通过post请求写一个binding到apiserver,而不会等kubelet创建pod