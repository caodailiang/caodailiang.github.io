---
layout:     post
title:      Kubernetes Scheduler 扩展开发
date:       2023-11-11
author:     caodailiang
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - kubernetes
    - scheduler
---
## Kubernetes Scheduler是什么
在 Kubernetes 中，调度是指将 Pod 放置到合适的节点上，以便对应节点上的 Kubelet 能够运行这些 Pod。

调度器通过 Kubernetes 的监测（Watch）机制来发现集群中新创建且尚未被调度到节点上的 Pod，Kube-scheduler 会根据调度策略选择一个最佳节点来调度该Pod。 由于 Pod 中的容器和 Pod 本身可能有不同的要求，调度程序会过滤掉任何不满足 Pod 特定调度需求的节点。

kube-scheduler 给一个 Pod 做调度选择时包含两个步骤：
1. **过滤**：将所有满足 Pod 调度需求的节点选出来，也就是可调度节点。如果可调度节点为空，那么这个 Pod 将一直停留在未调度状态直到调度器能够找到合适的 Node。
2. **打分**：根据当前启用的打分规则，调度器会给每一个可调度节点进行打分，从而在所有可调度节点中选取一个最合适的节点。

Kubernetes支持在集群中同时部署多个调度器，使用不同的调度器来对不同的Pod进行调度。Workload或Pod通过指定 `schedulerName` 来指定由哪个调度器来调度Pod，如果 `schedulerName` 未指定则由Kubernetes默认的 `default-scheduler` 来负责调度。
```
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  schedulerName: my-scheduler   # 指定调度器
  containers:
    - image: nginx
```

## 如何扩展Kubernetes Scheduler
#### 1、最简单的自定义调度器
调度器可以用任何语言来实现简单或复杂的调度器，调度器的最终目的就是将选择出来的Node名称更新到 Pod 的 `spec.nodeName` 字段。

下面的例子是官方用 Bash 实现的一个最简单的调度器，随机指派一个节点调度Pod。

```
#!/bin/bash
SERVER='localhost:8001'
while true;
do
    for PODNAME in $(kubectl --server $SERVER get pods -o json | jq '.items[] | select(.spec.schedulerName == "my-scheduler") | select(.spec.nodeName == null) | .metadata.name' | tr -d '"')
;
    do
        NODES=($(kubectl --server $SERVER get nodes -o json | jq '.items[].metadata.name' | tr -d '"'))
        NUMNODES=${#NODES[@]}
        CHOSEN=${NODES[$[ $RANDOM % $NUMNODES ]]}
        curl --header "Content-Type:application/json" --request POST --data '{"apiVersion":"v1", "kind": "Binding", "metadata": {"name": "'$PODNAME'"}, "target": {"apiVersion": "v1", "kind"
: "Node", "name": "'$CHOSEN'"}}' http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding/
        echo "Assigned $PODNAME to $CHOSEN"
    done
    sleep 1
done
```
该调度器仅负责调度 `schedulerName` 指定为 `my-scheduler` 的Pod，先找出 `spec.nodeName` 为空的Pod，然后随机选择一个节点，最终通过 `Binding` 操作将节点名称绑定到Pod上从而实现调度。 

#### 2、早期Scheduler Extender方案
社区最初提供的方案是通过Extender的形式来扩展scheduler，从而在原生的调度过程中加入自定义的调度策略。Extender是外部服务，支持Filter、Prioritize和Bind的扩展，kube-scheduler运行到相应阶段时，通过调用Extender注册的webhook来运行扩展的逻辑，影响调度流程中各阶段的决策结果。

