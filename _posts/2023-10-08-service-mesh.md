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

## Service Mesh 模式演变
#### Sidecar 模式

#### Proxyless 模式

#### Ambient Mesh 模式

#### Sidecarless 模式

## istio 技术架构

[Istio](https://istio.io/zh) 是由 Google、IBM、Lyft 等共同开源的 Service Mesh（服务网格）框架，于2017年初开始进入大众视野，作为云原生时代下承 Kubernetes、上接 Serverless 架构的重要基础设施层，地位至关重要。

Istio 服务网格从逻辑上分为 **数据平面** 和 **控制平面**。

**数据平面** 由一组被部署为 Sidecar 的智能代理（Envoy） 组成。这些代理负责协调和控制微服务之间的所有网络通信。它们还收集和报告所有网格流量的遥测数据。

**控制平面** 管理并配置代理来进行流量路由。

下图展示了组成每个平面的不同组件：

![istio-mesh](https://caodailiang.github.io/img/posts/service-mesh-istio-arch.svg)