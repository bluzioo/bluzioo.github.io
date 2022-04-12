# Harvester 初探


## 概述

Harvester 是基于 Kubernetes 构建的开源超融合基础架构 (HCI) 软件。

何为HCI？参考下 wikipedia 的定义：

```text
Hyperconverged infrastructure (HCI) is a software-defined IT infrastructure that virtualizes all of the elements of conventional "hardware-defined" systems. HCI includes, at a minimum, virtualized computing (a hypervisor), software-defined storage, and virtualized networking (software-defined networking).
```

简单来说，HCI 融合了虚拟化计算，虚拟化存储，虚拟化网络于一体。

Harvester 支持在裸机服务器上实施 HCI。Harvester 使用本地、直接连接的存储，而不是复杂的外部 SANs。它提供了一个可集成启动的镜像进行交付，可以通过 ISO 或 PXE 引导直接部署到服务器上。

## Haveseter 架构

Harvester 基于 k8s 及多个开源项目构建而成。架构如下图：

![harvester-arch](/images/harvester-arch.png)

* Longhorn 是一个轻量级、可靠、易用的 Kubernetes 分布式块存储系统。
* KubeVirt 是一个 Kubernetes 的虚拟机管理插件。
* Elemental for openSUSE Leap 15.3 是一个定制的 Linux 发行版，旨在尽量减少 Kubernetes 集群中节点的操作系统维护工作。

## Havester 实践

Havester 提供了 web 界面操作虚拟机、网络、存储。

* Havester 访问地址：https://{{HARVESTER_VIP}}
* 内嵌的 Rancher 访问地址：https://{{HARVESTER_VIP}}/dashboard/c/local/explorer 
* 内嵌的 Longhorn 访问地址：https://{{HARVESTER_VIP}}/dashboard/c/local/longhorn 

### 初始化工作

#### 新建虚拟机镜像

#### 初始化 vlan 网络

### 创建虚拟机

* 选择创建单个实例或多个实例。
* 输入虚拟机名称（必填）。
* 使用虚拟机模板（可选）。你可以选择 ISO，raw，或 Windows 镜像模板作为默认选项。
* 配置虚拟机的 CPU 和内存。
* 选择 SSH 密钥或上传新密钥。
* 在卷选项卡上选择自定义虚拟机镜像卷。默认磁盘将是根磁盘。你可以向虚拟机添加更多磁盘。
* 如果需要配置网络，前往网络选项卡。默认情况下会添加 Management Network。你也可以使用 VLAN 网络向虚拟机添加辅助网络。请在高级选项 > 网络数据中进行配置。
* 主机名和 cloud-init 数据等高级选项是可选的。你可以在高级选项选项卡中进行配置。

cloud-init 配置默认用户的密码配置示例：

```yaml
# cloud-config
password: password
chpasswd: { expire: False }
ssh_pwauth: True
```

使用 DHCP 进行网络数据配置：

```yaml
version: 1
config:
  - type: physical
    name: eth0
    subnets:
      - type: dhcp
  - type: physical
    name: eth1
    subnets:
      - type: dhcp
```

## KubeVirt

kubevirt 是 Red Hat 开源的以容器方式运行虚拟机的项目，基于 kubernetes 运行，增加资源类型VirtualMachineInstance 等，配以 virt-controller 管理虚拟机的生命周期。

### KubeVirt 架构

![kubevirt-arch](/images/kubevirt-arch.png)

* virt-api：kubevirt是以CRD形式去管理VM Pod，virt-api就是所有虚拟化操作的入口，这里面包括常规的CDR更新验证、以及console、vm start、stop等操作。

* virt-controller：virt-controller会根据vmi CRD，生成对应的virt-launcher Pod，并且维护CRD的状态。与kubernetes api-server通讯监控VMI资源的创建删除等状态。

* virt-handler：virt-handler会以deamonset形式部署在每一个节点上，负责监控节点上的每个虚拟机实例状态变化，一旦检测到状态的变化，会进行响应并且确保相应的操作能够达到所需（理想）的状态。virt-handler还会保持集群级别VMI Spec与相应libvirt域之间的同步；报告libvirt域状态和集群Spec的变化；调用以节点为中心的插件以满足VMI Spec定义的网络和存储要求。