![Kubernetes Scheduler Extender](https://caodailiang.github.io/img/posts/k8s-scheduler-extender.png)

**Scheduler Extender开发示例**

例如一个Filter扩展示例：

```
// scheduler-extender-filter.go
func main() {
    router := httprouter.New()
    router.POST("/filter", Filter)
    log.Fatal(http.ListenAndServe(":8888", router))
}

func Filter(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
    var (
        extenderArgs         schedulerapi.ExtenderArgs
        extenderFilterResult *schedulerapi.ExtenderFilterResult
    )
    
    // 读取输入
    body, err := ioutil.ReadAll(r.Body)
    json.NewDecoder(body).Decode(&extenderArgs);
    
    // 调度逻辑
    pod := extenderArgs.Pod
    nodes := extenderArgs.Nodes
    extenderFilterResult = filter(pod, nodes)

    // write response
    b, err := json.Marshal(extenderFilterResult)
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    w.Write(b)
}
```

运行方式：

1）生成extender策略配置，主要是配置上述filter的接口地址
```
$ cat /etc/sysconfig/kube-scheduler/extender-policy.json
{
    "kind" : "Policy",
    "apiVersion" : "v1",
    "extenders" : [{
        "urlPrefix": "http://localhost:8888/",
        "filterVerb": "filter",
        "enableHttps": false
    }]
}
```
2）生成 KubeSchedulerConfiguration 配置文件，引用上述extender策略配置文件
```
$ cat /etc/sysconfig/kube-scheduler/extender-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/root/.kube/config"
algorithmSource:
  policy:
    file:
      path: "/etc/sysconfig/kube-scheduler/extender-policy.json"
```
3）在kube-scheduler的启动配置中加入以上配置
```
$ /opt/kube/bin/kube-scheduler \
  --config=/etc/sysconfig/kube-scheduler/extender-config.yaml \     # 上述KubeSchedulerConfiguration配置文件
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2
```

**Extender存在的问题：**

- 调用Extender的接口是HTTP请求，受到网络环境的影响，性能远低于本地的函数调用。同时每次调用都需要将Pod和Node的信息进行marshaling和unmarshalling的操作，会进一步降低性能。
- 用户可以扩展的点比较有限，位置比较固定，无法支持灵活的扩展，例如只能在执行完默认的Filter策略后才能调用。

基于以上介绍，Extender的方式在集群规模较小，调度效率要求不高的情况下，是一个灵活可用的扩展方案，但是在正常生产环境的大型集群中，Extender无法支持高吞吐量，性能较差。

#### 3、Scheduling Framework方案
Scheduling Framework 是面向 Kubernetes 调度器的一种插件架构，它由一组直接编译到调度程序中的 “插件” API 组成。一个插件可能实现多个接口，以执行更为复杂或有状态的任务。这些 API 允许大多数调度功能以插件的形式实现，同时使调度 “核心” 保持简单且可维护。

Scheduling Framework 在原有的调度流程中, 定义了丰富扩展点接口，开发者可以通过实现扩展点所定义的接口来实现插件，将插件注册到扩展点。Scheduling Framework 在执行调度流程时，运行到相应的扩展点时，会调用用户注册的插件，这些插件中的一些可以改变调度决策，而另一些仅用于提供信息。通过这种方式来将用户的调度逻辑集成到 Scheduling Framework 中。

每次调度一个 Pod 的尝试都分为两个阶段，即调度周期和绑定周期。调度周期为 Pod 选择一个节点，绑定周期将该决策应用于集群。调度周期和绑定周期统称为 `Scheduling Context`。调度周期是串行运行的，而绑定周期可能是同时运行的。

![Kubernetes Scheduler Framework](https://caodailiang.github.io/img/posts/k8s-scheduling-framework-extensions.png)

**Scheduling Framework 开发示例**

为了方便管理调度插件，kube-scheduler中相关代码已经抽出来作为一个独立项目 [scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins)，用户可以直接基于这个项目来定义自己的插件。编译之后是个包含默认调度器和用户自定义插件的总调度器程序，既有内置调度器的功能，也包括了用户自己开发的功能。

以QoS插件为例，QoS插件的作用是根据Pod的QoS来决定调度顺序。

1）在scheduler-plugins的pkg目录下新建一个插件目录，如“qos”，然后在其中定义插件的对象和构造函数
```
// QoSSort is a plugin that implements QoS class based sorting.
type Sort struct{}

// Name is the name of the plugin used in the plugin registry and configurations.
const Name = "QOSSort"

// Name returns name of the plugin.
func (pl *Sort) Name() string {
	return Name
}

// New initializes a new plugin and returns it.
func New(_ *runtime.Unknown, _ framework.FrameworkHandle) (framework.Plugin, error) {
    return &Sort{}, nil
}
```

2）根据插件要对应的扩展点来实现对应的接口。QoS是对待调度Pod进行排序，作用于QueueSort部分，QueueSortPlugin扩展点定义的接口如下。

```
// file: kubernetes/pkg/scheduler/framework/interface.go

// QueueSortPlugin is an interface that must be implemented by "QueueSort" plugins.
// These plugins are used to sort pods in the scheduling queue. Only one queue sort
// plugin may be enabled at a time.
type QueueSortPlugin interface {
	Plugin
	// Less are used to sort pods in the scheduling queue.
	Less(*QueuedPodInfo, *QueuedPodInfo) bool
}
```

QueueSortPlugin接口只定义了一个函数Less，所以只需要实现这个函数即可。

```
// Less is the function used by the activeQ heap algorithm to sort pods.
// It sorts pods based on their priorities. When the priorities are equal, it uses
// the Pod QoS classes to break the tie.
func (*Sort) Less(pInfo1, pInfo2 *framework.QueuedPodInfo) bool {
	p1 := corev1helpers.PodPriority(pInfo1.Pod)
	p2 := corev1helpers.PodPriority(pInfo2.Pod)
	return (p1 > p2) || (p1 == p2 && compQOS(pInfo1.Pod, pInfo2.Pod))
}

func compQOS(p1, p2 *v1.Pod) bool {
	p1QOS, p2QOS := v1qos.GetPodQOS(p1), v1qos.GetPodQOS(p2)
	if p1QOS == v1.PodQOSGuaranteed {
		return true
	}
	if p1QOS == v1.PodQOSBurstable {
		return p2QOS != v1.PodQOSGuaranteed
	}
	return p2QOS == v1.PodQOSBestEffort
}
```
3）在程序入口的 main 函数中注册插件
```
func main() {
	// Register custom plugins to the scheduler framework.
	// Later they can consist of scheduler profile(s) and hence
	// used by various kinds of workloads.
	command := app.NewSchedulerCommand(
	    // other plugins ...
		app.WithPlugin(qos.Name, qos.New),
	)

	code := cli.Run(command)
	os.Exit(code)
}
```
4）编辑 KubeSchedulerConfiguration 配置使插件生效

对每个扩展点，可以禁用默认插件或者是启用自己的插件，未显式配置的扩展点则保持默认配置。

另外，kube-scheduler 支持运行多个配置文件。每个配置文件都有一个关联的调度器名称，并且可以在其扩展点中配置一组不同的插件。

比如我们保持 default-scheduler 默认调度策略不变，再增加一个名为 qos-scheduler 的调度器来使用我们刚定制的QoSSort插件，可按以下方式配置。

```
$ cat /etc/sysconfig/kube-scheduler/scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler    // 默认调度器
  - schedulerName: qos-scheduler        // 自定义扩展调度器
    plugins:
      queueSort:
        enabled:
        - name: 'QOSSort'
        disabled:
        - name: '*'
```
在kube-scheduler的启动参数中加入以上配置文件即可，和上述 scheduler-extender 示例中一样。

## 参考文档
- [Kubernetes 调度器](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Kubernetes 调度器配置](https://kubernetes.io/zh-cn/docs/reference/scheduling/)
- [Kubernetes 调度框架](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Kubernetes 调度系统之 Scheduling Framework](https://developer.aliyun.com/article/766998)