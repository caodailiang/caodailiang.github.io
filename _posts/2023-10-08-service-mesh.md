---
layout:     post
title:      Service Mesh 与 istio 技术简介
date:       2023-10-08
author:     caodailiang
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - service-mesh
    - istio
---

## Service Mesh 是什么
服务网格（ServiceMesh）是一个**基础设施层**，用于处理服务间通信。云原生应用有着复杂的服务拓扑，服务网格保证**请求在这些拓扑中可靠地穿梭**。在实际应用当中，服务网格通常是由一系列轻量级的**网络代理**组成的，它们与应用程序部署在一起，但**对应用程序透明**。

典型的服务网格是在每台节点上同时运行着业务逻辑和代理，这个代理被形象地称为 Sidecar。服务之间通过 Sidecar 发现和调用目标服务。

![service-mesh-sidecar](https://caodailiang.github.io/img/posts/service-mesh-sidecar.png)

只看单机代理组件和控制面板的服务网格全局部署视图如下，在服务之间形成一种网络状依赖关系，这也是服务网格如此形象命名的由来。

![service-mesh](https://caodailiang.github.io/img/posts/service-mesh.png)

## Service Mesh 模式演变
#### Sidecar 模式

典型的服务网格的核心是 Sidecar，Sidecar 本质上是一个服务代理，通过劫持发送到应用容器的流量从而实现对流量的控制，随着服务网格落地实践，Sidecar 的缺点也逐渐被暴露：
- **延迟问题**：Sidecar 常规的做法是使用 iptbales 实现请求的拦截，在原本的请求链路中加入iptables和sidecar，必然导致延迟增加和性能损耗。
- **资源占用问题**：Sidecar 作为一个独立的进程或容器必然会占用一定的系统资源，对于超大规模集群来说，巨大的基数使得 Sidecar 占用资源总量变成了不小的数目。同时，这类集群的网络通信拓扑也更加复杂，配置下发的规模也会让 Sidecar 占用的内存剧烈的增长。
- **Sidecar升级问题**：Sidecar伴生在服务进程或容器旁，每个服务进程或容器伴随着一个Sidecar，在超大规模集群下Sidecar升级变得特别困难。

#### Proxyless 模式

基于Sidecar模式存在的问题，Proxyless模式的核心就是把 Sidecar 去掉。

服务间总是要选择一种协议进行通信，就必然要依赖于该协议的类库（SDK）进行编解码工作。Proxyless 理念是将协议的类库扩展，使其具有流量控制的能力。且 SDK 和应用同属于同一进程，必然有更优秀的性能表现，Sidecar 最为诟病的延迟问题也会迎刃而解。

在这种模式中，服务网格核心的流控能力被集成在 gRPC 库中，不再使用代理进行数据面通信。但这种方案仍然需要一个 Agent 进行初始化并与控制平面交互。

例如 Istio 官方博客发表的一篇基于 gRPC 实现的 Proxyless 方案。

![service-mesh-proxyless](https://caodailiang.github.io/img/posts/service-mesh-proxyless.svg)

这个方案下，所谓 Proxyless 其实和传统的 SDK 并无二致，只是将流控能力内嵌到负责通信协议的类库中，因此它具有和传统 SDK 服务框架相同的缺点，也正因为如此，业内很多人认为 Proxyless 本质上是一种倒退，是回归到传统的方式去解决服务通信的问题。

#### Ambient Mesh 模式

2022 年 9 月 Istio 发布了一个名为 “Ambient Mesh” 的数据平面模型。Ambient Mesh 使用了共享代理而不是 Sidecar，简单地讲就是将数据面的代理从应用 Pod 中剥离出来独立部署，以彻底解决 mesh 基础设施和应用部署耦合的问题。

在之前的模式中，Istio 在单一的架构组件 sidecar 中实现了从基本的加密到高级的 L7 策略的所有数据平面功能。Ambient mesh 则将 Istio 的功能分成两个不同的层次。在底层，有一个 `安全覆盖层（ztunnel）` 来处理流量的路由和零信任安全。在这之上，当需要时，用户可以通过启用 `L7 处理（waypoint）` 来获得 Istio 的全部能力。L7 处理模式虽然比安全覆盖层更重，但仍然作为基础设施的一个 ambient 组件运行，不需要对应用 pod 进行修改。

![ambient-layers](https://caodailiang.github.io/img/posts/service-mesh-ambient-layers.png)

`Ztunnel` 实现了一个服务网格的核心功能：零信任。当为一个 namespace 启用 ambient 时，Istio 会创建一个安全覆盖层(secure overlay)，该安全覆盖层为工作负载提供 mTLS, 遥测和认证，以及 L4 权限控制，并不需要中断 HTTP 链接或者解析 HTTP 数据。

![ambient-secure-overlay](https://caodailiang.github.io/img/posts/service-mesh-ambient-secure-overlay.png)

在启用 ambient mesh 并创建安全覆盖层后，一个 namepace 也可以配置使用 L7 的相关特性。这样可以在一个 namespae 中提供完整的 Istio 功能，包括 Virtual Service API、L7 遥测 和 L7授权策略。以这种模式运行的 namespace 使用一个或多个基于 Envoy 的 `waypoint proxy`（waypoint 意味路径上的一个点） 来为工作负载进行 L7 处理。Istio 控制平面会配置集群中的 ztunnel，将所有需要进行 L7 处理的流量发送到 waypoint proxy。

![ambient-waypoint](https://caodailiang.github.io/img/posts/service-mesh-ambient-waypoint.png)

#### Sidecarless 模式

2022 年 Cilium 基于 eBPF 技术发布了具有服务网格能力的产品。

Cilium 的服务网格产品提供了两种模式，对于 L3/L4 层的能力直接由 eBPF 支持，L7 层能力由一个公共的代理负责，以 DaemonSet 方式部署。

![sidecarless](https://caodailiang.github.io/img/posts/service-mesh-sidecarless.png)

基于 eBPF 的服务网格在设计思路上其实和 Proxyless 类似，即找到一个非 Sidecar 的地方去实现流量控制能力，它们一个是基于通信协议类库，一个是基于内核的扩展性。eBPF 通过内核层面提供的可扩展能力，在流量经过内核时实现了控制、安全和观察的能力，从而构建出一种新形态的服务网格。

![sidecarless-ebpf](https://caodailiang.github.io/img/posts/service-mesh-sidecarless-ebpf.webp)

## istio 技术架构

[Istio](https://istio.io/zh) 是由 Google、IBM、Lyft 等共同开源的 Service Mesh（服务网格）框架，于2017年初开始进入大众视野，作为云原生时代下承 Kubernetes、上接 Serverless 架构的重要基础设施层，地位至关重要。

Istio 服务网格从逻辑上分为 **数据平面** 和 **控制平面**。

- **数据平面**通常采用轻量级的代理（例如 Envoy）作为 Sidecar，这些代理负责协调和控制服务之间的通信和流量处理。

- **控制平面**负责配置和管理数据平面，并提供服务发现、智能路由、流量控制、安全控制等功能。

下图展示了组成每个平面的不同组件：

![istio-mesh](https://caodailiang.github.io/img/posts/service-mesh-istio-arch.svg)

**数据平面组件：**

Istio 使用 `Envoy` 代理的扩展版本，该代理是以 C++ 开发的高性能代理，用于调解服务网格中所有服务的所有入站和出站流量。

`Envoy` 代理被部署为服务的 sidecar，在逻辑上为服务增加了 Envoy 的许多内置特性，例如:

- 动态服务发现
- 负载均衡
- TLS 终端
- HTTP/2 与 gRPC 代理
- 熔断器
- 健康检查
- 基于百分比流量分割的分阶段发布
- 故障注入
- 丰富的指标

这种 Sidecar 部署允许 Istio 可以执行策略决策，并提取丰富的遥测数据， 接着将这些数据发送到监视系统以提供有关整个网格行为的信息。

Sidecar 代理模型还允许您向现有的部署添加 Istio 功能，而不需要重新设计架构或重写代码。

由 `Envoy` 代理启用的一些 Istio 的功能和任务包括：

- 流量控制功能：通过丰富的 HTTP、gRPC、WebSocket 和 TCP 流量路由规则来执行细粒度的流量控制。
- 网络弹性特性：重试设置、故障转移、熔断器和故障注入。
- 安全性和身份认证特性：执行安全性策略，并强制实行通过配置 API 定义的访问控制和速率限制。
- 基于 WebAssembly 的可插拔扩展模型，允许通过自定义策略执行和生成网格流量的遥测。

**控制平面组件**

控制平面组件 `Istiod` 提供服务发现、配置和证书管理，包含3个部分：

- `Pilot`：从上（如 Kubernetes）获取服务信息，完成服务发现，往下（Proxy）下发流量管理以及路由规则等 xDS 配置，驱动数据面按照规则实现流量管控（A/B测试、灰度发布）、弹性（超时、重试、熔断）、调试（故障注入、流量镜像）等功能。
- `Citadel`：充当证书颁发机构（CA），负责身份认证和证书管理，可提供服务间和终端用户的身份认证，实现数据平面内安全的 mTLS 通信。
- `Galley`：负责将其他 Istio 组件和底层平台（Kubernetes）进行解耦，负责配置获取、处理和分发组件。

## 参考文档
- [服务网格概论](https://www.thebyte.com.cn/ServiceMesh/summary.html)
- [istio 架构](https://istio.io/latest/zh/docs/ops/deployment/architecture/)
- [gRPC Proxyless Service Mesh](https://istio.io/latest/blog/2021/proxyless-grpc/)
- [Introducing Ambient Mesh](https://istio.io/latest/blog/2022/introducing-ambient-mesh/)