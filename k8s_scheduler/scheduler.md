# k8s scheduler源码初探

## 概览

scheduler是k8s一个核心的组件，它监测集群中创建的但是还没有被调度到node的pod，并将它调度到一个合适的node上。

## 需求和设计

什么是合适的node呢？

实际情况需要考虑的情况特别多，这里举例一些常见的情况：
- nodeSelector

        一个pod可能会筛选满足条件的node，通过label实现，灵活实现各种筛选条件。
- nodeAffinity

        节点亲和性，相比nodeSelector来说是软需求/偏好。
        亲和性也分软亲和/硬亲和，硬亲和和nodeSelector类似，但是提供了比nodeSelector更强大的筛选语法。

- podAffinity/podAntiAffinity

        pod之间亲和性和反亲和性，亲和性一个示例：“将服务 A 和服务 B 的 Pod 放置在同一区域，因为它们之间进行大量交流”，反亲和性一个示例：“将此服务的 pod 跨区域分布”。
- taints/toleration

        taints（污点）是应用于node，toleration（容忍度）应用于pod，调度器会避免将不能容忍污点的pod调度给污点对应的node。

- pod优先级和抢占

        当资源不足等情况时，一个高优先级pod可能会抢占低优先级的pod，具体抢占哪个node上的哪些pod，又需要综合考虑很多因素。

- node各种资源是否满足pod需求

        例如cpu，内存，磁盘，网络，端口等等。

- 优先调度到资源最充足的节点
- 资源平衡

        如果cpu被大量分配，但是memory分配很少，就属于资源不平衡，这样memory就被浪费了，这种情况应该尽量避免。

- 高可用

        例如同一个deployment的不同pod应该分配到不同的node上，尽量实现高可用。

- 等等其他因素

从上述列表可以看到，scheduler在调度的时候需要综合考虑一系列因素，然后找出最合适的node。


笔者认为，一个好的调度器它有以下优点：
- 资源高效利用
- 快速响应调度请求
- 灵活扩展，轻松实现自定义调度需求
- 良好的架构设计，降低维护和扩展的难度

接下来让我们来看看k8s是怎么实现scheduler的吧。

## 实现



### 整体架构

#### 流程图
![](./img/scheduler.png)

#### 概念解释

- informer

        利用etcd的Watch&List机制，在本地维护了自己所关心对象的缓存，同时能监测获取到这些关心对象的变化，并更新本地缓存，减轻api server压力。
- schedulerQueue

        activeQ： 一个堆数据结构构成的队列，即将被调度的pod就放在这个队列里，每次获取的pod都是优先级最高的一个。

        unschedulableQ：一个map数据结构构成的队列，一般情况下，当pod调度失败了，就放入这个队列。

        podBackoffQ：一个堆数据结构构成的队列，这里面的pod会按照指数退避策略重试。

- scheduleCache

        调度相关的cache，里面存储了跟调度相关的各种数据，很多是聚合产生的数据，这样不用每次再去查询、聚合，能大幅提升性能。

- framework

        framework是针对调度器设计的一种插件机制，上文提到的各种需求都是通过各种插件来实现的，这样能让调度器代码保持足够简单，还能保证扩展性，当需要更改一些策略时，更改相应的插件代码就行了，其他代码都不用动。
        流程图中倒数第二行都属于framework部分。

- predicates
        通过各种插件来过滤掉不满足条件的node。

- priorities
        通过各种插件将满足过滤条件的nodes打分，得到分数最高的。

- reserve
        为pod申明的一些资源预留，防止绑定的时候资源被其他pod占用导致绑定失败，因为绑定是异步的。

- permit
        通过插件来决定是否permit，可能允许，拒绝或者等待。

- bind
        通过调用apiserver接口将node和pod绑定，对应node上的kubelet会监听这个事件，来启动pod。
- preempt

        当调度失败时，scheduler会尝试寻找一个合适的node，并且抢占上面优先级较低的pods。




#### 流程简述：
- 当scheduler接收到自己关心的事件时，比如一个还未被调度的pod被创建了，就将这个pod放入调度队列中。

        还会监听各种跟调度相关的事件，比如某些资源变化，调度器认为这些资源变化了可能会让某些pod从不可调度状态变为可调度状态，会重新尝试调度。
- 启用一个协程不断从activeQ里面去pod，为这个pod做调度处理
- 使用framework的各种插件来筛选掉不合适的，将剩下的nodes进入下一阶段。
- 调度器同样使用插件来给各个nodes打分，然后选出最优的node。

- 当调度失败后，则执行抢占操作，调度器尝试找到合适的node，并将这些node上一些优先级较低等pods驱逐，然后再下一轮调取周期就很可能能够成功被调度了，如果抢占失败，这个pod会被放入调度队列等待再次被调度。
- 如果调度成功，调度器通过插件为pod预留一些pod申明的资源，以防异步绑定pod到node的时候资源已经被占用。
- 资源预留成功后，经过permit插件处理，处理结果可能是允许/拒绝或等待。
- 接下来调度器就开始下一个调度周期了，因为绑定是一个比较耗时的操作，所以将绑定异步处理，能够加快调度速度，

