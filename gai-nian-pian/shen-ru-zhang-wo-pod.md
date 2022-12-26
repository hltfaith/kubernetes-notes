

主要围绕Pod 和容器的使用、应用配置管理、Pod的控制和调度管理、Pod的升级和回滚、Pod的扩缩容机制等。





# Pod定义详解









# Pod的基本用法

Pod可以由1个或多个容器组合而成，可以将两个容器应用为紧耦合的关系，并组合一个整体对外提供服务，将这两个容器打包为一个Pod。



补充拓扑图！！！



配置文件内容：

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: redis-php
	labels:
		name: redis-php
spec:
	containers:
	- name: frontend
		image: kubeguide/guestbook-php-frontend:localredis
		ports:
		- containerPort: 80
	- name: redis
		image: kubeguide/redis-master
		ports:
		- containerPort: 6379
```

属于同一个 Pod 的多个容器应用之间互相访问时仅需通过 localhost 就可以通信，使得这一组容器被 "绑定" 在一个环境中。



# 静态Pod

静态 Pod 是由 kubelet 进行管理的仅存在于特定 Node 上的 Pod。 它们不能通过 API Server 进行管理，无法与 ReplicationController、Deployment 或者 DaemonSet 进行关联，并且 kubelet 无法对它们进行健康检查。静态 Pod 总是由 kubelet 创建的，并且总在 kubelet 所在的 Node 上运行。

创建静态 Pod 有两种方式：配置文件和HTTP方式。

## 配置文件方式

需要设置 kubelet 的启动参数 `--pod-manifest-path` (或者在 kubelet 配置文件中设置 staticPodPath，这也是新版本推荐的设置方式， --pod-manifest-path 参数将被逐渐弃用)，指定 kubelet 需要监控的配置文件所在的目录，kubelet 会定期扫描该目录，并根据该目录下的 .yaml 或 .json 文件进行创建操作。

假设配置目录为 /etc/kubelet.d/， 配置启动参数为 `--pod-manifest-path=/etc/kubelet.d/`，然后重启 kubelet 服务。

在 /etc/kubelet.d 目录下放入 static-web.yaml文件

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: static-web
	labels:
		name: static-web
spec:
	containers:
	- name: static-web
		image: nginx
		ports:
		- name: web
			containerPort: 80
```

等一会，查看本机中已经启动的容

```shell
$ docker ps -a
```



> 注意：由于静态 Pod 无法通过 API Server 直接管理，所以在 Master 上尝试删除这个 Pod 时，会使其变成 Pending 状态，且不会被删除。
>
> 删除该Pod的操作只能是到其所在 Node 上 将其定义文件 static-web.yaml 从 /etc/kubelet.d 目录下删除。
>
> `rm /etc/kubelet.d/static-web.yaml`



## HTTP 方式

通过设置 kubelet  的启动参数 `--manifest-url`，kubelet 将会定期从该 URL 地址下载 Pod 的定义文件，并以 .yaml 和 .json 文件的格式进行解析，然后创建 Pod。其实现方式与配置文件方式是一致的。



# Pod容器共享Volume

同一个 Pod 中的多个容器能共享 Pod 级别的存储卷 Volume。Volume可以被定义为各种类型，多个容器各自进行挂载操作，将一个Volume挂载为容器内部需要的目录。



补充图!!!



举例，在Pod内包含两个容器：tomcat和busybox，在Pod级别设置 Volume 名 `app-logs` 用于 tomcat 容器向其中写日志文件，busybox容器从中读日志文件。

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: volume-pod
spec:
	containers:
	- name: tomcat
		image: tomcat
		ports:
		- containerPort: 8080
		volumeMounts:
		- name: app-logs
			mountPath: /usr/local/tomcat/logs
	- name: busybox
		image: busybox
		command: ["sh", "-c", "tail -f /logs/catalina*.log"]
		volumeMounts:
		- name: app-logs
			mountPath: /logs
	volumes:
	- name: app-logs
		emptyDir: {}
```

设置的Volume名称为 app-logs，类型为 emptyDir，挂载到tomcat容器内的 /usr/local/tomcat/logs 目录下，同时挂载到 busybox 容器内的 /logs 目录下。



# Pod的配置管理

应用部署的一个最佳实践是将应用所需的配置信息与程序分离，这样可以使应用程序被更好地复用，通过不同的配置也能实现更灵活的功能。将应用打包为容器镜像后，可以通过环境变量或者外挂文件的方式在创建容器时进行配置注入，但在大规模容器集群的环境中，对多个容器进行不同的配置将变得非常复杂。

Kubernetes 从 1.2 版本开始提供了一种统一的应用配置管理方案 ConfigMap。



ConfigMap供容器使用的典型用法如下

- 生成容器内的环境变量
- 设置容器启动命令的启动参数 (需设置为环境变量)
- 以Volume的形式挂载为容器内部的文件或目录

ConfigMap 以一个或多个 key:value 的形式保存在 Kubernetes 系统中供应用使用，即可以用于表示一个变量的值 (例如 apploglevel=info)，也可以用于表示一个完整配置文件的内容 （例如 server.xml=<?xml...>...）

也可以通过 YAML 文件或直接使用 `kubectl create configmap` 命令的方式创建ConfigMap



## 创建 ConfigMap 资源对象

**通过 YAML 文件方式创建**

cm-appvars.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: cm-appvars
data:
	apploglevel: info
	appdatadir: /var/data
```

