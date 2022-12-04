

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
















































