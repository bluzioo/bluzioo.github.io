---
title: "Kubernetes 裸金属环境负载均衡器 - MetalLB"
date: 2021-08-10T15:38:37+08:00
draft: false
keywords: ["loadbalancer", "kubernetes", "k8s", "metallb"]
description: "MetalLB是一款使用标准路由协议的K8S负载均衡器"
tags: ["kubernetes", "loadbalancer"]
categories: ["kubernetes"]
---

## 背景

Kubernetes service对外提供服务的方式有NodePort/ClusterIP/Ingress/LoadBalancer。NodePort以集群内主机绑定随机或自定义端口方式暴露服务，存在单点问题，不会用于生产环境。Ingress基于7层协议，只支持http/https，搭配Ingress Controller再前置LVS-keepalived或LoadBalancer才可对外提供高可用服务。

在公有云的K8S集群通常使用云厂商的LB对外暴露的方式，如AWS的ELB。由于K8S本身不提供LB服务，就需解决自建方式（裸金属环境）的负载均衡器缺失问题。

裸金属环境的负载均衡器开源解决方案：

* MetalLB
* OpenELB （原PorterLB）

以下对MetalLB进行展开

## 概念

MetalLB具备两项功能：address allocation和external announcement

### Address Allocation

MetalLB支持地址分配，前提是配置好IP池。面向内网的集群，即分配好内网IP段，而面向公网的，也可列举可用的公网IP列表。

如果不指定service的spec.loadbalancerIP，metalLB则会从IP池随机分配给LoadBalancer型的service。

### External Announcement

分配了的LB IP对外又是怎么被发现及访问到？MetalLB提供了两种模式：Layer 2和BGP。

在Layer 2模式下，可指定几台集群内主机作为响应LB IP的地址发现（ARP for IPv4, NDP for IPv6）。

在BGP模式下，需要上层路由器支持BGP协议，MetalLB与路由器建立BGP session，更新LB IP的路由信息。

## 部署

### 环境要求

* Kubernetes version > 1.13.0, 集群没有负载均衡器
* 集群网络兼容MetalLB，兼容清单<https://metallb.universe.tf/installation/network-addons/>
* 空闲的IP或IP段可供MetalLB分配
* BGP模式需要支持BGP协议的上层路由器
* 7946端口在节点间开放

### 测试环境

|  组件   | 描述  |
|  ----  | ----  |
|  系统  | centos7  |
|  内核  | 5.13.0-1.el7.elrepo.x86_64  |
| kubernetes  | v1.19.12 |
| cilium  | v1.9.3 |
| metallb  | v0.10.2 |
| master ip  | 192.168.199.77 |
| LB ip pool  | 192.168.199.223-224 |

### 准备

如果k8s集群里使用的是kube-proxy，k8s版本>v1.14.2，需要开启strict ARP，而使用了kube-router作为service proxy，默认已开启。

```bash
kubectl edit configmap -n kube-system kube-proxy

# 编辑设置
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

### 安装

以Manifest安装

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
```

安装后，metallb的资源对象隔离在名为metallb-system的namespace，这时metallb还未能工作，LoadBalancerIP未分配

```bash
[root@k8s-master01 ~]# k get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                     AGE
istio-ingressgateway   LoadBalancer   10.107.221.79   <pending>   15021:31462/TCP,80:32243/TCP,443:31087/TCP,8081:30765/TCP   36d
```

> 除了使用manifests安装，社区还提供helm和kustomize方式安装，详见<https://metallb.universe.tf/installation>

#### 配置Layer 2模式

开启layer2模式，需配置一份configMap，添加至集群

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.199.223-192.168.199.224
```

再查看sevice，已分配到了LoadBalancerIP: 192.168.199.223

```bash
[root@k8s-master01 ~]# k get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                                                     AGE
istio-ingressgateway   LoadBalancer   10.107.221.79   192.168.199.223   15021:31462/TCP,80:32243/TCP,443:31087/TCP,8081:30765/TCP   36d
```

#### 配置BGP模式

开启BGP模式需要上层路由器支持BGP模式，configMap配置如下

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    peers:
    - peer-address: 192.168.199.68
      peer-asn: 64512
      my-asn: 64513
    address-pools:
    - name: default
      protocol: bgp
      addresses:
      - 192.168.199.223-192.168.199.224
```

* peer-address：BGP路由器IP
* peer-asn: 对方自治系统编号
* my-asn: 本集群自治系统编号

> 更多配置参考<https://metallb.universe.tf/configuration>

### 模拟BGP路由器

在没有上层路由器支持BGP协议的情况下，可通过安装模拟路由器来进行测试验证，模拟路由器有BIRD、Guagga，下面以Ggugga为示例安装

```bash
# 确保yum源
yum install quagga

# 关闭selinux
setsebool -P zebra_write_config 1 

# （可选）配置，或直接使用/etc/quagga/zebra.conf，修改其中的hostname为主机名
cp /usr/share/doc/quagga-0.99.22.4/zebra.conf.sample /etc/quagga/zebra.conf

# 启动zebra
systemctl start zebra

# 查看zebra状态
systemctl status zebra
```

