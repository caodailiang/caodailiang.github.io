---
layout:     post
title:      Kubernetes 多集群管理
date:       2023-04-04
author:     caodailiang
header-img: img/post-bg-coffee.jpeg
catalog: 	true
tags:
    - kubernetes
    - federation
    - clusternet
    - karmada
---

## 多集群背景与内容
为何需要多集群：
- 单个集群节点数量有限
- 研发流程上不同用途
- 可用性要求多云、多地、多中心
- 企业采购或成本方面要求需要使用多家供应商
- 等等

Kubernetes多集群管理主要解决三类问题：
- 多Kubernetes集群管理
- 多集群中workload管理
- 多集群中Service通信

## 多集群开源项目
#### KubeFed v2

#### Clusternet
![Clusternet-Architecture](https://caodailiang.github.io/img/posts/k8s-multi-cluster-clusternet-arch.png)
Clusternet 是由腾讯开源的一个轻量级插件，由clusternet-agent、clusternet-scheduler和clusternet-hub三个组件组成。

clusternet-agent 负责：
- 自动将当前集群注册到父集群作为子集群，也称为ManagedCluster
- 报告当前集群的心跳，包括Kubernetes版本、运行平台、healthz/readyz/livez、状态等
- 建立一个 websocket 连接，它通过单个 TCP 连接来提供全双工通信通道到父集群

clusternet-scheduler 负责：
- 基于SchedulingStrategy来调度资源/feeds到匹配的子集群

clusternet-hub 负责：
- 批准集群注册请求并为每个子集群创建专用资源，例如命名空间、服务帐户和 RBAC 规则
- 作为aggregated apiserver (AA) 服务。提供 shadow APIs，并用作 websocket 服务器，来维护子集群的多个活动 websocket 连接
- 提供 Kubernstes 风格的 API， 将请求重定向/代理/升级到每个子集群
- 利用 API， 协调和部署应用程序到多个集群

Clusternet核心逻辑类似于一个代理多个Kubernetes集群和兼容多个不同Kubernetes版本的适配层服务，clusternet中AA的作用，就是把k8s原生的api转换为AA中的映射api，例如pod存在apis/shadow/v1alpha1/pods下。所以客户端必须要进行改造，从原来使用k8s原生的api，变为使用clusternet专用的api。

#### Karmada
Karmada 是一个部署在k8s中的定制化的k8s（k8s-on-k8s），这个k8s唯一的作用就是部署 Karmada 的k8s元集群，用户工作负载部署运行在其它子集群中，Karmada只对外暴露元集群apiserver的API。

Karmada 的总体架构如下所示：

![Karmada-Architecture](https://caodailiang.github.io/img/posts/k8s-multi-cluster-karmada-arch.png)

Karmada 控制平面包括以下组件：
- Karmada API Server：直接使用 Kubernetes 的 kube-apiserver 实现的，对外暴露 Karmada API 以及 Kubernetes 原生API
- Karmada Controller Manager：负责监视Karmada对象，并与底层集群的API服务器通信，以创建原生的 Kubernetes 资源。
- Karmada Scheduler：负责将 Kubernetes 原生API资源对象（以及CRD资源）调度到成员集群，调度器依据策略约束和可用资源来确定哪些集群对调度队列中的资源是可用的，然后调度器对每个可用集群进行打分排序，并将资源绑定到最合适的集群。

ETCD 存储了 karmada API 对象，API Server 是所有其他组件通讯的 REST 端点，Karmada Controller Manager 根据您通过 API 服务器创建的 API 对象执行操作。

Karmada Controller Manager 在管理面运行各种 Controller，这些 Controller 监视 karmada 对象，然后与成员集群的 API Server 通信以创建常规的 Kubernetes 资源。 
1. Cluster Controller：将 Kubernetes 集群连接到 Karmada，通过创建集群对象来管理集群的生命周期。
2. Policy Controller：监视 PropagationPolicy 对象。当添加 PropagationPolicy 对象时，Controller 将选择与 resourceSelector 匹配的一组资源，并为每个单独的资源对象创建 ResourceBinding。
3. Binding Controller：监视 ResourceBinding 对象，并为每个带有单个资源清单的集群创建一个 Work 对象。
4. Execution Controller：监视 Work 对象。当创建 Work 对象时，Controller 将把资源分发到成员集群。

## Karmada 多集群服务治理
#### 多集群网络
Karmada 可使用 Submariner 实现成员集群彼此联网，Submariner 将相连集群之间的网络扁平化，并实现 Pod 和服务之间的 IP 可达性。

Submariner 架构：

![Submariner-Architecture](https://caodailiang.github.io/img/posts/k8s-multi-cluster-submariner-arch.jpg)

Submariner 包括下面几个重要组件 :
- Broker：没有实际的 Pod 和 Service，本质上就是两个用于交换集群信息的 CRD（Endpoint 和 Cluster）,我们需要选择一个集群作为 Broker 集群，其他集群连接到 Broker 集群的 API Server 来交换集群信息。
- Gateway Engine：建立和维护集群间的隧道，打通跨集群的网络通讯。
- Route Agent：在 Gateway 节点和工作节点之间建立 Vxlan 隧道，使工作节点上的跨集群流量先转发到 Gateway 节点，再通过跨集群的隧道从 Gateway 节点发送至对端。
- Service Discover：包括 Lighthouse-agent 和 Lighthouse-dns-server 组件，实现 KMCS API，提供跨集群的服务发现。
- Submariner Operator：负责在 Kubernetes 集群中安装 Submariner 组件，例如 Broker, Gateway Engine, Route Agent 等等。

可选组件：
- Globalnet Controller：支持重叠子网的集群间互连。

简单来说，Submariner 由一个集群元数据中介服务（broker）掌握不同集群的信息（Pod/Service CIDR），通过 Route Agent 将 Pod 流量从 Node 导向网关节点（Gateway Engine），然后由网关节点打通隧道丢到另一个集群中去，这个过程就和不同主机的容器之间使用 VxLAN 网络通信的概念是一致的。 要达成集群连接也很简单，在其中一个集群部署 Broker，然后通过 kubeconfig 或 context 分别进行注册即可。

#### 多集群服务发现
Karmada 可使用 ServiceExport 和 ServiceImport，实现跨集群的服务发现。

在karmada控制平面上安装完 ServiceExport 和 ServiceImport 之后，再创建 ClusterPropagationPolicy 来分发这两个 CRD 到成员集群。

![ErieCanal Architecture](https://caodailiang.github.io/img/posts/k8s-multi-cluster-eriecanal.png)

#### 多集群服务治理
可使用 ErieCanal 完成 Karmada 多集群的互联互通，实现跨集群的服务治理。

ErieCanal 是一个 MCS（多集群服务 Multi-Cluster Service）实现，为 Kubernetes 集群提供 MCS、Ingress、Egress、GatewayAPI。

## 参考文档
- [浅谈开源集群联邦的设计和实现原理](https://cvvz.fun/post/kube-federation/)
- [Clusternet：一款开源的跨云多集群云原生管控利器！](https://juejin.cn/post/7056585357164281886)
- [Kubernetes多集群社区方案介绍](https://www.ctyun.cn/developer/article/430438379012165)
- [Kubernetes 多集群网络方案系列 1 -- Submariner 介绍](https://www.cnblogs.com/ztguang/p/17959579)
- [KubeFed: Kubernetes Federation v2 详解](https://www.kubernetes.org.cn/5702.html)