通过 `kubectl create` 命令创建该 ConfigMap

```shell
$ kubectl create -f cm-appvars.yaml
```

查看刚创建好的 ConfigMap 对象

```shell
$ kubectl get configmap
$ kubectl describe configmap cm-appvars
$ kubectl get configmap cm-appvars -o yaml
```



也可以将配置文件定义为 ConfigMap资源，将key设置配置文件的别名，value 则是配置文件的全部文本内容

cm-appconfigfiles.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: cm-appconfigfiles
data:
	key-serverxml: |
		<?xml version='1.0' encoding='utf-8' ?>
		....
		...
		</xxx>
	key-loggingproperties: "handlers
		xxxxx
		xxxx
		end\n"
```

同上，按照上面 `kubectl create` 命令将该configmap 创建



**通过 kubectl 命令方式创建**

> 不使用 YAML 文件，直接通过 `kubectl create configmap` 也可以创建 ConfigMap，可以使用参数 --from-file 或 --from-literal 指定内容，并且可以在一行命令中指定多个参数。

1. 通过 `--from-file` 参数从文件中进行创建， 可以指定 key 的名称，也可以在一个命令行中创建包含多个 key 的 ConfigMap

```shell
$ kubectl create configmap NAME --from-file=/opt/test
```

>该目录下的每个配置文件名都被设置为key，文件的内容被设置为value。



2. 使用`--from-literal` 时会从文本中进行创建，直接将指定的 key=value创建为 ConfigMap 的内容

```shell
$ kubectl create configmap NAME --from-literal=key1=value1 --from-literal=key2=value2
```



## 在Pod中使用 ConfigMap

容器应用对 ConfigMap 的使用有以下两种方法

- 通过环境变量获取 ConfigMap 中的内容
- 通过 Volume 挂载的方式将 ConfigMap 中的内容挂载为容器内部的文件或目录



1. 通过环境变量方式使用 ConfigMap

比如刚刚创建的 cm-appvars 对象，下面将该configmap 环境变量引用

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: cm-test-pod
spec:
	containers:
	- name: cm-test
		image: busybox
		command: ["/bin/sh", "-c", "env | grep APP"]
		env:
		- name:	APPLOGLEVEL			# 定义环境变量的名称
			valueFrom:						# key apploglevel 对应的值
				configMapKeyRef:
					name: cm-appvars	# 环境变量的值取自 cm-appvars 资源
					key: apploglevel	# key 为 apploglevel
		- name: APPDATADIR
			valueFrom:
				configMapKeyRef:
					name: cm-appvars
					key: appdatadir
	restartPolicy: Never
```

> 注意：该Pod在运行完启动命令后将会退出，并且不会被系统自动重启 (restartPolicy=Never)



说明：Kubernetes 从 1.6 版本开始引入一个新的字段 envFrom，实现了在 Pod 环境中将 ConfigMap （也可用于 Secret 资源对象）中所有定义的 key=value自动生成为环境变量

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: cm-test-pod
spec:
	containers:
	- name: cm-test
		image: busybox
		command: ["/bin/sh", "-c", "env"]
		envFrom:
		- configMapRef
			name: cm-appvars		# 根据 cm-appvars 中的 key=value 自动生成环境变量
	restartPolicy: Never
```

> 需要说明的是：环境变量的名称受 [POSIX](https://baike.baidu.com/item/%E5%8F%AF%E7%A7%BB%E6%A4%8D%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E6%8E%A5%E5%8F%A3/12718298?fromtitle=POSIX&fromid=3792413&fr=aladdin) 命名规范 `[a-zA-Z_][a-zA-Z0-9_]*` 约束，不能以数字开头。如果包含非法字符，则系统将跳过该条环境变量，并记录一个 Event 来提示环境变量无法生成，但并不阻止 Pod 的启动。



2. 通过volumeMount使用ConfigMap

举例，将上文中 ConfigMap `cm-appconfigfiles` 的内容以文件的形式挂载到容器内部的 /configfiles 目录下

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: cm-test-app
spec:
	containers:
	- name: cm-test-app
		image: kubeguide/tomcat-app:v1
		ports:
		- containerPort: 8080
		volumeMounts:
		- name: serverxml						# 引用 Volume 的名称
			mountPath: /configfiles		# 挂载到容器内的目录
	volumes:
	- name: serverxml							# 定义Volume的名称
		configMap:
			name: cm-appconfigfiles		# 使用 ConfigMap 资源的 cm-appconfigfiles
			items:
			- key: key-serverxml			# key=key-serverxml
				path: server.xml				# value将 server.xml 文件名进行挂载
			- key: key-loggingproperties	# key=key-loggingproperties
				path: logging.properties		# value将 logging.properties 文件名进行挂载
```

