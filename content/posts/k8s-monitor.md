---
title: "K8s Monitor"
date: 2021-08-31T11:42:38+08:00
draft: true
---


基础设施层:
指容器服务ACK所依赖的底层资源的可观测场景：定位Pod与节点组成的资源池的调用链路，可视化拓扑关系，以及基础设施监控，例如宿主机节点、网络基础组件的性能监控等。

容器性能层
指基于容器服务ACK构建系统的容器抽象层的可观测场景，包括集群的性能、事件等监控，容器的性能，以及容器组件等监控。

应用性能层
指基于容器服务ACK构建系统的具体应用场景，包括应用指标性能（Metric）、系统调用链（Tracing）、日志监控（Logging）等，例如基于容器服务构建一个JAVA应用，JAVA应用的线程数指标等。

用户业务层
基于容器服务ACK构建的业务系统的具体业务场景，例如基于容器服务构建一套高可用可扩展的网站，网站的业务运营数据PV、UV等，例如应用的成本审计场景等。
