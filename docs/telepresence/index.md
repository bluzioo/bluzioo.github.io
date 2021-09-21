# Telepresence


## 概述

在对微服务进行容器化部署至 Kubernetes 这个过程属于持续交付阶段，已经通过 Argo CD 等交付工具解决。然而，如果我们要在本地开发调试一个服务 A，但服务 A 可能依赖服务B、C，而服务 B 又有一层依赖 D，我们就需要在本地把服务 B、C、D 都搭建起来才能调试服务 A。这显然是一个很痛苦的过程。

Telepresence 是一个 CNCF 基金会下的项目。它的工作原理是在本地和 Kubernetes 集群中搭建一个透明的双向代理，这使得我们可以在本地用熟悉的 IDE 和调试工具来运行一个微服务，同时该服务还可以无缝的与 Kubernetes 集群中的其他服务进行交互，好像它就运行在这个集群中一样。

## Telepresence 实践

### 安装

在本地环境安装，参考 <https://www.telepresence.io/docs/latest/install/>

### 使用场景

以 sage 服务为例，在 ARGO CD 配置 sage 服务部署至 Kubernetes 开发测试环境，查看 sage 服务是否运行。

```bash
[root@k8s-master01 ~]# kubectl get pod -n wjyl-dev
NAME                    READY   STATUS    RESTARTS   AGE
sage-aaa5686745-xxft9   2/2     Running   1          7h38m
```

此时 sage pod 有两个 container 一个是 sage 服务，另一个是 istio-proxy。请求下测试接口，显示的数据是 "from k8s"

```bash
[root@k8s-master01 ~]# curl -v -H "Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJpc3MiOiJjYWNhbyIsInN1YiI6ImF5Zm9vZCIsImF1ZCI6IndlYiIsImlhdCI6MTYzMTUxOTAzNiwiZXhwIjoxNjMxNTYyMjM2LCJvcmdfdW5pdCI6InRlbmFudCJ9.OEGQ5LIzfAFXyaSUuU4ZS1koaJ0ufAJK-j97fjhqpRf5j5YqCugbYW37Je-I6dfPdMQQwlXK-WlpU3MCxmM-lg" sage.dev.netfuse.cn/v1/usage/test
{"code":"0","msg":"","date":1632237007295,"data": "from k8s"}
```

在本地开发环境，运行 telepresence connect 连接至 kubernetes 开发测试环境。

```bash
$ telepresence connect
Launching Telepresence Root Daemon
Need root privileges to run: /usr/local/bin/telepresence daemon-foreground /Users/bluz/Library/Logs/telepresence '/Users/bluz/Library/Application Support/telepresence' ''
Password:
Launching Telepresence User Daemon
Connected to context dev01-context (https://192.168.199.77:6443)
```

此时 Kubernetes 集群会部署 telepresence 流量管理的服务。

```bash
[root@k8s-master01 ~]# kubectl get pod -n ambassador
NAME                               READY   STATUS    RESTARTS   AGE
traffic-manager-85d645d956-6vswr   1/1     Running   1          3d23h
```

使用 telepresence 拦截集群中 sage 服务的流量至本地开发环境。

```bash
$ telepresence intercept sage --port 10519:50100 -n wjyl-dev
Using Deployment sage
intercepted
    Intercept name         : sage-wjyl-dev
    State                  : ACTIVE
    Workload kind          : Deployment
    Destination            : 127.0.0.1:10519
    Service Port Identifier: 50100
    Volume Mount Error     : sshfs is not installed on your local machine
    Intercepting           : all TCP connections

```

修改本地测试接口返回的数据为 “from local”，以 debug 模式启动 sage 服务，监听端口为 10519。再次请求测试接口，显示的数据即是本地修改 "from local"。在 IDE 设置断点即可调试。

```bash
[root@k8s-master01 ~]# curl -H "Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJpc3MiOiJjYWNhbyIsInN1YiI6ImF5Zm9vZCIsImF1ZCI6IndlYiIsImlhdCI6MTYzMTUxOTAzNiwiZXhwIjoxNjMxNTYyMjM2LCJvcmdfdW5pdCI6InRlbmFudCJ9.OEGQ5LIzfAFXyaSUuU4ZS1koaJ0ufAJK-j97fjhqpRf5j5YqCugbYW37Je-I6dfPdMQQwlXK-WlpU3MCxmM-lg" sage.dev.netfuse.cn/v1/usage/test
{"code":"0","msg":"","date":1632237007295,"data": "from local"}
```

## 参考

* <https://www.telepresence.io/docs/latest/quick-start/>
* <https://jimmysong.io/blog/how-to-debug-microservices-in-kubernetes-with-proxy-sidecar-or-service-mesh/>
* <https://zhuanlan.zhihu.com/p/106051607>