创建Pod后，登录到容器中，查看 /configfiles目录下会存在 server.xml 和 logging.properties文件。



如果在引用 ConfigMap 时不指定 `items`，则使用 volumeMount 方式在容器内的目录下为每个 item 都生成一个文件名为 key 的文件。

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: cm-test-app
spec:
	containers:
	- name: cm-test-app
		image: kubeguide/tomcat-app:v1
		imagePullPolicy: Never
		ports:
		- containerPorts: 8080
		volumeMounts:
		- name: serverxml						# 引用 Volume 的名称
			mountPath: /configfiles		# 挂载到容器内的目录
	volumes:
	- name: serverxml							# 定义 Volume 的名称
		configMap:
			name: cm-appconfigfiles		# 使用 ConfigMap 资源的 cm-appconfigfiles
```

文件的名称来自在 ConfigMap cm-appconfigfiles 中定义的两个 key 的名称，文件的内容则为 value 的内容。



## 使用 ConfigMap 的限制条件

- ConfigMap必须在Pod之前创建，Pod才能引用它
- 如果 Pod 使用 envFrom 基于 ConfigMap 定义环境变量，则无效的环境变量名称（例如名称以数字开头）将被忽略，并在事件中被记录为 InvalidVariableNames。
- ConfigMap 受命名空间限制，只有处于相同命名空间中的Pod才可以引用它。
- ConfigMap 无法用于静态 Pod。



# Downward API

> 在容器内获取 Pod 信息

Kubernetes 在创建 Pod 成功之后，会为 Pod 和 容器设置一些额外的信息，例如 Pod 级别的 Pod 名称、 Pod IP、Node IP、Label、Annotation、容器级别的资源限制等。为了在容器内获取 Pod 级别的这些信息，Kubernetes 提供了 Downward API 机制来将 Pod 和 容器的某些元数据信息注入容器环境内，供容器应用方便地使用。

Downward API 可以通过以下两种方式将 Pod 和 容器的元数据信息注入容器内部

- 环境变量：将 Pod 或 Container 信息设置为容器内的环境变量
- Volume挂载：将 Pod 或 container信息以文件的形式挂载到容器内部



## 环境变量方式

**1). 将Pod信息设置为容器内的环境变量**

dapi-envars-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-fieldref
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "sh", "-c"]
      args:
      - while true; done
        echo -en '\n';
        printenv MY_NODE_NAME MY_POD_NAME MY_POD_NAMESPACE;
        printenv MY_POD_IP MY_POD_SERVICE_ACCOUNT;
        sleep 10;
        done;
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: MY_POD_SERVER_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
    restartPolicy: Never
```

> 注意：环境变量不直接设置 value，而是设置 valueFrom 对Pod 的元数据进行引用。



**3). 将  Container 信息设置为容器内的环境变量**

Container的资源请求和资源限制信息设置环境变量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-resourcefieldref
spec:
  containers:
  - name: test-container
    image: busybox
    imagePullPolicy: Never
    command: ["sh", "-c"]
    args:
    - while true; do
      echo -en '\n';
      printenv MY_CPU_REQUEST MY_CPU_LIMIT;
      printenv MY_MEM_REQUEST MY_MEM_LIMIT;
      sleep 3600;
      done;
    resources:
      requests:
        memory: "32Mi"
        cpu: "125m"
      limits:
        memory: "64Mi"
        cpu: "250m"
    env:
      - name: MY_CPU_REQUEST
        valueFrom:
          resourceFieldRef:
            containerName: test-container
            resource: requests.cpu
      - name: MY_CPU_LIMIT
        valueFrom:
          resourceFieldRef:
            containerName: test-container
            resource: limit.cpu
      - name: MY_MEM_REQUEST
        valueFrom:
          resourceFieldRef:
            containerName: test-container
            resource: requests.memory
      - name: MY_MEM_LIMIT
        valueFrom:
          resourceFieldRef:
            containerName: test-container
            resource: limits.memory
    restartPolicy: Never
```

- requests.cpu: 容器的CPU请求值
- limits.cpu: 容器的CPU限制值
- requests.memory: 容器的内存请求值
- limits.memory: 容器的内存限制值



## Volume挂载方式

**1). 将Pod信息挂载为容器内的文件**

通过 Downward API 将 Pod 的 Label、Annotation 信息通过 Volume 挂载为容器中的文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
  annotations:
    build: two
    builder: john-doe
spec:
  containers:
    - name: client-container
      image: busybox
      command: ["sh", "-c"]
      args:
      - while true; do
        if [[ -e /etc/podinfo/labels ]]; then
          echo -en '\n\n'; cat /etc/podinfo/labels; fi;
        if [[ -e /etc/podinfo/annotations ]]; then
          echo -en '\n\n'; cat /etc/podinfo/annotations; fi;
        sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```

在 Pod 的 volumes 字段中使用 Downward API 方法，通过 fieldRef 字段设置需要引用 Pod 的元数据信息，将其设置到 volume 的 item中

