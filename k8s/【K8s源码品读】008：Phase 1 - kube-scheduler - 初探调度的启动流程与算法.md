# 【K8s源码品读】008：Phase 1 - kube-scheduler - 初探调度的启动流程与算法

## 聚焦目标

理解kube-scheduler启动的流程



## 目录

1. [kube-scheduler的启动](#run)
2. [Scheduler的注册](#Scheduler)
3. [了解一个最简单的算法NodeName](#NodeName)



## run

```go
// kube-scheduler 类似于kube-apiserver，是个常驻进程，查看其对应的Run函数
func runCommand(cmd *cobra.Command, opts *options.Options, registryOptions ...Option) error {
	// 根据入参，返回配置cc与调度sched
   cc, sched, err := Setup(ctx, opts, registryOptions...)
	// 运行
   return Run(ctx, cc, sched)
}

// 运行调度策略
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	// 将配置注册到configz中，会保存在一个全局map里
	if cz, err := configz.New("componentconfig"); err == nil {
		cz.Set(cc.ComponentConfig)
	} else {
		return fmt.Errorf("unable to register configz: %s", err)
	}

	// 事件广播管理器，涉及到k8s里的一个核心资源 - Event事件，暂时不细讲
	cc.EventBroadcaster.StartRecordingToSink(ctx.Done())

	// 健康监测的服务
	var checks []healthz.HealthChecker

	// 异步各个Informer。Informer是kube-scheduler的一个重点
	go cc.PodInformer.Informer().Run(ctx.Done())
	cc.InformerFactory.Start(ctx.Done())
	cc.InformerFactory.WaitForCacheSync(ctx.Done())

	// 选举Leader的工作，因为Master节点可以存在多个，选举一个作为Leader
	if cc.LeaderElection != nil {
		cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
      // 两个钩子函数，开启Leading时运行调度，结束时打印报错
			OnStartedLeading: sched.Run,
			OnStoppedLeading: func() {
				klog.Fatalf("leaderelection lost")
			},
		}
		leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}
    // 参与选举的会持续通信
		leaderElector.Run(ctx)
		return fmt.Errorf("lost lease")
	}

	// 不参与选举的，也就是单节点的情况时，在这里运行
	sched.Run(ctx)
	return fmt.Errorf("finished without leader elect")
}

/*
到这里，我们已经接触了kube-scheduler的2个核心概念：
1. scheduler：正如程序名kube-scheduler，这个进程的核心作用是进行调度，会涉及到多种调度策略
2. Informer：k8s中有各种类型的资源，包括自定义的。而Informer的实现就将调度和资源结合了起来
*/
```



## Scheduler

```go
// 在创建scheduler的函数
func Setup() {
	// 创建scheduler，包括多个选项
	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory,
		cc.PodInformer,
		recorderFactory,
		ctx.Done(),
		scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
		scheduler.WithAlgorithmSource(cc.ComponentConfig.AlgorithmSource),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
		scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
		scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
		scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
	)
	return &cc, sched, nil
}

// 我们再看一下New这个函数
func New() (*Scheduler, error) {
  // 先注册了所有的算法，保存到一个 map[string]PluginFactory 中
  registry := frameworkplugins.NewInTreeRegistry()
  
  // 重点看一下Scheduler的创建过程
  var sched *Scheduler
	source := options.schedulerAlgorithmSource
	switch {
   // 根据Provider创建，重点看这里
	case source.Provider != nil:
		sc, err := configurator.createFromProvider(*source.Provider)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
		}
		sched = sc
  // 根据用户设置创建，来自文件或者ConfigMap
	case source.Policy != nil:
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
		configurator.extenders = policy.Extenders
		sc, err := configurator.createFromConfig(*policy)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler from policy: %v", err)
		}
		sched = sc
	default:
		return nil, fmt.Errorf("unsupported algorithm source: %v", source)
	}
}

// 创建
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	klog.V(2).Infof("Creating scheduler from algorithm provider '%v'", providerName)
  // 实例化算法的Registry
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
	if !exist {
		return nil, fmt.Errorf("algorithm provider %q is not registered", providerName)
	}

  // 将各种算法作为plugin进行设置
	for i := range c.profiles {
		prof := &c.profiles[i]
		plugins := &schedulerapi.Plugins{}
		plugins.Append(defaultPlugins)
		plugins.Apply(prof.Plugins)
		prof.Plugins = plugins
	}
	return c.create()
}

// 从这个初始化中可以看到，主要分为2类：默认与ClusterAutoscaler两种算法
func NewRegistry() Registry {
  // 默认算法包括过滤、打分、绑定等，有兴趣的去源码中逐个阅读
	defaultConfig := getDefaultConfig()
	applyFeatureGates(defaultConfig)
	// ClusterAutoscaler 是集群自动扩展的算法，被单独拎出来，
	caConfig := getClusterAutoscalerConfig()
	applyFeatureGates(caConfig)

	return Registry{
		schedulerapi.SchedulerDefaultProviderName: defaultConfig,
		ClusterAutoscalerProvider:                 caConfig,
	}
}
/*
在这里，熟悉k8s的朋友会有个疑问：以前听说kubernets的调度有个Predicate和Priority两个算法，这里怎么没有分类？
这个疑问，我们在后面具体场景时再进行分析。
*/
```



## NodeName

```go
// 为了加深大家对Plugin的印象，我选择一个最简单的示例：根据Pod的spec字段中的NodeName，分配到指定名称的节点
package nodename

import (
	"context"

	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	framework "k8s.io/kubernetes/pkg/scheduler/framework/v1alpha1"
)

type NodeName struct{}

var _ framework.FilterPlugin = &NodeName{}

// 这个调度算法的名称和错误信息
const (
	Name = "NodeName"
	ErrReason = "node(s) didn't match the requested hostname"
)

// 调度算法的明明
func (pl *NodeName) Name() string {
	return Name
}

// 过滤功能，这个就是NodeName算法的实现
func (pl *NodeName) Filter(ctx context.Context, _ *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
  // 找不到Node
	if nodeInfo.Node() == nil {
		return framework.NewStatus(framework.Error, "node not found")
	}
  // 匹配不到，返回错误
	if !Fits(pod, nodeInfo) {
		return framework.NewStatus(framework.UnschedulableAndUnresolvable, ErrReason)
	}
	return nil
}

/*
  匹配的算法，两种条件满足一个就认为成功
  1. spec没有填NodeName 
  2.spec的NodeName和节点匹配
*/
func Fits(pod *v1.Pod, nodeInfo *framework.NodeInfo) bool {
	return len(pod.Spec.NodeName) == 0 || pod.Spec.NodeName == nodeInfo.Node().Name
}

// 初始化
func New(_ runtime.Object, _ framework.FrameworkHandle) (framework.Plugin, error) {
	return &NodeName{}, nil
}
```

