---
title: "Kubernetes裸金属环境负载均衡器-MetalLB"
date: 2021-08-10T15:38:37+08:00
draft: false
keywords: ["loadbalancer", "kubernetes", "k8s", "metallb"]
description: "MetalLB是一款使用标准路由协议的K8S负载均衡器"
tags: ["kubernetes"]
categories: ["kubernetes"]
author: "Bluz Mao"
---

## 概述

Kubernetes service对外提供服务的方式有NodePort/ClusterIP/Ingress/LoadBalancer。NodePort以集群内主机绑定随机或自定义端口方式暴露服务，存在单点问题，不会用于生产环境。Ingress基于7层协议，只支持http/https，搭配Ingress Controller再前置LVS-keepalived或LoadBalancer才可对外提供高可用服务。
在公有云的K8S集群通常使用云厂商的LB对外暴露的方式，如AWS的ELB。由于K8S本身不提供LB服务，就需解决自建方式（裸金属环境）的负载均衡器缺失问题。

裸金属环境的负载均衡器开源解决方案：

* MetalLB
* OpenELB （原PorterLB）

以下对MetalLB进行展开。

## 基本概念

MetalLB具备两项功能：address allocation和external announcement

### Address Allocation

MetalLB支持地址分配，前提是配置好IP池。面向内网的集群，即分配好内网IP段，而面向公网的，也可列举可用的公网IP列表。
如果不指定service的spec.loadbalancerIP，metalLB则会从IP池随机分配给LoadBalancer型的service。

### External Announcement

分配了的LB IP对外又是怎么被发现及访问到？MetalLB提供了两种模式：Layer 2和BGP。

在Layer 2模式下，可指定几台集群内主机作为响应LB IP的地址发现（ARP for IPv4, NDP for IPv6）。
在BGP模式下，需要上层路由器支持BGP协议，MetalLB与路由器建立BGP session，更新LB IP的路由信息。

## 部署验证
