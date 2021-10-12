---
title: "Loft VCluster 虚拟集群"
date: 2021-09-15T17:21:09+08:00
draft: false
keywords: ["virtual cluster"]
description: "Kubernetes虚拟集群"
tags: ["kubernetes", "k3s"]
categories: ["kubernetes"]
---

## 概述

Kubernets 原生提供的多租户的方案是通过namespace进行的，这种方式是多个用户共享同一个集群和控制平面，工作负载或者应用。如果CRD资源是cluster-scope，要想让不同用户可以使用不同版本的CRD版本，以namespace就无法做到这等隔离性。

vcluster 是运行在Kubernetes集群上的虚拟集群。vcluster虚拟集群拥有自己的控制平面，不过vcluster没有自己的node节点，pod工作负载交由底层的实际k8s集群进行调度。vcluster 的出现解决 namespace 作为租户隔离方案的一些问题，比如：

* cluster-scope类型的资源在集群里是全局的，并不能通过namespace进行隔离，像istio等组件在同一集群里同时运行不同版本是不可行的

* 即使以namesapce隔离，同一集群还是共享这控制平面，如果对控制平面进行配置调整有误的话，将会造成整个集群不可用

vcluster 与其他隔离方案的一个对比参考如下官网的宣传
![vcluster-comparison.png](/download/attachments/31754800/vcluster-comparison.png)

## vcluster 架构

![vcluster-architecture.svg](/download/attachments/31754800/images/vcluster-architecture.svg)

vcluster本身只包含核心 Kubernetes 组件：API 服务器、控制器管理器和存储后端（如 etcd、sqlite、mysql 等）。为了减少虚拟集群开销，vcluster 构建在k3s 上，这是一个完全有效的、经过认证的、轻量级的 Kubernetes 发行版，它将 Kubernetes 组件编译成单个二进制文件并禁用所有不需要的 Kubernetes 功能，例如 pod 调度程序或某些控制器。

除了 k3s 之外，还有一个 Kubernetes hypervisor 替代了 Kubernetes 调度程序，并在虚拟集群中模拟完全工作的 Kubernetes 设置。该组件同步一些对虚拟和主机集群之间的集群功能至关重要的核心资源：

* Pods：所有在虚拟集群中启动的 Pod 都被重写，然后在宿主机集群中以虚拟集群的命名空间中启动。服务帐户令牌、环境变量、DNS 和其他配置被交换以指向虚拟集群而不是主机集群。在 pod 中，pod 似乎是在虚拟集群中而不是主机集群中启动的。

* Service：所有服务和端点都在主机集群中的虚拟集群的命名空间中重写和创建。虚拟集群和主机集群共享相同的服务集群 IP。这也意味着可以从虚拟集群内部访问主机集群中的服务，而不会造成任何性能损失。

* PersistentVolumeClaims：如果在虚拟集群中创建了持久卷声明，它们将在主机集群中的虚拟集群的命名空间中进行变异和创建。如果它们绑定在主机集群中，则相应的持久卷信息将同步回虚拟集群。

* Configmaps & Secrets：挂载到 pod 的虚拟集群中的 ConfigMaps 或 secrets 将同步到主机集群，所有其他 configmaps 或 secrets 将纯粹保留在虚拟集群中。

* 其他资源：部署、状态集、CRD、服务帐户等不同步到主机集群，纯粹存在于虚拟集群中。

除了虚拟和主机集群资源的同步之外，hypervisor 还代理某些 Kubernetes API 请求到主机集群，例如 pod 端口转发或容器命令执行。它本质上充当虚拟集群的反向代理。

## vcluster 实践

### 部署

#### 下载vcluster CLI

```bash
curl -s -L "https://github.com/loft-sh/vcluster/releases/latest" | sed -nE 's!.*"([^"]*vcluster-darwin-amd64)".*!https://github.com\1!p' | xargs -n 1 curl -L -o vcluster && chmod +x vcluster;
sudo mv vcluster /usr/local/bin;
```

#### 创建虚拟集群

官方提供三种方式创建，包括vcluster 客户端、helm、kubectl，为了方便定制配置，以下采用kubectl进行创建。