- metadata.labels: Pod 的 Label 列表
- metadata.annotations: Pod 的 Annotation 列表



## Downwared API 支持的Pod、Container信息

可以通过 fieldRef 设置的元数据如下

- metadata.name: Pod 名称
- metadata.namespace: Pod 所在的命名空间名称
- metadata.uid: Pod的UID，从Kubernetes 1.8.0-alpha.2 版本开始支持
- metadata.labels['<KEY>']: Pod 某个 Label 的值，通过 <KEY> 进行引用，从Kubernetes 1.9 版本开始支持。
- metadata.annotations['KEY']: Pod 某个 Annotation 的值，通过 <KEY> 进行引用，从 Kubernetes 1.9 版本开始支持。



通过 resourceFieldRef 设置的元数据如下

- Container 级别的 CPU Limit
- Container 级别的 CPU Request
- Container 级别的 Memory Limit
- Container 级别的 Memory Request
- Container 级别的临时存储空间 (ephemeral-storage) Limit，从 Kubernetes 1.8.0支持
- Container 级别的临时存储空间 (ephemeral-storage) Request，从 Kubernetes 1.8.0支持



以下Pod的元数据信息可以被设置为容器内的环境变量

- status.podIP: Pod 的IP地址
- spec.serviceAccountName: Pod使用的 ServiceAccount 名称
- spec.nodeName: Pod 所在 Node 的名称
- status. hostIP: Pod所在的Node的IP地址



# Pod 生命周期和重启策略



**Pod的状态**

- Pending: API Server 已经创建该 Pod，但在 Pod 内还有一个或多个容器的镜像没有创建，包括正在下载镜像的过程
- Running: Pod 内所有容器均已创建，且至少有一个容器处于运行状态、正在启动状态或正在重启状态
- Succeeded: Pod内所有容器均成功执行后退出，且不会再重启
- Failed: Pod内所有



**Pod的重启策略**

Pod的重启策略 (RestartPolicy) 应用于 Pod 内的所有容器，在Pod所处的Node上由 kubelet 进行判断和重启操作。

Pod重启策略包括 Always、OnFailure 和 Never

- Always: 当容器失效时，由 kubelet 自动重启该容器
- OnFailure: 当容器终止运行且退出码不为 0 时，由 kubelet 自动重启该容器。
- Never: 不论容器运行状态如何，kubelet 都不会重启该容器。

> kubelet 重启失效容器的时间间隔以 sync-frequency 乘以 2n 来计算，例如 1、2、4、8 倍等，最长延时 5min，并且在成功重启后的 10min 后重置该事件

pod的重启策略与控制方式息息相关，当前可用于管理 Pod 的控制器包括 ReplicationController、Job、DaemonSet，还可以通过 kubelet 管理(静态Pod)。 每种控制器对Pod的重启策略要求如下

- RC 和 DaemonSet：必须设置为 Always，需要保证该容器持续运行。
- Job: OnFailure 或 Never，确保容器执行完成后不再重启。
- kubelet: 在 Pod 失效时自动重启它，不论将 RestartPolicy  设置什么值，也不会对 Pod 进行健康检查。



# Pod 健康检查和服务可用性检查

对 Pod 健康状态可以通过三类探针来检查： **LivenessProbe**、**ReadinessProbe** 及 **StartupProbe**，其中最主要的探针为 LivenessProbe 与 ReadinessProbe，kubelet 会定期执行这两类探针来诊断容器的健康状况。

## LivenessProbe 探针

用于判断容器是否存活 (Running状态)，如果 LivenessProbe 探针探测到容器不健康，则 kubelet 将 "杀掉" 该容器，并根据容器的重启策略做相应的处理。 如果一个容器不包含 LivenessProbe 探针，那么 kubelet 认为该容器的 LivenessProbe 探针返回的值永远是 Success。

## ReadinessProbe 探针

用于判断容器服务是否可用 (Ready状态)，达到 Ready 状态的 Pod 才可以接收请求。 对于被 Service 管理的 Pod，Service与Pod Endpoint 的关联关系也将基于 Pod 是否 Ready 进行设置。如果在运行过程中 Ready状态变为 False，则系统自动将其从 Service 的后端 Endpoint 列表中隔离出去，后续再把恢复到 Ready 状态的 Pod 加回后端 Endpoint 列表。这样就能保证客户端在访问 Service 时不会被转发到服务不可用的 Pod 实例上。需要注意的是，ReadinessProbe 也是定期触发执行的，存在于 Pod 的生命周期中。

## StartupProbe 探针

某些应用会遇到启动比较慢的情况，例如应用程序启动时需要与远程服务器建立网络连接，或者遇到网络访问较慢等情况时，会造成容器启动缓慢，此时 ReadinessProbe 就不适用了，因为这属于 “有且仅有一次” 的超长延时，可以通过 StartupProbe探针解决该问题。



## 探针的三种实现方式

> 以上探针均可配置以下三种实现方式

**1). ExecAction**

在容器内部运行一个命令，如果该命令的返回码为0，则表明容器健康。

