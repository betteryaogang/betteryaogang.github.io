title: Win10 +Docker Desktop 安装k8s 部署过程记录
author: betteryaogang
date: 2022-03-11 16:04:29
tags:
---
### 安装Docker Desktop
1. win10 先开启Hyper-V功能
2. 下载并安装Docker Desktop

```javascript
$ docker version
Client:
 Cloud integration: v1.0.22
 Version:           20.10.12
 API version:       1.41
 Go version:        go1.16.12
 Git commit:        e91ed57
 Built:             Mon Dec 13 11:44:07 2021
 OS/Arch:           windows/amd64
 Context:           default
 Experimental:      true

Server: Docker Desktop 4.5.1 (74721)
 Engine:
  Version:          20.10.12
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.16.12
  Git commit:       459d0df
  Built:            Mon Dec 13 11:43:56 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.12
  GitCommit:        7b11cfaabd73bb80907dd23182b9347b4245eb5d
 runc:
  Version:          1.0.2
  GitCommit:        v1.0.2-0-g52b36a2
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```
记得切换一下国内镜像仓库，这样下载会快点，我使用的是阿里云的。

### 安装Kubernetes
打开`settings`查看支持得Kubernetes版本是`Kubernetes v1.22.5`。

**1. 准备安装镜像**

如果不提前准备好镜像的话，在启用Kubernetes的时候，页面会卡死，原因就是镜像默认会从Kubernetes提供的官方地址下载，该地址在国内是被墙的，所以我们需要将相关镜像先下载到本地。

