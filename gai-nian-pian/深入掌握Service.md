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
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  selector:
    app: webapp
```

可以设置会话保持的最长时间，在此时间之后重置客户端来源IP的保持规则，配置参数为 `service.spec.sessionAffinityConfig.clientIP.timeoutSeconds`



## Service 的多端口设置

一个容器应用可以提供多个端口的服务，在 Service 的定义中也可以相应地设置多个端口号。

可以设置两个端口号分别提供不同的服务，也可以使用同一个端口号使用的协议不同。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 169.169.0.100
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```



## 将外部服务定义为 Service

普通的 Service 通过 Label Selector 对后端 Endpoint 列表进行了一次抽象，如果后端的 Endpoint 不是由 Pod 副本集提供的，则 Service 还可以抽象定义任意其他服务，将一个 Kubernetes 集群外部的已知服务定义为 Kubernetes 内的一个 Service，供集群内的其他应用访问。

在创建 Service 资源对象时不设置 Label Selector，同时再定义一个与Service关联的 Endpoint资源对象，在 Endpoint 中设置外部服务的IP地址和端口号。

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
  - protocol: tcp
    port: 80
    targetPort: 80
    
---
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
- addresses:
  - IP: 1.2.3.4
  ports:
  - port: 80
```



## 将 Service 暴露到集群外部

Kubernetes 为 Service 创建的 ClusterIP 地址是对后端 Pod 列表的一层抽象，对于集群外部来说并没有意义，但有许多 Service 是需要对集群外部提供服务的，Kubernetes 提供了多种机制将 Service 暴露出去，供集群外部的客户端访问。

Service 的类型如下

- ClusterIP:  Kubernetes 默认会自动设置 Service 的虚拟IP地址，仅可被集群内部的客户端应用访问。当然，用户也可手工指定一个 ClusterIP 地址，不过需要确保该 IP 在 Kubernetes 集群设置的 ClusterIP 地址范围内（通过 kube-apiserver 服务的启动参数 --service-cluster-ip-range 设置）,并且没有被其他Service 使用。
- NodePort: 将Service的端口号映射到每个Node的一个端口号上，这样集群中的任意Node都可以作为 Service 的访问入口地址，即NodeIP:NodePort
- LoadBalancer: 将 Service 映射到一个已存在的负载均衡器的IP地址上，通常在公有云环境中使用。
- ExternalName: 将 Service 映射到一个外部域名地址，通过 externalName 字段进行设置。



### NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 8081
  selector:
    app: webapp
```

创建后该 Service，可以通过 Node的IP和 NodePort 8081端口号访问服务了。

在默认情况下，Node的 kube-proxy 会在全部网卡 (0.0.0.0) 上绑定 NodePort 端口号。

从 Kubernetes 1.10 版本开始，kube-proxy 可以通过设置特定的 IP 地址将 NodePort 绑定到特定的网卡上，而无须绑定在全部网卡上，其设置方式为配置启动参数 `--nodeport-addresses`，指定需要绑定的网卡IP地址，多个地址之间使用逗号分隔。

```
--nodeport-addresses=10.0.0.0/8,192.168.18.0/24
```



### LoadBalancer 类型

可以将 Service 映射到公有云提供的某个负载均衡器的IP地址上，客户端通过负载均衡器的IP和 Service 的端口号就可以访问到具体的服务，无须再通过 kube-proxy 提供的负载均衡机制进行流量转发。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
  clusterIP: 10.0.171.239
```

在服务创建后，云服务商会在 Service 的定义中补充 `LoadBalancer` 的IP地址

```yaml
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```



### ExternalName 类型

ExternalName 类型的服务用于将集群外的服务定义为 Kubernetes 的集群的 Service，并且通过 externalName 字段指定外部服务的地址，可以使用域名或 IP 格式。集群内的客户端应用通过访问这个 Service 就能访问外部服务了。 这种类型的 Service 没有后端Pod，所以无须设置 Label Selector。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

客户端访问服务地址 my-service.prod.svc.cluster.local 时，系统将自动指向外部域名 my.database.example.com。



## Service 支持的网络协议

Service 支持的网络协议

- TCP:  Service的默认网络协议，可用于所有类型的 Service。
- UDP:  可用于大多数类型的Service，LoadBalancer 类型取决于云服务商对 UDP的支持。
- HTTP:  取决于云服务商是否支持 HTTP 和实现机制。
- PROXY: 取决于云服务商是否支持 HTTP 和实现机制。
- SCTP:  从 Kubernetes 1.12 版本引入，到 1.19 版本时达到 Beta 阶段，默认启用，如需关闭该特性，则需要设置 kube-apiserver 的启动参数 `--feature-gates=SCTPSupport=false` 进行关闭。

Kubernetes 从 1.17 版本，可以为 Service 和 Endpoint 资源对象设置一个新的字段 AppProtocol，用于标识后端服务在某个端口号上提供的应用层协议类型，例如HTTP、HTTPS、SSL、DNS等。

要使用 AppProtocol，需要设置 kube-apiserver 的启动参数 `--feature-gates=ServiceAppProtocol=true` 进行开启，然后在 Service 或 Endpoint 的定义中设置 AppProtocol 字段指定应用层协议的类型。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  ports:
  - port: 8080
    targetPort: 8080
    AppProtocol: HTTP
  selector:
    app: webapp
```



## Kubernetes 的服务发现机制

服务发现机制指客户端应用在一个 Kubernetes 集群中如何获知后端服务的访问地址。 Kubernetes 提供了两种机制供客户端应用以固定的方式获取后端服务的访问地址：环境变量方式和DNS方式。

### 环境变量方式

比如在一个新创建的Pod(客户端应用)中，设置环境变量

```shell
WEBAPP_SERVICE_HOST=169.169.81.174
WEBAPP_SERVICE_PORT=8080
```

然后，客户端应用就能够根据 Service 相关环境变量的命名规则，从环境变量中获取需要访问的目标服务的地址。



### DNS方式

Service 在 Kubernetes 系统中遵循 DNS 命名规范，Service 的DNS域名表示方法为 `<servicename>.<namespace>.svc.<clusterdomain>`，其中 servicename 为服务的名称， namespace 为其所在 namespace 的名称， clusterdomain 为 Kubernetes 集群设置的域名后缀 (例如 cluster.local)，服务名称的命名规范遵循 RFC 1123 规范的要求。

当 Service 以 DNS 域名形式进行访问时，就需要在 Kubernetes 集群中存在一个 DNS 服务器来完成域名到 ClusterIP 地址的解析工作，经过多年的发展，目前由 CoreDNS 作为 Kubernetes 集群的默认DNS服务器提供域名解析服务。

> 如果，Service 定义中的端口号设置了名称 (name)，则该端口号也会拥有一个DNS域名，在DNS服务器中以SRV记录的格式保存：`_<portname>._<protcol>.<servicename>.<namespace>.svc.<clusterdomain>`，其值为端口号的数值。

以 webapp 服务为例，将其端口号命名为 "http"

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
    name: http
  selector:
    app: webapp
```

解析名为 "http" 端口的 DNS SRV记录

```shell
$ nslookup -q=srv _http._tcp.webapp.default.svc.cluster.local
```

可以查询到其端口号的值为 8080













































