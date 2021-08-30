---
title: "Istio初探"
date: 2021-08-26T11:35:08+08:00
draft: false
keywords: ["istio"]
description: "Istio服务网格"
tags: ["istio"]
categories: ["istio"]
---

## 概述

回望应用架构的演进，单体 -> SOA -> 微服务，不同的架构在一个企业发展的不同时期都有重要的意义。

在微服务架构中，一个应用程序基于领域驱动设计分解成不同的小服务，服务间的通信使用HTTP Rest、gRPC等协议，为满足服务治理等需求，微服务中间件都会提供不同语言的SDK供服务组件集成。

微服务架构优势在于模块化，扩展性，分布式等，然而在渐渐的落地实践时，也暴露出了一些问题。微服务SDK囊括了服务注册发现，负载均衡，熔断，限流等功能，与业务应用耦合在一个可执行程序，这在SDK需要升级时，本与业务无关的需求，业务应用也需进行发布升级。同时，因服务的异构性，对不同语言的SDK进行维护也是一种煎熬。于是，为了使服务间的通信进一步与业务应用解耦，催生了服务网格（Service Mesh）。

## 服务网格

Service Mesh 一词最早由开发 Linkerd 的 Buoyant 公司提出，Willian Morgan，Buoyant CEO 对 Service Mesh 的解释：
> Service Mesh 是一个专门处理服务通讯的基础设施层。它的职责是在由云原生应用组成服务的复杂拓扑结构下进行可靠的请求传送。在实践中，它是一组和应用服务部署在一起的轻量级的网络代理，并且对应用服务透明。

简单地说，假设把服务网格比作是应用程序或者说微服务间的 TCP/IP，负责服务之间的网络调用、限流、熔断和监控。对于编写应用程序来说一般无须关心 TCP/IP 这一层（比如通过 HTTP 协议的 RESTful 应用），同样使用 Service Mesh 也就无须关心服务之间的那些原本通过服务框架实现的事情，比如 Spring Cloud、Netflix OSS 和其他中间件，现在只要交给 Service Mesh 就可以了。

服务网格的需求包括服务发现、负载均衡、故障恢复、度量和监控等。服务网格通常还有更复杂的运维需求，比如 A/B 测试、金丝雀发布、速率限制、访问控制和端到端认证。

目前两款流行的 Service Mesh 开源软件 Istio 和 Linkerd 都可以直接在 Kubernetes 中集成，其中 Linkerd 是最早开源的，已经成为 CNCF 中的项目，而Istio 是由 Google、IBM、Lyft 等共同开源的 Service Mesh 框架，于2017年开源。

## Istio 简介

Istio 可以轻松地创建一个已经部署了服务的网络，而服务的代码只需很少更改甚至无需更改。通过在整个环境中部署一个特殊的 sidecar 代理为服务添加 Istio 的支持，而代理会拦截微服务之间的所有网络通信，然后使用其控制平面的功能来配置和管理 Istio，这包括：

* 为 HTTP、gRPC、WebSocket 和 TCP 流量自动负载均衡。
* 通过丰富的路由规则、重试、故障转移和故障注入对流量行为进行细粒度控制。
* 可插拔的策略层和配置 API，支持访问控制、速率限制和配额。
* 集群内（包括集群的入口和出口）所有流量的自动化度量、日志记录和追踪。
* 在具有强大的基于身份验证和授权的集群中实现安全的服务间通信。

Istio 为可扩展性而设计，可以满足不同的部署需求。

## Istio 架构

Istio 服务网格从逻辑上分为数据平面和控制平面。

* 数据平面由一组智能代理（Envoy）组成，被部署为 Sidecar。这些代理负责协调和控制微服务之间的所有网络通信。它们还收集和报告所有网格流量的遥测数据。
* 控制平面管理并配置代理来进行流量路由。

下图展示了组成每个平面的不同组件：

![istio-arch](/images/istio-arch.svg)

### 数据平面

Istio 在数据平面使用 Envoy 代理的扩展版本。Envoy 是用 C++ 开发的高性能代理，用于协调服务网格中所有服务的入站和出站流量。Envoy 代理是唯一与数据平面流量交互的 Istio 组件。

#### Envoy 架构

![envoy-arch](/images/envoy-arch.png)

Envoy 接收到请求后，会先走 FilterChain，通过各种 L3/L4/L7 Filter 对请求进行微处理，然后再路由到指定的集群，并通过负载均衡获取一个目标地址，最后再转发出去。