> 例如：通过运行 `cat /tmp/health` 命令来判断一个容器运行是否正常。在Pod运行后，将在创建 /tmp/health 文件 10s 后删除该文件，而 LivenessProbe 健康检查的开始探测时间 (initialDelaySeconds) 为15s，探测结果是 Fail，将导致 kubelet 杀掉该容器并重启它

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels: 
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: gcr.io/google_containers/busybox
    args:
    - /bin/sh
    - -c
    - echo ok > /tmp/health; sleep 10; rm -rf /tmp/health; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/health
      initialDelaySeconds: 15
      timeoutSeconds: 1
```



**2). TCPSocketAction**

通过容器的 IP 地址和端口号执行 TCP 检查，如果能够建立TCP连接，则表明容器健康。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-healthcheck
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      tcpSocket:
        port: 80
      initalDelaySeconds: 30
      timeoutSeconds: 1
```



**3). HTTPGetAction**

通过容器的IP地址、端口号及路径调用 HTTP Get 方法，如果响应的状态码大于等于 200 且小于 400，则认为容器健康。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-withh-healthcheck
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /_status/healthz
        port: 80
      initialDelaySeconds: 30
      timeoutSeconds: 1
```

注意：对于每种探测方式，都要设置 `initialDelaySeconds` 、`timeoutSeconds` 两个参数。

- `initialDelaySeconds`: 启动容器后进行首次健康检查的等待时间，单位为s。
- `timeoutSeconds`: 健康检查发送请求后等待响应的超时时间，单位为s。当超时发送时，kubelet 会认为容器已经无法提供服务，将会重启该容器。



> 扩展知识： Kubernetes 的 Pod 可用性探针机制可能无法满足某些复杂应用对容器内服务可用状态的判断，所以 Kubernetes 从 1.11版本开始，引入了 Pod Ready++ 特性对Readiness探测机制进行扩展。 在1.14 版本时达到 GA 稳定版本，称其为 Pod Readiness Gates 。
>
> Pod Readiness Gates 给予了 Pod 之外的组件控制某个Pod就绪的能力，通过 Pod Readiness Gates 机制，用户可以设置自定义的 Pod 可用性探测方式来告诉Kubernetes某个Pod是否可用，具体使用方式是用户提供一个外部的控制器(Controller) 来设置相应Pod的可用性状态。



# Pod 调度



## 全自动调度

Deployment、RC的主要功能之一就是自动部署一个容器应用的多份副本，以及持续监控副本的数量，在集群内始终维持用户指定的副本数量。

可以通过 `kubectl get rs` 和 `kubectl get pods` 查看已创建的 ReplicaSet (RS) 和 Pod 信息。

从调度策略上来说，创建的Pod由系统全自动完成调度。他们各自最终运行在哪个节点上，完全由 Master 的Scheduler 经过一系列算法计算得出，用户无法干预调度过程和结果。

> 除了使用系统自动调度算法完成一组 Pod 的部署， Kubernetes 也提供了多种丰富的调度策略，用户只需在 Pod 的定义中使用 NodeSelector、NodeAffinity、PodAffinity、Pod驱逐等更加细颗粒的调度策略设置，就能完成对 Pod 的精准调度。



## NodeSelector 定向调度

Kubernetes Master 上的 Scheduler 服务 (kube-scheduler进程) 负责实现 Pod 的调度，整个调度过程通过执行一系列复杂的算法，最终为每个 Pod 都计算出一个最佳的目标节点，这一过程是自动完成的，通常我们无法知道 Pod 最终会被调度到哪个节点上。

在实际情况下，也可能需要将 Pod 调度到指定的一些 Node 上，可以通过 Node 的标签 (Label) 和 Pod 的 nodeSelector 属性相匹配，来达到上述目的

```shell
$ kubectl label nodes <node-name> <label-key>=<label-value>
```

例如，为 k8s-node-1 节点打上一个 zone=north 标签

```shell
$ kubectl label nodes k8s-node-1 zone=north
```

然后，在 Pod 的定义中加上 nodeSelector 的设置

redis-master-controller.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-master
  labels:
    name: redis-master
spec:
  replicas: 1
  selector:
    name: redis-master
  template:
    metadata:
      labels:
        name: redis-master
    spec:
      containers:
      - name: master
        image: kubeguide/redis-master
        ports:
        - containerPort: 6379
      nodeSelector:
        zone: north
```

运行 kubectl create -f 命令创建 Pod， scheduler 就会将该 Pod 调度到拥有 "zone=north" 标签的Node上。

> 注意：如果给多个 Node 都定义了相同的标签 (例如 zone=north)，则 scheduler 会根据调度算法从这组 Node 中挑选一个可用的 Node进行 Pod 调度。
>
> 如果指定了 Pod 的 nodeSelector 条件，且在集群中不存在包含相应标签的 Node，则即使在集群中还有其他可供使用的Node，这个 Pod 也无法被成功调度。



Kubernetes 也会给 Node 预定义一些标签

- kubernetes.io/hostname
- beta.kubernetes.io/os
- beta.kubernetes.io/arch
- kubernetes.io/os
- kubernetes.io/arch



