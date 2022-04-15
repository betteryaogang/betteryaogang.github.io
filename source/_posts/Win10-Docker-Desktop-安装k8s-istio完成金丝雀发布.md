title: Win10 +Docker Desktop 安装k8s istio完成金丝雀发布
author: betteryaogang
tags:
  - docker
  - k8s
  - istio
  - kubernetes
  - devops
categories:
  - kubernetes
  - istio
  - service mesh
date: 2022-03-28 17:52:00
---
### 调用链路说明
调用链路为：`客户端`-> `itsio gateway` -> `a服务` - > `b服务`

其中：

- a服务只有v1版本，b服务包含v1和v2版本，a服务内部会调用b服务。
- 金丝雀发布要求部分指定header的请求走v2版本，其余请求走v1版本。


**1. 部署几个基础服务 **

demo.yaml文件

```
apiVersion: v1
kind: Service
metadata:
  name: svc-b
  labels:
    app: demo-b
    service: svc-b
spec:
  ports:
  - port: 8080
    name: http
    targetPort: 8080
    protocol: TCP
  selector:
    app: demo-b
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-b-v1
  labels:
    app: demo-b
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-b
      version: v1
  template:
    metadata:
      labels:
        app: demo-b
        version: v1
    spec:
      containers:
      - name: demo-b
        image: yaogang/demo-b-v1:v1.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
      restartPolicy: Always
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-b-v2
  labels:
    app: demo-b
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-b
      version: v2
  template:
    metadata:
      labels:
        app: demo-b
        version: v2
    spec:
      containers:
      - name: demo-b-v1
        image: yaogang/demo-b-v2:v1.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
      restartPolicy: Always
---
##################################################################################################
# demo-a
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: svc-a
  labels:
    app: demo-a
    service: svc-a
spec:
  ports:
  - port: 8080
    name: http
    targetPort: 8080
    protocol: TCP
  selector:
    app: demo-a
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-a-v1
  labels:
    app: demo-a
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-a
      version: v1
  template:
    metadata:
      labels:
        app: demo-a
        version: v1
    spec:
      containers:
      - name: demo-a
        image: yaogang/demo-a:v1.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
      restartPolicy: Always
---

```
网关转发路由配置：客户端/greet请求转发至svc-a服务。
demo-gateway.yaml文件
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: demo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
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
  name: demo
spec:
  hosts:
  - "*"
  gateways:
  - demo-gateway
  http:
  - match:
    - uri:
        exact: /greet
    route:
    - destination:
        host: svc-a
        port:
          number: 8080

```
**2. 配置转发规则 **

配置b服务的VirtualService和 DestinationRule规则（VirtualService是配置请求到该服务时如何转发至不同版本网络组(可以根据header头或是query参数，也可以按权重转发)，DestinationRule主要是配置后续在一个Deployment内部多个replicas副本之间的转发规则（是每个pod依次轮询还是随机pod））

destination-rule-svc-b-canary.yaml文件:
```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule #service到内部具体pod的转发规则
metadata:
  name: svc-b-destination-rule
spec:
  host: svc-b
  trafficPolicy:  #这里指的应该是一个Deployment内部多个replicas之间的转发规则
    loadBalancer:
      simple: RANDOM  
  subsets:  #svc-b内部分成v1和v2两个网络分组，
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

---
# 金丝雀发布-按流量比转发，确定上线后可先小比例用户走v2 观察，没问题再慢慢放量，直至流量100%走v2
# apiVersion: networking.istio.io/v1alpha3
# kind: VirtualService
# metadata:
  # name: svc-b
# spec:
  # hosts:
    # - svc-b
  # http:
  # - route:
    # - destination:
        # host: svc-b
        # subset: v1
      # weight: 5
    # - destination:
        # host: svc-b
        # subset: v2
      # weight: 95



---
# 金丝雀发布-特定用户走v2版本，可用于上线前指定部分测试人员测试
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: svc-b
spec:
  hosts:
    - svc-b  #VirtualService相当于创建了一个kubernetes service（这里指svc-b 服务）的代理，
  http:
  - match:
    - queryParams:
        name:
          exact: xx  #匹配请求参数name=xx的请求，可控制流量转发比例至v1或v2网络
    route:        
    - destination:
        host: svc-b
        subset: v1
      weight: 0
    - destination:
        host: svc-b
        subset: v2
      weight: 100
  - route:           #从上至下匹配，最后的默认匹配规则，这里没匹配上都走v1
    - destination:
        host: svc-b
        subset: v1
```
测试请求：`http://localhost/greet?name=xx`，始终返回：
```
response from [svc-b] [v2] HostName: demo-b-v2-678cd56448-kxj8n HostAddress: 10.1.0.88 name is: xx-v2
```
name参数等于其他值时，始终走v1版本。

注意点：

- 不管是根据header或是query参数进行转发，该标识值一定要在服务调用之间进行传递，