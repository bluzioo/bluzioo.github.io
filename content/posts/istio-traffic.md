---
title: "Istio Traffic"
date: 2021-08-24T14:43:15+08:00
draft: true
---


```yaml

api_inner/v?/     # 过渡时用域名sage.dev.netfuse.cn，组件内部鉴权
api_basic/v?/     # 没有鉴权，
api_core/v?/      # 有鉴权，
api/v?/           # openApi,
v?/               # 供前端使用api,  过渡时用域名sage.dev.netfuse.cn


```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: dev-gateway
  namespace: dev-netfuse
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: gateway-vs
  namespace: dev-netfuse
spec:
  hosts:
  - "*"
  gateways:
  - dev-gateway
  http:
  - match:
    - uri:
        prefix: /api_core/v1/
    - uri:
        prefix: /api_inner/v1/config
    route:
    - destination:
        host: hyacinth
        port:
          number: 50100
  - match:
    - uri:
        prefix: /sage
    route:
    - destination:
        host: sage
        port:
          number: 50100
    timeout: 10s
  - match:
    - headers:
        host:
          exact: sage.dev.netfuse.cn
    route:
    - destination:
        host: sage
        port:
          number: 50100
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: plantain-vs
  namespace: dev-netfuse
spec:
  hosts:
  - "plantain.dev.netfuse.cn"
  gateways:
  - dev-gateway
  http:
  - route:
    - destination:
        host: plantain
        port:
          number: 50100
---
```

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
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: sage-entry
  namespace: algo
spec:
  hosts:
    - "sage.dev.netfuse.cn"
  gateways:
    - algo-gateway
  http:
    - route:
        - destination:
            host: sage
            port:
              number: 50100
---
```