## NodeAffinity 亲和性调度

NodeAffinity 意为 Node 亲和性的调度策略，是用于替换 NodeSelector 的全新调度策略。

目前有两种节点亲和性表达

- RequiredDuringSchedulingIgnoredDuringExecution: 必须满足指定的规则才可以调度 Pod 到 Node 上(功能上 nodeSelector 很像，但是使用的是不同的语法)，相当于硬限制。
- PreferredDuringSchedulingIgnoredDuringExecution: 强调优先满足指定规则，调度器会尝试调度 Pod 到 Node上，但并不强求，相当于软限制。多个优先级规则还可以设置权重 (weight) 值，以定义执行的先后顺序。

IgnoredDuringExecution 的意思是：如果一个 Pod 所在的节点在 Pod 运行期间标签发生了变更，不在符合该 Pod 的节点亲和性需求，则系统将忽略 Node 上 Label 的变化，该 Pod 能继续在该节点运行。

举例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoreDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: beta.kubernetes.io/arch
            operator: In
            values:
            - amd64
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
            - key: disk-type
              operator: In
              values:
              - ssd
  containers:
  - name: with-node-affinity
    image: gcr.io/google_containers/pause:2.0
```

> 从配置中可以看到 In 操作符， NodeAffinity 语法支持的操作符包括 In、NotIn、Exists、DoesNotExist、Gt、Lt。

NodeAffinity 规则设置的 注意事项如下：

- 如果同时定义了 nodeSelector 和 nodeAffinity，那么必须两个条件都得到满足， Pod 才能最终运行在指定的 Node 上。
- 如果 nodeAffinity 指定了多个 nodeSelectorTerms，那么其中一个能匹配成功即可。
- 如果在 nodeSelectorTerms 中有多个 matchExpressions，则一个节点必须满足所有 matchExpressions 才能运行该 Pod。



## PodAffinity 亲和与互斥调度

简单的说，就是相关联的两种或多种 Pod 是否可以在同一个拓扑域中共存或者互斥。

> 拓扑域可以理解为：由一些Node节点组成，这些 Node 节点通常有相同的地址空间坐标，比如在同一个机架、机房或地区，一般用 region 表示机架、机房等的拓扑区域，用Zone表示地区这样跨度更大的拓扑区域。 在极端情况下，我们也可以认为一个 Node 就是一个拓扑区域。

Kubernetes 内置了一些常用的默认拓扑域

- kubernetes.io/hostname
- topology.kubernetes.io/region
- topology.kubernetes.io/zone

> 注意：以上拓扑域是由 kubernetes 自己维护，在Node节点初始化时，controller-manager 会为 Node 打上许多标签，比如 kubernetes.io/hostname 这个标签就会被设置为 Node 节点的 hostname。



Pod亲和与互斥的调度具体做法，是通过在 Pod 的定义上增加 topologyKey 属性，来声明对应的目录拓扑区域内几种相关联的 Pod 要 “在一起或不在一起”。 与节点亲和相同， Pod亲和与互斥的条件设置也是 requiredDuringSchedulingIgnoredDuringExecution 和 preferredDuringSchedulingIgnoredDuringExecution。Pod 的亲和性被定义于 PodSpec 的 affinity 字段的 podAffinity 子字段中，Pod 间的互斥性则被定义于同一层次的 podAntiAffinity子字段中。



## Taints、Tolerations 污点和容忍

简单地说，被标记为 Taint 的节点就是存在问题的节点，比如磁盘要满、资源不足、存在安全隐患要进行升级维护，希望新的 Pod 不会被调度过来，但被标记为 Taint 的节点并非故障节点，仍是有效的工作节点，所以仍需将某些 Pod 调度到这些节点上时，可以通过使用 Toleration 属性来实现。



## Pod Priority Preemption 优先级调度



>  :new: Tip：
>
> ​	在 Kubernetes 1.8 之前，当集群的可用资源不足时，在用户提交新的 Pod 创建请求后，该 Pod 会一直处于 Pending 状态，即使这个 Pod 是一个很重要 (很有身份) 的 Pod，也只能被动等待其他 Pod 被删除并释放资源，才能有机会被调度成功。
>
> ​    Kubernetes 1.8 版本引入了基于 Pod 优先级抢占 (Pod Priority Preemption)的调度策略，此时 Kubernetes 会尝试释放目标节点上低优先级的 Pod，以腾出空间（资源）安置高优先级的 Pod，这种调度方式被称为 "抢占式调度"。
>
>    在 Kubernetes 1.11 版本中，该特性升级为 Beta 版本，默认开启，在后续的 Kubernetes 1.14 版本中正式 Release。 



优先级抢占调度策略的核心行为分别是驱逐 (Eviction) 与抢占 (Preemption)，这两种行为的使用场景不同，效果相同。 Eviction 是 kubelet 进程的行为，即当一个 Node 资源不足 (under resource pressure)时，该节点上的 Kubelet 进程会执行驱逐动作，此时 kubelet 会综合考虑 Pod 的优先级、资源申请量与实际使用量等信息来计算哪些 Pod 需要被驱赶；当同样优先级的 Pod 需要被驱赶时，实际使用的资源量超过申请量最大倍数的高耗能 Pod 会被首先驱逐。

对于 Qos 等级为 "Best Effort" 的 Pod 来说，由于没有定义资源申请，所以他们实际使用的资源可能非常大。 Preemption 则是 Scheduler 执行的行为，当一个新的 Pod 因为资源无法满足而不能被调度时， Scheduler 可能 (有权决定) 选择驱逐部分低优先级的 Pod 实例来满足此 Pod 的调度目标，这就是 Preemption 机制。



Pod优先级调度示例

```yaml
apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