## 调度框架

介绍完大概的调度流程后，我们再来介绍下framework，我们知道大部分调度策略都是通过插件实现的，这样当我们需要更改或者定制我们自己的策略时，整个调度流程都是不用改的，扩展性得以体现。

这里我借用下官方文档的调度框架：
![](./img/framework.png)

框架设计了很多扩展点，会在不同阶段执行：

- sort
        pod排序逻辑，默认按照优先级、时间排序
- preFilter
        主要为过滤准备必要数据
- filter
        过滤nodes，剩下的nodes作为调度候选
- postFilter
        当调度失败时执行，主要用来抢占

- preScore
        为评分阶段准备好数据
- score
        将候选节点进行打分，每个配置的插件都参与打分，最后根据插件权重计算最终分数

- normalizeScore
        不同插件计算的最大分数可能不一样，无形中引入了权重，要将其标准化。

- reserve
        为pod在节点上预留资源

- permit

        一个调度周期最后执行的插件扩展点。


## 源码

啰嗦了一大堆，接下来让我们看下k8s是如何实现调度的吧。

### 代码目录结构

调度器代码入口在 `cmd/kube-scheduler`目录下,通过函数调用路径
`main` -> `runCommand` -> `Run` -> `sched.Run`来完成调度器的启动,cli, 参数校验，配置初始化等都在这里面实现的。


上面是cmd入口代码，而调度器主体代码在`pkg/scheduler`下面，执行下`tree -Ld 3`，看下代码目录结构：

![](./img/tree.png)

`apis/config`是目录相关的代码

`framework`从名字就能看出来是干嘛的了，可以看到里面有很多插件，有过滤和打分的，有的插件可能会有多个扩展点，比如nodeAffinity既有过滤又有打分扩展点。我们可以挑两个来说明它们的作用：

`selectorsSpread`：通过它可以将同一个deploy/rc的多个pods尽量分布在不同节点上，以实现高可用。

`imagelocality`:它可以帮助调度器决策时，考虑将pod调度到已经有镜像的节点上。

`internal`: `queue`, `heap`, `cache`这几个重要的数据结构都存放在这。

`profile`: 不同的framework注册的地方，调度的时候可以指定不同的framework，默认只有一个。

### scheduler默认插件配置
scheduler默认会开启以下插件,你可以配置不同的插件和他们的权重，来实现您想要的效果：

```golang

// getDefaultPlugins returns the default set of plugins.
func getDefaultPlugins() *v1beta3.Plugins {
	plugins := &v1beta3.Plugins{
		MultiPoint: v1beta3.PluginSet{
			Enabled: []v1beta3.Plugin{
				{Name: names.PrioritySort},//用来排序队列中的pod
				{Name: names.NodeUnschedulable},// 用来过滤pod无法调度的node
				{Name: names.NodeName},// pod可能指定了node名字
				{Name: names.TaintToleration, Weight: pointer.Int32(3)},// taint/toleration过滤和打分
				{Name: names.NodeAffinity, Weight: pointer.Int32(2)},
				{Name: names.NodePorts},// 过滤pod申明的端口被占用的node
				{Name: names.NodeResourcesFit, Weight: pointer.Int32(1)},// 根据资源情况过滤和打分
				{Name: names.VolumeRestrictions},
				{Name: names.EBSLimits},
				{Name: names.GCEPDLimits},
				{Name: names.NodeVolumeLimits},
				{Name: names.AzureDiskLimits},
				{Name: names.VolumeBinding},
				{Name: names.VolumeZone},
				{Name: names.PodTopologySpread, Weight: pointer.Int32(2)},//尽量将相同deploy/rc下的pods分布在不同的拓扑区域下 
				{Name: names.InterPodAffinity, Weight: pointer.Int32(2)}, // pod间的亲和性
				{Name: names.DefaultPreemption}, // 默认抢占逻辑
				{Name: names.NodeResourcesBalancedAllocation, Weight: pointer.Int32(1)}, // 资源平衡
				{Name: names.ImageLocality, Weight: pointer.Int32(1)},
				{Name: names.DefaultBinder}, // 默认bind实现
			},
		},
	}
    ...

	return plugins
}
```

### scheduler结构体
上面说到cmd的入口最终调用到主体代码中的函数是`sched.Run`，它的逻辑很简单：调用schedulingQueue.Run(),并循环调用sched.ScheduleOne()，开始循环调度。


```golang

// Scheduler watches for new unscheduled pods. It attempts to find
// nodes that they fit on and writes bindings back to the api server.
type Scheduler struct {
	// It is expected that changes made via SchedulerCache will be observed
	// by NodeLister and Algorithm.
	SchedulerCache internalcache.Cache
    // 调度逻辑
	Algorithm ScheduleAlgorithm
    // 第三方调度策略扩展，通过http api调用
	Extenders []framework.Extender


    // 取activeQ队首的pod，当没有pod可取的时候，会阻塞。
	NextPod func() *framework.QueuedPodInfo

	// 当调度失败的时候会由这个函数处理错误，它会在没有调度成功和抢占成功时，将pod重新放置到backOffQ或者unschedulableQ里面
	Error func(*framework.QueuedPodInfo, error)

	// Close this to shut down the scheduler.
	StopEverything <-chan struct{}

	// SchedulingQueue holds pods to be scheduled
	SchedulingQueue internalqueue.SchedulingQueue

	// Profiles are the scheduling profiles.
	Profiles profile.Map
    // client-go客户端，用来和api server通信
	client clientset.Interface
}

// Run begins watching and scheduling. It starts scheduling and blocked until the context is done.
func (sched *Scheduler) Run(ctx context.Context) {
	sched.SchedulingQueue.Run()
    // 不断循环调用sched.scheduleOne
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
	sched.SchedulingQueue.Close()
}



```

