Service 是 kubernetes 实现微服务架构的核心概念，通过创建 Service，可以为一组具有相同功能的容器应用提供一个统一的入口地址，并且将请求负载分发到后端的各个容器应用上。



# Service 定义详解

service 用于为一组提供服务的 Pod 抽象一个稳定的网络访问地址，是 Kubernetes 实现微服务的核心概念。 通过 Service 的定义设置的访问地址是 DNS 域名格式的服务名称，对于客户端应用来说，网络访问方式并没有改变 (DNS 域名的作用等价于主机名、互联网域名或IP地址)。Service还提供了负载均衡器功能，将客户端请求负载分发到后端提供具体服务的各个 Pod 上。



Service 的Yaml 格式的定义文件完整内容如下

```yaml
apiVersion: v1 // Required
kind: Service  // Required
metadata: 
  name: string // Required
  namespace: string // Required
  labels:
    - name: string
  annotations:
    - name: string
spec:
  selector: [] // Required
  type: string // Required
  clusterIP: string
  sessionAffinity: string
  ports:
  - name: string
    protocol: string
    port: int
    targetPort: int
    nodePort: int
  status:
    loadBalancer:
      ingress:
        ip: string
        hostname: string
```





# Service 的概念和原理



## Service 的概念

举例：创建一个Web服务的 Pod 集合，由两个 Tomcat 容器副本组成，每个容器提供的服务端口号都为 8080

webapp-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
```

创建该 Deployment

```shell
$ kubectl create -f webapp-deployment.yaml
```

现状，可以通过命令 `kubectl get pods -l app=webapp -o wide` 查看到这两个 Pod 的 IP 地址和端口号，通过ip:端口访问WEB服务。

> 通过 Service 的定义，可以对客户端应用屏蔽后端  Pod 实例数量及 Pod IP 地址的变化，通过负载均衡策略实现请求到后端 Pod 实例的转发，为客户端应用提供一个稳定的服务访问入口地址。

Kubernetes 提供了一种快速创建 service 的方法，通过 kubectl expose 命令

```shell
$ kubectl expose deployment webapp
```

`kubectl get svc` 命令查看新创建的 Service， 可以看到系统为它分配了一个虚拟 IP 地址 (ClusterIP 地址)，Service 的端口号则从 Pod 中的 containerPort 复制而来。

最后，通过 Service 的 IP 地址和 Service 端口号访问该Service了。 (通过访问ClusterIP，被自动负载到了后端两个POD之一)



除了使用 `kubectl expose` 命令创建 Service，更便于管理的方式是通过YAML文件来创建 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  selector:
    app: webapp
```

例子中的 ports 定义部分指定了 Service 本身的端口号为 8080，targetPort 则用来指定后端 Pod 的容器端口号，selector 定义部分设置的是后端 Pod 所拥有的 label：`app=webapp`

> 注意：在提供服务的 Pod 副本集运行过程中，如果 Pod 列表发生了变化，则 Kubernetes 的 Service 控制器会持续监控后端 Pod 列表的变化，实时更新 Service 对应的后端 Pod 列表。



一个 Service 对应的后端由 Pod 的 IP 和容器端口号组成，即一个完整的 "IP:Port"访问地址，在 kubernetes 系统中叫作 Endpoint。通过查看 Service 的详细信息，可以看到后端 Endpoint 列表

```shell
$ kubectl get svc webapp
```

Kubernetes 自动创建了与 Service 关联的 Endpoint 资源对象，可以通过查询 Endpoint 对象进行查看

```shell
$ kubectl get endpoints
```

Service 不仅具有标准网络协议的IP地址，还以 DNS 域名的形式存在。 Service 的域名表示方法为 <servicename>.<namespace>.svc.<clusterdomain>，servicename 为服务的名称，namespace 为其所在 namespace 的名称， clusterdomain 为 Kubernetes 集群设置的域名后缀。



## Service 的负载均衡机制

从服务 IP 到后端 Pod 的负载均衡机制，则是由每个 Node 上的 kube-proxy 负责实现的。

主要对 kube-proxy 的代理模式、会话保持机制和基于拓扑感知的服务路由机制 (EndpointSlices) 进行说明。



### kube-proxy 的代理模式

目前 kube-proxy 提供以下代理模式 (通过启动参数 `--proxy-mode` 设置)

- userspace 模式：用户空间模式，由 kube-proxy完成代理的实现，效率最低，不再推荐使用。
- iptables模式：kube-proxy通过设置 Linux Kernel 的 iptables 规则，实现从 Service 到后端 Endpoint 列表的负载分发规则，效率很高。但是，如果某个后端 Endpoint 在转发不可用，此次客户端请求就会得到失败的响应，相对于 userspace 模式来说更不可靠。 此时应该通过为 Pod 设置 readinessprobe (服务可用性健康检查)来保证只有达到 ready 状态的 Endpoint 才会被设置为 Service 的后端 Endpoint。
- ipvs模式：在 Kubernetes 1.11 版本中达到 Stable 阶段，kube-proxy 通过设置  Linux Kernel 的 netlink 接口设置 IPVS规则，转发效率和支持的吞吐率都是最高的。 ipvs模式要求 Linux Kernel 启用 IPVS 模块，如果操作系统未启用 IPVS 内核模块，kube-proxy 则会自动切换至 iptables  模式。 同时，ipvs 模式支持更多的负载均衡策略。
- kernelspace 模式： Windows Server 上的代理模式。



### 会话保持机制

Service 支持通过设置 sessionAffinity 实现基于客户端 IP 的会话保持机制，即首次将某个客户端来源 IP 发起的请求转发到后端的某个 Pod 上，之后从相同的客户端IP发起的请求都将被转发到相同的后端Pod上，配置参数为 service.spec.sessionAffinity

```yaml
```











