PriorityClass 不属于任何命名空间，优先级为1000000，数字越大，优先级越高，超过一亿的数字被系统保留，用于指派给系统组件。

在Pod中引用优先级类别

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels: 
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```



## 自定义调度器

一般情况下，每个新Pod都会由默认的调度器进行调度。但是如果在 Pod 中提供了自定义的调度器名称，那么默认的调度器会忽略该 Pod，转由指定的调度器完成 Pod 的调度。

为Pod指定一个名为 my-scheduler 的自定义调度器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  schedulerName: my-scheduler
  containers:
  - name: nginx
    image: nginx
```

如果自定义的调度器还未在系统中部署，则默认的调度器会忽略这个 Pod，这个 Pod 将会永远处于 Pending 状态。

可以用任意语言实现简单或复杂的自定义调度器。

下面使用 Bash 脚本进行实现，调度策略为随机选择一个 Node。这个调度器需要通过 `kubectl proxy` 来运行

```shell
#!/bin/bash
SERVER='localhost:8001'
while true;
do
	for PODNAME in $(kubectl --server $SERVER get pods -o json | jq '.items[] | select(.spec.schedulerName == "my-scheduler") | select(.spec.nodeName == null) | .metadata.name' | tr -d '"');
	do
		NODES=($(kubectl --server $SERVER get nodes -o json | jq '.items[].metadata.name' | tr -d '"'))
		NUMNODES=${#NODES[@]}
		CHOSEN=${NODES[$[$RANDOM % $NUMNODES]]}
		curl --header "Content-Type:application/json" --request POST --data '{"apiVersion":"v1", "kind":"Binding", "metadata":{"name": "'$PODNAME'"}, "target": {"apiVersion": "v1", "kind": "Node", "name": "'$CHOSEN'"}}' http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding/
		echo "Assigned $PODNAME to $CHOSEN"
	done
	sleep 1
done
```

> Tip: 该脚本需要实操论证。



# Init Container 初始化容器

需要场景中，都需要进行如下初始化操作

- 基于环境变量或配置模版生成配置文件。
- 从远程数据库获取本地所需配置，或者将自身注册到某个中央数据库中。
- 下载相关依赖包，或者对系统进行一些预配置操作。

**init container 与应用容器在本质上是一样的，但它们是仅运行一次就结束的任务，并且必须在成功运行完成后，系统才能继续执行下一个容器。**

根据 Pod 的重启策略 (RestartPolicy)，当 init container 运行失败而且设置了 `RestartPolicy=Never` 时， Pod将会启动失败；而设置 `RestartPolicy=Always`时，Pod将会被系统自动重启。



(补充图 3.10)



以 Nginx 应用为例，在启动 Nginx 之前，通过初始化容器 busybox 为 Nginx 创建一个 index.html 主页文件。

