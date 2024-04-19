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

例如一个Filter示例：

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

**运行方式：**

1、生成extender策略配置，主要是配置上述filter的接口地址
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
2、生成 KubeSchedulerConfiguration 配置文件，就是引用上述extender策略配置文件
```
$ cat /etc/sysconfig/kube-scheduler/extender-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/root/.kube/config"
algorithmSource:
  policy:
    file:
      path: "/etc/sysconfig/kube-scheduler/extender-policy.json"
```
3、在kube-scheduler的启动配置中加入以上配置
```
$ /opt/kube/bin/kube-scheduler \
  --config=/etc/sysconfig/kube-scheduler/extender-config.yaml \     # 上述KubeSchedulerConfiguration配置文件
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2
```

#### 3、Scheduling Framework方案
// coming soon

![Kubernetes Scheduler Framework](https://caodailiang.github.io/img/posts/k8s-scheduling-framework-extensions.png)


## 参考文档
- [Kubernetes 调度器配置](https://kubernetes.io/zh-cn/docs/reference/scheduling/)
- [Kubernetes 调度框架](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/scheduling-framework/)