* virt-launcher：每个virt-launcher pod对应着一个VMI，kubelet只负责virt-launcher pod运行状态，不会去关心VMI创建情况。virt-handler会根据CRD参数配置去通知virt-launcher去使用本地的libvirtd实例来启动VMI，随着Pod的生命周期结束，virt-lanuncher也会去通知VMI去执行终止操作；其次在每个virt-launcher pod中还对应着一个libvirtd，virt-launcher通过libvirtd去管理VM的生命周期，这样做到去中心化，不再是以前的虚拟机那套做法，一个libvirtd去管理多个VM。

### 虚拟机创建工作流

虚拟机创建内部流程如下：

![kubevirt-workflow](/images/kubevirt-flow.png)



## Vlan 网络



## Longhorn


Assign Multiple Storage Frontends for Each Volume
Common front-ends include a Linux kernel device (mapped under /dev/longhorn) and an iSCSI target.



2.1、Design
Longhorn 总体设计分为两层: 数据平面和控制平面；Longhorn Engine 是一个存储控制器，对应数据平面；Longhorn Manager 对应控制平面。

2.1.1、Longhorn Manager
Longhorn Manager 使用 Operator 模式，作为 Daemonset 运行在每个节点上；Longhorn Manager 负责接收 Longhorn UI 以及 Kubernetes Volume 插件的 API 调用，然后创建和管理 Volume；

Longhorn Manager 在与 kubernetes API 通信并创建 Longhorn Volume CRD(heml 安装直接创建了相关 CRD，查看代码后发现 Manager 里面似乎也会判断并创建一下)，此后 Longhorn Manager watch 相关 CRD 资源和 Kubernetes 原生资源(PV/PVC…)，一但集群内创建了 Longhorn Volume 则 Longhorn Manager 负责创建物理 Volume。

当 Longhorn Manager 创建 Volume 时，Longhorn Manager 首先会在 Volume 所在节点创建 Longhorn Engine 实例(对比实际行为后发现所谓的 “实例” 其实只是运行了一个 Linux 进程，并非创建 Pod)，然后根据副本数量在所需放置副本的节点上创建对应的副本。

2.1.2、Longhorn Engine
Longhorn Engine 始终与其使用 Volume 的 Pod 在同一节点上，它跨存储在多个节点上的多个副本同步复制卷；同时数据的多路径保证 Longhorn Volume 的 HA，单个副本或者 Engine 出现问题不会影响到所有副本或 Pod 对 Volume 的访问。

下图中展示了 Longhorn 的 HA 架构，每个 Kubernetes Volume 将会对应一个 Longhorn Engine，每个 Engine 管理 Volume 的多个副本，Engine 与 副本实质都会是一个单独的 Linux 进程运行:


注意: 图中的 Engine 并非是单独的一个 Pod，而是每一个 Volume 会对应一个 golang exec 出来的 Linux 进程。







副本 Replicas
每个副本都包含 Longhorn 卷的一系列快照。快照就像镜像(image)的一层，最旧的快照用作基础层，较新的快照在顶部。如果数据覆盖旧快照中的数据，则数据仅包含在新快照中。一系列快照一起显示了数据的当前状态。
Longhorn 副本使用支持精简配置的 Linux sparse files 构建。




Longhorn 云原生容器分布式存储 - 故障排除指南
<https://www.51cto.com/article/686247.html>

pv attach 不上的问题：
重建backing image

配置vlan的虚拟机起不来：
/etc/cni/net.d/00-multus.conf 文件不存在，重启multus pod 恢复


## 参考

云计算底层技术-虚拟网络设备(Bridge,VLAN)
<https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/>

云计算底层技术-虚拟网络设备(tun/tap,veth)
<https://opengers.github.io/openstack/openstack-base-virtual-network-devices-tuntap-veth>

Longhorn 云原生分布式块存储解决方案设计架构和概念
<https://www.51cto.com/article/678246.html>

Longhorn 微服务化存储初试
<https://mritd.com/2021/03/06/longhorn-storage-test>

Longhorn 云原生容器分布式存储 - 故障排除指南
<https://www.51cto.com/article/686247.html>

kubevirt以容器方式运行虚拟机
<https://remimin.github.io/2018/09/14/kubevirt/>