nginx-init-containers.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  annotations:
spec:
  initContainers:
  - name: install
    image: busybox
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://kubernetes.io
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```



> 注意：可以设置多个 init container，将按顺序逐个运行，并且只有前一个 init container 运行成功后才能运行后一个 init container。在所有 init container 都成功运行后，Kubernetes 才会初始化 Pod 的各种信息，并开始创建和运行应用容器。
>
> 在Pod重新启动时，init container 将会重新运行。



# Pod 的升级和回滚

如果 Pod 是通过 Deployment 创建的，则可以在运行时修改 Deployment 的 Pod 定义(spec.template) 或镜像名称，并应用到 Deployment 对象上，系统即可完成 Deployment 的 rollout 可被视为 Deployment 的自动更新或者自动部署动作。

如果在更新过程中发生了错误，可以通过回滚操作恢复 Pod 的版本。



## Deployment 升级

以 Deployment nginx 为例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

现在  Pod 镜像需要被更新为 Nginx:1.9.1，可以通过 `kubectl set image` 命令为 Deployment 设置新的镜像名称

```shell
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
```

另一种更新的方式是使用 `kubectl edit` 命令修改 Deployment 的配置，将 `spec.template.spec.containers[0].image` 从 Nginx:1.7.9 更改为 Nginx:1.9.1

```shell
$ kubectl edit deployment/nginx-deployment
```

使用  `kubectl rollout status` 命令查看 Deployment 的更新过程

```shell
$ kubectl rollout status deployment/nginx-deployment
```

在 Deployment 的定义中，可以通过 `spec.strategy` 指定 Pod 更新的策略，目前支持两种策略：Recreate(重建)和 RollingUpdate(滚动更新)，默认值为 RollingUpdate

- Recreate: 设置 `spec.strategy.type=Recreate`，表示 Deployment 在更新 Pod 时，会先 "杀掉" 所有正在运行的 Pod，然后创建新的Pod。
- RollingUpdate: 设置 spec.strategy.type=RollingUpdate，表示 Deployment 会以滚动更新的方式来逐个更新 Pod。同时，可以通过设置 spec.strategy.rollingUpdate 下的两个参数 (maxUnavailable 和 maxSurge) 来控制滚动更新的过程。



## Deployment 回滚

将 Deployment 回滚到之前的版本时，只有 Deployment 的 Pod 模版部分会被修改，在默认情况下，所有 Deployment 的发布历史记录都被保留在系统中 (可以配置历史记录数量)，便于随时进行回滚操作。

注意，在创建 Deployment 时使用 `--record` 参数，就可以在 `CHANGE-CAUSE`列看到每个版本使用的命令。

如果需要查看特定版本的详细信息，则可以加上 `--revision=<N>参数`：

```shell
$ kubectl rollout history deployment/nginx-deployment --revision=3
```

例如，回滚到上一个部署的版本

```shell
$ kubectl rollout undo deployment/nginx-deployment
```

也可以使用 `--to-revision` 参数指定回滚到的部署版本

```shell
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
```



## 暂停、恢复 Deployment

通过 `kubectl rollout pause` 命令暂停

```shell
$ kubectl rollout pause deployment/nginx-deployment
```

在暂停 Deployment 部署后，可以根据需要进行任意次数的配置更新

```shell
$ kubectl set resources deployment nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
```

最后恢复 Deployment

```shell
$ kubectl rollout resume deploy nginx-deployment
```

查看 Deployment 事件，查看更新

```shell
$ kubectl describe deployment/nginx-deployment
```



## DaemonSet 更新策略

主要分为两种： OnDelete 和 RollingUpdate。

**OnDelete:** DaemonSet 的默认升级策略。 当使用 OnDelete 作为升级策略时，在创建好新的 DaemonSet 配置之后，新的 Pod 并不会被自动创建，直到用户手动删除旧版的 Pod，才触发新建操作，即只有手工删除了 DaemonSet 创建的 Pod 副本，新的 Pod 副本才会被创建出来。

**RollingUpdate:**  当使用 RollingUpdate 作为升级策略对 DaemonSet 进行更新时，旧版本的 Pod 将自动 "杀掉"，然后自动创建新版本的 DaemonSet Pod。整个过程与普通 Deployment 的滚动升级一样是可控的。 **不过有两点不同于普通 Pod 的滚动升级：1.目前 Kubernetes 还不支持查看和管理 DaemonSet的更新历史记录；2.DaemonSet的回滚(Rollback) 并不能如同 Deployment 一样直接通过 kubectl rollback 命令来实现，必须通过再次提交旧版配置的方式实现。**

下面是 DaemonSet 采用 RollingUpdate 升级策略的 YAML定义：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: goldpinger
spec:
  updateStrategy:
    type: RollingUpdate
```



## StatefulSet 更新策略

StatefulSet 更新策略增加了 updateStrategy 字段给予用户更强的升级控制能力，并实现了 RollingUpdate、OnDelete 和 Partitioned 这几种策略，保证 StatefulSet 中各 Pod 有序、逐个更新，并且能够保留更新历史，也能回滚到某个历史版本。 如果用户未设置 updateStrategy 字段，则系统默认使用 RollingUpdate 策略。



# Pod 扩缩容

## 手动扩缩容机制

通过 `kubectl scale` 命令可以将 Pod 副本数量从初始的 3 个更新为 5 个

```shell
$ kubectl scale deployment nginx-deployment --replicas 5
```

> 将 --replicas 的值设置为比当前 Pod 副本数量更小的数字，系统将会 "杀掉" 一些运行中的 Pod。



## 自动扩缩容机制

kubernetes 从1.1版本，新增了 Horizontal Pod Autoscaler (HPA) 的控制器，用于实现基于 CPU使用率进行自动 Pod 扩缩容的功能。默认定义的探测周期 (15s)，周期性地检测目标 Pod 的资源性能指标，并与HPA资源对象中的扩缩容条件进行对比，在满足条件时对 Pod 副本数量进行调整。

从1.11版本，kubernetes 全面转向基于 Metrics Server 完成数据采集。



**HPA 工作原理**

kubernetes 中的某个 Metrics Server 持续采集所有  Pod 副本的指标数据。 HPA控制器通过 Metrics Server 的API获取这些数据，基于用户定义的扩缩容规则进行计算，得到目标 Pod 的副本数量。当目标 Pod 副本数量与当前副本数量不同时， HPA控制器就向 Pod 副本控制器 (Deployment、RC或 ReplicaSet) 发起 scale 操作，调整 Pod 的副本数量，完成扩缩容操作。