- [github脚本仓库](https://github.com/AliyunContainerService/k8s-for-docker-desktop)
checkout v1.22.5的分支代码
- 以管理员身份打开powershell
- 进入`k8s-for-docker-desktop`目录，执行`Set-ExecutionPolicy RemoteSigned`设置脚本执行权限
- 执行下载镜像脚本`./load_images.ps1`

**2. 启用Kubernetes集群 **
- 到Docker Desktop，`settings`——> 左侧切换Kubernetes Tab，勾选`Enable Kubernetes`——>点击`Apply&Restart`按钮，等待。该过程会有点久，正常启动后，左下角Kubernetes图标会显绿色。

**3. 下载Kubectl客户端 **

[kubectl客户端下载地址](https://www.kubernetes.org.cn/installkubectl)，下载下来后执行一下，或放到System32目录下，配置环境变量略。
- 确保kubectl命令可用

**4. 安装ingress-nginx **

Kubernetes总共三种暴露服务的方式。

- LoadBlancer Service
- NodePort Service
- Ingress

这里使用Ingress从外部访问集群内部服务。

不同版本的Kubernetes，其配套的ingress-nginx版本是不一样的，我们可以看一下我们现在对应的版本是啥。
进入k8s-for-docker-desktop目录，查看`images.properties`

```javascript
k8s.gcr.io/pause:3.5=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5
k8s.gcr.io/kube-controller-manager:v1.22.5=registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.22.5
k8s.gcr.io/kube-scheduler:v1.22.5=registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.22.5
k8s.gcr.io/kube-proxy:v1.22.5=registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.22.5
k8s.gcr.io/kube-apiserver:v1.22.5=registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.22.5
k8s.gcr.io/etcd:3.5.0-0=registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0
k8s.gcr.io/coredns/coredns:v1.8.4=registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.8.4
k8s.gcr.io/ingress-nginx/controller:v1.1.1=registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.1.1
k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1=registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.1.1
```
可以看到ingress-nginx版本为`nginx-ingress-controller:v1.1.1`。

- 打开 [ingress-nginx的github仓库](https://github.com/kubernetes/ingress-nginx)，checkout tag为`controller-v1.1.1`的代码
- 依次进入`ingress-nginx-controller-v1.1.1`——>`deploy`——>`static`——>`provider`——>`cloud`目录
- 打开git bash窗口，执行命令：`kubectl apply -f deploy.yaml`
- 查看`kubectl get pods -n ingress-nginx`
- 访问localhost，熟悉的nginx 404页面，因为现在还没有配后端转发路由，所以找不到。
- 执行：`kubectl get svc -n ingress-nginx`，查看

```
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.104.57.10   localhost     80:32472/TCP,443:32398/TCP   15s
ingress-nginx-controller-admission   ClusterIP      10.102.38.64   <none>        443/TCP                      15s
```

**5. 准备Java服务docker镜像 **
- 新建一个SpringBoot服务，一个简单controller如下，返回hostname和ip。

```	java
    @GetMapping("/hello")
	public Object hello() throws UnknownHostException {
		InetAddress localHost = InetAddress.getLocalHost();
		return "HostName: " + localHost.getHostName() + "  HostAddress: " + localHost.getHostAddress();
	}

```
- 准备打包的Dockerfile

```	java
FROM buildo/java8-wkhtmltopdf
COPY ./demo.jar /usr/app/
WORKDIR /usr/app
ENTRYPOINT ["java", "-jar", "demo.jar"]
```
- 打包镜像，在Dockerfile目录下，执行：`docker build -t xxx/test-demo:v5.0 .`，别忘了最后有个点。首次打包需要拉基础镜像，可能会慢一点，完成后可执行：`docker images`查看一下镜像是否打包成功。

**6. 创建Deployment **

- 新建deployment.yaml文件：

```javascript
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-demo
  labels:
    app: test-demo
spec:
  replicas: 2  //启动2个pod
  template:
    metadata:
      name: test-demo
      labels:
        app: test-demo
    spec:
      containers:
        - name: test-demo
          image: xxx/test-demo:v5.0   //上一步打包的镜像
          imagePullPolicy: IfNotPresent
          ports:
            - name: http-port
              containerPort: 8080  //pod端口
      restartPolicy: Always
  selector:
    matchLabels:
      app: test-demo
```
- 命令行切换到文件目录，执行：`kubectl apply -f deployment.yaml`，等待一会儿，执行：`kubectl get pod -o wide` 查看：

```
$ kubectl get pod -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP          NODE             NOMINATED NODE   READINESS GATES
test-demo-5879578cc6-88zd9   1/1     Running   0          2m23s   10.1.0.39   docker-desktop   <none>           <none>
test-demo-5879578cc6-jhstw   1/1     Running   0          2m23s   10.1.0.40   docker-desktop   <none>           <none>
```
需要注意的是，该IP是集群内布IP，外面是访问不了的，可进入2个pod内部curl，发现可以相互调通。


**7. 创建Service **

- 新建service.yaml文件：

```javascript
apiVersion: v1
kind: Service
metadata:
  name: demo-service  //service名称
spec:
  selector:
    app: test-demo  //代理有label app: test-demo的pod，
  ports:
  - port: 8080  //service暴露端口
    targetPort: 8080  //pod端口
    protocol: TCP

```
- 命令行切换到文件目录，执行：`kubectl apply -f service.yaml`，等待一会儿，执行：`kubectl get svc -o wide` 查看：

```
$ kubectl get svc -o wide
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE    SELECTOR
demo-service   ClusterIP   10.108.174.0   <none>        8080/TCP   3h7m   app=test-demo
kubernetes     ClusterIP   10.96.0.1      <none>        443/TCP    47h    <none>
```
需要注意的是，CLUSTER-IP也是集群内布IP，外面是访问不了的，可进入2个pod内部curl，发现可以调通该service的CLUSTER-IP。

**7. 创建Ingress **

- 新建ingress.yaml文件：

```javascript
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  ingressClassName: nginx  //重要，不加不会更新nginx配置文件
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix  //路由匹配方式
            backend:
              service:
                name: demo-service  //转发到的service名称
                port:
                  number: 8080   //转发到的service端口

```
- 命令行切换到文件目录，执行：`kubectl apply -f ingress.yaml`，等待完成。

- 执行：` kubectl describe ingress test-ingress`，查看：

```
$  kubectl describe ingress test-ingress
Name:             test-ingress
Namespace:        default
Address:          localhost
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /   demo-service:8080 (10.1.0.41:8080,10.1.0.42:8080)
Annotations:  <none>
Events:       <none>
```

 **进入ingress-controller容器，确认ingress配置生效**

- 首先执行：`kubectl -n ingress-nginx get pods`，查找容器名：

```
$ kubectl -n ingress-nginx get pods
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create--1-bjl5c     0/1     Completed   0          23h
ingress-nginx-admission-patch--1-nxqg8      0/1     Completed   0          23h
ingress-nginx-controller-54d8b558d4-blfp7   1/1     Running     0          23h
```
- 执行`$ kubectl exec -n ingress-nginx -it ingress-nginx-controller-54d8b558d4-blfp7 -- //bin/bash`进入容器

- 执行：`cat nginx.conf` 查看nginx配置文件，完成文件太长，截取一段，发现ingress规则已生效：
```
 location / {

                        set $namespace      "default";
                        set $ingress_name   "test-ingress";
                        set $service_name   "demo-service";
                        set $service_port   "8080";
                        set $location_path  "/";
                        set $global_rate_limit_exceeding n;

                        rewrite_by_lua_block {
                                lua_ingress.rewrite({
                                        force_ssl_redirect = false,
                                        ssl_redirect = true,
                                        force_no_ssl_redirect = false,
                                        preserve_trailing_slash = false,
                                        use_port_in_redirects = false,
                                        global_throttle = { namespace = "", limit = 0, window_size = 0, key = { }, ignored_cidrs = { } },
                                })
                                balancer.rewrite()
                                plugins.run()
                        }
```



最后，访问`http://localhost/hello`，页面显示
```
HostName: test-demo-569bc94589-f87tn HostAddress: 10.1.0.41
```
或
```
HostName: test-demo-569bc94589-b66m6 HostAddress: 10.1.0.42
```
表示请求最终来回切换转发到了不同的pod上。

ok，至此win10上安装Docker Desktop、并搭建Kubernetes集群，到部署一个简单的Deployment、Service、Ingress的实践就搞完了，中间踩了不少坑，记录一下。