其中每一个环节可以静态配置，也可以动态服务发现，也就是所谓的 xDS。

#### xDS-Api

Envoy xDS 为 Istio 控制平面与数据平面通信的基本协议，只要代理支持该协议表达形式就可以创建自己 Sidecar 来替换 Envoy。

Envoy中，提供的xDS-API列举如下：

* SDS/EDS(Service/Endpoint(v2) Discovery Service): 节点发现服务，针对的是那些提供服务的节点(可能是一个pod 或者其他),让节点可以聚合成服务的方式提供给调用方。V2版本API中，Service 升级为Endpoint.
* CDS(Cluster Discovery Service):集群发现服务，集群指的是Envoy接管的集群。Istio可以利用这个接口创建虚拟机集群，例如一个应用可以划分不同版本的部署结构。
* RDS(Route Discovery Service): 路由规则发现服务，路由规则的作用就是动态转发，基于此可以实现请求漂移，蓝绿发布等。
* LDS(Listener Discovery Service): 监听器发现服务，监听器主要作用于Envoy的链接状态，如链接总数，活动的连接数等。

### 控制平面

Istiod 提供服务发现、配置和证书管理。

Istiod 将控制流量行为的高级路由规则转换为 Envoy 特定的配置，并在运行时将其传播给 Sidecar。Pilot 提取了特定平台的服务发现机制，并将其综合为一种标准格式，任何符合 Envoy API 的 Sidecar 都可以使用。

#### Pilot

![pilot-arch](/images/pilot-arch.png)

为了实现对不同服务注册中心 （Kubernetes、consul） 的支持，Pilot 需要对不同的输入来源的数据有一个统一的存储格式，也就是抽象模型。

Pilot 的实现是基于平台适配器（Platform adapters） 的，借助平台适配器 Pilot 可以实现服务注册中心数据到抽象模型之间的数据转换

Pilot 使用了一套起源于 Envoy 项目的标准数据面 API 来将服务信息和流量规则下发到数据面的 sidecar 中。通过采用该标准 API， Istio 将控制面和数据面进行了解耦，为多种数据平面 sidecar 实现提供了可能性。

## Istio 实战

### 部署

### 流量管理

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: sage
  namespace: algo
spec:
  host: sage    # The name of a service from the service registry       
  subsets:
    - name: v1
      labels:
        app.kubernetes.io/name: sage
  trafficPolicy:    # Traffic policies to apply (load balancing policy, connection pool sizes, outlier detection)
    tls:
      mode: ISTIO_MUTUAL
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
      tcp:
        maxConnections: 1
    outlierDetection:
      baseEjectionTime: 180.000s
      consecutiveErrors: 1
      interval: 1.000s
      maxEjectionPercent: 100
       
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: sage
  namespace: algo
spec:
  hosts:
    - sage
  http:
    - route:
      - destination:
          host: sage
          subset: v1
    - route:                # 默认路由，针对没有匹配到的路由规则
      - destination:
          host: sage
          subset: v1
---
```

### 追踪

```yaml
spec:
  components:
    base:
      enabled: true
    cni:
      enabled: false
    egressGateways:
    - enabled: false
      name: istio-egressgateway
    ingressGateways:
    - enabled: true
      name: istio-ingressgateway
    - enabled: true
      name: dev-netfuse-gateway
      namespace: dev-netfuse
    istiodRemote:
      enabled: false
    pilot:
      enabled: true
      k8s:
        env:
        - name: PILOT_TRACE_SAMPLING
          value: "100"
```

### 安全

### JWT Auth

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-example
  namespace: algo
spec:
  jwtRules:
  - issuer: cacao
    jwks: "{ \"keys\":[{ \n  \"kty\" : \"oct\",\n  \"kid\" : \"0afee142-a0af-4410-abcc-9f2d44ff45b5\",\n
      \ \"alg\" : \"HS512\",\n  \"k\"   : \"d2p5bEA4ODg4\"\n}]}\n"
  selector:
    matchLabels:
      app.kubernetes.io/name: sage
```

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: algo
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals:
        - '*'
  selector:
    matchLabels:
      app.kubernetes.io/name: sage
```

## 场景应用

### 金丝雀

### 故障注入

### 错误诊断

## 参考

* <https://istio.io/latest/docs/>
* <https://www.envoyproxy.io/docs/envoy/latest/>
* <https://jimmysong.io/>
* <https://www.servicemesher.com/>
* <https://zhuanlan.zhihu.com/p/369068128>