dev-cluster.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dev-cluster
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-cluster
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets", "services", "services/proxy", "pods", "pods/proxy", "pods/attach", "pods/portforward", "pods/exec", "pods/log", "events", "endpoints", "persistentvolumeclaims"]
    verbs: ["*"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["statefulsets"]
    verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-cluster
subjects:
  - kind: ServiceAccount
    name: dev-cluster
roleRef:
  kind: Role
  name: dev-cluster
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Service
metadata:
  name: dev-cluster
spec:
  type: ClusterIP
  ports:
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
  selector:
    app: dev-cluster
---
apiVersion: v1
kind: Service
metadata:
  name: dev-cluster-headless
spec:
  ports:
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
  clusterIP: None
  selector:
    app: dev-cluster
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: dev-cluster
  labels:
    app: dev-cluster
spec:
  serviceName: dev-cluster-headless
  replicas: 1
  selector:
    matchLabels:
      app: dev-cluster
  template:
    metadata:
      labels:
        app: dev-cluster
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: dev-cluster
      containers:
      - image: rancher/k3s:v1.19.5-k3s2
        name: virtual-cluster
        command:
          - "/bin/k3s"
        args:
          - "server"
          - "--write-kubeconfig=/k3s-config/kube-config.yaml"
          - "--data-dir=/data"
          - "--disable=traefik,servicelb,metrics-server,local-storage"
          - "--disable-network-policy"
          - "--disable-agent"
          - "--disable-scheduler"
          - "--disable-cloud-controller"
          - "--flannel-backend=none"
          - "--kube-controller-manager-arg=controllers=*,-nodeipam,-nodelifecycle,-persistentvolume-binder,-attachdetach,-persistentvolume-expander,-cloud-node-lifecycle"  
          - "--service-cidr=10.96.0.0/12"
        volumeMounts:
          - mountPath: /data
            name: data
      - name: syncer
        image: "loftsh/vcluster:0.3.0"
        args:
          - --service-name=dev-cluster
          - --suffix=dev-cluster
          - --owning-statefulset=dev-cluster
          - --out-kube-config-secret=dev-cluster
          - --tls-san=192.168.199.77
        volumeMounts:
          - mountPath: /data
            name: data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: local-path        
        resources:
          requests:
            storage: 5Gi
```

```bash
kubectl create namespace dev
kubectl apply -f dev-cluster.yaml -n dev
```

#### 连接虚拟集群

```bash
vcluster connect dev-cluster -n dev
```

在当前目录会生产一份kubeconfig.yaml

#### 使用虚拟集群

```bash
# 另启一个终端，进入上一步的目录
export KUBECONFIG=./kubeconfig.yaml

# 查看虚拟集群的namespace列表
kubectl get namespace

# 查看虚拟集群的kube-system下的pod列表
kubectl get pods -n kube-system
```

#### 验证虚拟集群

```bash
kubectl create namespace demo-nginx
kubectl create deployment nginx-deployment -n demo-nginx --image=nginx

kubectl get pods -n demo-nginx
```

而在主集群发生什么？

查看主机群的命名空间， 发现存在先前创建的虚拟集群存在于主集群命名空间

```bash
kubectl get namespaces
```

```bash
NAME               STATUS   AGE
default            Active   11d
dev                Active   9m17s
kube-node-lease    Active   11d
kube-public        Active   11d
kube-system        Active   11d
```

查看在命名空间dev下的deployment，并不存在nginx-deployment，表明vcluster不会同步deployment资源

```bash
kubectl get deployments -n dev
```

```bash
No resources found in dev namespace.
```

查看在命名空间dev下的pods，存在一个名为nginx-deployment-84cd76b964-mnvzz-x-demo-nginx-x-dev-cluster的pod

```bash
kubectl get pods -n host-namespace-1
```

```bash
NAME                                                          READY   STATUS    RESTARTS   AGE
coredns-66c464876b-p275l-x-kube-system-x-dev-cluster           1/1     Running   0          14m
nginx-deployment-84cd76b964-mnvzz-x-demo-nginx-x-dev-cluster   1/1     Running   0          10m
dev-cluster-0                                                  2/2     Running   0          14m
```

要想在外部访问该虚拟集群，需要暴露该vcluster，可以同过 ingress、nodeport、loadbalance，下面以 nodeport为例。

nodeport.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: dev-cluster-nodeport
  namespace: dev
spec:
  selector:
    app: dev-cluster
  ports:
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
  type: NodePort
```

注意在host 集群上下文下进行创建nodeport.yaml

```bash
kubectl apply -f nodeport.yaml
```

```bash
# kubectl get svc -n dev dev-cluster-nodeport

NAME                  TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
dev-cluster-nodeport   NodePort   10.99.72.124   <none>        443:31344/TCP   58m
```

```bash
vcluster connect dev-cluster -n dev --server=https://192.168.199.77:31344

export KUBECONFIG=./kubeconfig.yaml

kubectl get ns
```

## 小结

vcluster的优势如下：

* 轻量级和低开销：它是基于 K3S，捆绑在一个 Pod 中，具有超低的资源消耗。

* 无性能损耗：所有的 Pod 被调度在底层主机集群中，因此它们在运行时不会受到任何性能影响。

* 减少主机集群的开销：将大型多租户集群分割成较小的 vcluster ，以减少复杂性并提高可扩展性。

* 灵活而简单的配置：可以通过 vcluster CLI、Helm、Kubectl、Argo 等任何工具来创建（它基本上只是一个 StatefulSet）。

* 不需要管理权限：如果你能将 Web 应用部署到 Kubernetes 命名空间，你也能部署 Vcluster。

* 单一命名空间封装：每个 vcluster 及其所有的工作负载都在底层主机集群的单一命名空间内。

* 易于清理：删除主机命名空间，Vcluster 及其所有工作负载将立即被清除。

## 参考

* <https://www.vcluster.com/docs>
* <https://liangyuanpeng.com/post/k8s-multi-tenancy-guide/>