配置BGP

```bash
cp /usr/share/doc/quagga-0.99.22.4/bgpd.conf.sample /etc/quagga/bgpd.conf

# 修改/etc/quagga/bgpd.conf
vi /etc/quagga/bgpd.conf
------
router bgp 64512

# 启动
systemctl start bgpd

# 查看bgpd状态
systemctl status bgpd
```

```bash
[root@node68 quagga]# vtysh

Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

node68# conf t 
node68(config)# router bgp 64512
node68(config-router)# no auto-summary  
node68(config-router)# no synchronization  
node68(config-router)# neighbor 192.168.199.77 remote-as 64513
node68(config-router)# neighbor 192.168.199.77 description  "k8s master01"
node68(config-router)# exit
node68(config)# exit
node68# write

#查看BGP建立状态
node68# show ip bgp summary
BGP router identifier 192.168.199.68, local AS number 64512
RIB entries 3, using 336 bytes of memory
Peers 1, using 4560 bytes of memory

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.199.77  4 64513       2       3        0    0    0 00:00:17        2

Total number of neighbors 1


node68# exit
# 查看主机路由信息，看到192.168.199.223和192.168.199.224的路由信息已经添加
[root@node68 ~]# ip route
default via 192.168.199.1 dev eth0
169.254.0.0/16 dev eth0  scope link  metric 1002
192.168.199.0/24 dev eth0  proto kernel  scope link  src 192.168.199.68
192.168.199.223 via 192.168.199.77 dev eth0  proto zebra
192.168.199.224 via 192.168.199.77 dev eth0  proto zebra

# 通过LB ip访问集群
[root@node68 ~]# curl -v 192.168.199.223:80
{"timestamp":"2021-08-13T06:40:22.413+0000","status":404,"error":"Not Found","message":"No message available","path":"/"}

```

## 原理

Metallb包含两个组件，Controller和Speaker，Controller为Deployment部署方式，而Speaker则采用daemonset方式部署到Kubernetes集群各个Node节点。

具体的工作原理如下图所示，Controller负责监听service变化，当service配置为LoadBalancer模式时，从IP池分配给到相应的IP，并进行IP的生命周期管理。Speaker则依据Service的变化，按具体的协议发起相应的广播或应答，根据工作模式（Layer2/BGP)的不同，可采用Leader的方式或负载均衡的方式来响应请求。

当业务流量通过TCP/UDP协议到达指定的Node时，由Node上面运行的Kube-Proxy组件对流量进行处理，并分发到对应的Pod上面。

![metallb-pricinple](/images/metallb-principle.png)

### Layer 2模式

第2层模式下，Metallb会在Node节点中选出一台做为Leader，与服务IP相关的所有流量都会流向该节点。在该节点上， kube-proxy将流量传播到所有服务的Pod，而当leader节点出现故障时，会由另一个节点接管。

局限性：

在二层模式中会存在以下两种局限性：单节点瓶颈以及故障转移慢的情况。

单个leader选举节点接收服务IP的所有流量。这意味着服务的入口带宽被限制为单个节点的带宽，单节点的流量处理能力将成为整个集群的接收外部流量的瓶颈。

在当前的实现中，节点之间的故障转移取决于客户端的合作。当发生故障转移时，MetalLB发送许多2层数据包，以通知客户端与服务IP关联的MAC地址已更改。大多数操作系统能正确处理数据包，并迅速更新其邻居缓存。在这种情况下，故障转移将在几秒钟内发生。在计划外的故障转移期间，在有故障的客户端刷新其缓存条目之前，将无法访问服务IP。对于生产环境如果要求毫秒性的故障切换，目前Metallb可能会比较难适应要求。

### BGP模式

在BGP模式下，群集中的每个节点都与网络路由器建立BGP对等会话，并使用该对等会话通告外部群集服务的IP。假设您的路由器配置为支持多路径，则可以实现真正的负载平衡：MetalLB发布的路由彼此等效。这意味着路由器将一起使用所有下一跳，并在它们之间进行负载平衡。数据包到达节点后，kube-proxy负责流量路由的最后一跳，将数据包送达服务中的一个特定容器。

负载平衡的方式取决于您特定的路由器型号和配置，但是常见的行为是基于数据包哈希值来平衡每个连接，这意味着单个TCP或UDP会话的所有数据包都将定向到群集中的单个计算机。

局限性：

基于BGP的路由器实现无状态负载平衡。他们通过对数据包头中的某些字段进行哈希处理，并将该哈希值用作可用后端数组的索引，将给定的数据包分配给特定的下一跳。
但路由器中使用的哈希通常不稳定，因此，只要后端集的大小发生变化（例如，当节点的BGP会话断开时），现有连接就会被随机有效地重新哈希，这意味着大多数现有连接最终将突然转发到另一后端，而该后端不知道所讨论的连接。

## 参考

* <https://metallb.universe.tf/>
* <https://blog.csdn.net/keith6785753/article/details/107088632/>
* <https://blog.csdn.net/kadwf123/article/details/96838805>