### schedule Queue

`scheduler.Run`首先调用的queue.Run函数，我们就先来看看它到底在做什么？

```golang

// PriorityQueue implements a scheduling queue.
// The head of PriorityQueue is the highest priority pending pod. This structure
// has three sub queues. One sub-queue holds pods that are being considered for
// scheduling. This is called activeQ and is a Heap. Another queue holds
// pods that are already tried and are determined to be unschedulable. The latter
// is called unschedulableQ. The third queue holds pods that are moved from
// unschedulable queues and will be moved to active queue when backoff are completed.
type PriorityQueue struct {
	// PodNominator abstracts the operations to maintain nominated Pods.
	framework.PodNominator
    ...

	// pod initial backoff duration.
	podInitialBackoffDuration time.Duration
	// pod maximum backoff duration.
	podMaxBackoffDuration time.Duration

    ...

	// activeQ is heap structure that scheduler actively looks at to find pods to
	// schedule. Head of heap is the highest priority pod.
	activeQ *heap.Heap
	// podBackoffQ is a heap ordered by backoff expiry. Pods which have completed backoff
	// are popped from this heap before the scheduler looks at activeQ
	podBackoffQ *heap.Heap
	// unschedulableQ holds pods that have been tried and determined unschedulable.
	unschedulableQ *UnschedulablePodsMap
	// schedulingCycle represents sequence number of scheduling cycle and is incremented
	// when a pod is popped.
    // 每个调度周期都会+1
	schedulingCycle int64

    // 当集群资源发生变化可能影响调度的时候，会更新这个值为schedulingCycle为schedulingCycle.
	moveRequestCycle int64

	clusterEventMap map[framework.ClusterEvent]sets.String

	// closed indicates that the queue is closed.
	// It is mainly used to let Pop() exit its control loop while waiting for an item.
	closed bool

	nsLister listersv1.NamespaceLister
}

// Run starts the goroutine to pump from podBackoffQ to activeQ
func (p *PriorityQueue) Run() {
    // 每1秒为backoffQ中的pod执行指数退避重试策略
	go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop)
    // 将这个队列里面待的时间超过一分钟的pod重新放入activeQ或者backoffQ
	go wait.Until(p.flushUnschedulableQLeftover, 30*time.Second, p.stop)
}


// flushBackoffQCompleted Moves all pods from backoffQ which have completed backoff in to activeQ
func (p *PriorityQueue) flushBackoffQCompleted() {
	p.lock.Lock()
	defer p.lock.Unlock()
	for {
		rawPodInfo := p.podBackoffQ.Peek()
		if rawPodInfo == nil {
			return
		}
		pod := rawPodInfo.(*framework.QueuedPodInfo).Pod
        // pod里会记录重试次数，这个函数会计算相应的重试时间
		boTime := p.getBackoffTime(rawPodInfo.(*framework.QueuedPodInfo))
		if boTime.After(p.clock.Now()) {
			return
		}
		_, err := p.podBackoffQ.Pop()
		if err != nil {
			klog.ErrorS(err, "Unable to pop pod from backoff queue despite backoff completion", "pod", klog.KObj(pod))
			return
		}
        // 将重试时间到了的pod加入activeQ
		p.activeQ.Add(rawPodInfo)
        ...
		defer p.cond.Broadcast()
	}
}

// flushUnschedulableQLeftover moves pods which stay in unschedulableQ longer than unschedulableQTimeInterval
// to backoffQ or activeQ.
func (p *PriorityQueue) flushUnschedulableQLeftover() {
	p.lock.Lock()
	defer p.lock.Unlock()

	var podsToMove []*framework.QueuedPodInfo
	currentTime := p.clock.Now()
	for _, pInfo := range p.unschedulableQ.podInfoMap {
		lastScheduleTime := pInfo.Timestamp
        // 当pod在这个队列中待了超过unschedulableQTimeInterval时间段
		if currentTime.Sub(lastScheduleTime) > unschedulableQTimeInterval {
			podsToMove = append(podsToMove, pInfo)
		}
	}

	if len(podsToMove) > 0 {
        // 如果集群中资源变化情况可能会影响unschedulableQ中的pod调度情况时，如果pod正在重试，则放入backoffQ，否则放入activeQ
		p.movePodsToActiveOrBackoffQueue(podsToMove, UnschedulableTimeout)
	}
}
```

为了更简单清晰描述pod在三个队列中的移动，我们可以用一张图来描述整个逻辑流程：

![](./img/queue.png)




