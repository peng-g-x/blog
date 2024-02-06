## Pod概念

### Pod结构

Pod是k8s集群进行管理的最小单元，程序要运行必须部署在容器中，而容器必须存在于pod中，一个pod中可以存在一个或者多个容器，这些容器可以分为两类：

1. 用户程序所在的容器，数量可多可少
2. Pause容器，这是每个pod都会有的一个根容器，作用如下：

- 可以以他为依据，评估整个pod的健康状态
- 可以在根容器上设置IP地址，其他容器都根据此IP（pod IP），以实现pod内部的网络通信

![](https://cdn.nlark.com/yuque/0/2023/png/12836966/1676860553921-3cc63fc1-57df-4aea-8f7e-4494512549be.png#averageHue=%2396fafa&clientId=ue5b557d6-7ecb-4&from=paste&height=355&id=u0803749c&originHeight=464&originWidth=633&originalType=url&ratio=1.5625&rotation=0&showTitle=false&status=done&style=none&taskId=ud809344b-9f4b-4b4b-a585-6064a9e4907&title=&width=484)
<a name="Vs9Nv"></a>

### Pod配置文件

```yaml
apiVersion: v1     #必选，版本号，例如v1
kind: Pod       　 #必选，资源类型，例如 Pod
metadata:       　 #必选，元数据
  name: string     #必选，Pod名称
  namespace: string  #Pod所属的命名空间,默认为"default"
  labels:       　　  #自定义标签列表
    - name: string      　          
spec:  #必选，Pod中容器的详细定义
  containers:  #必选，Pod中容器列表
  - name: string   #必选，容器名称
    image: string  #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 
    command: [string]   #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      #容器的启动命令参数列表
    workingDir: string  #容器的工作目录
    volumeMounts:       #挂载到容器内部的存储卷配置
    - name: string      #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean #是否为只读模式
    ports: #需要暴露的端口库号列表
    - name: string        #端口的名称
      containerPort: int  #容器需要监听的端口号
      hostPort: int       #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string    #端口协议，支持TCP和UDP，默认TCP
    env:   #容器运行前需设置的环境变量列表
    - name: string  #环境变量名称
      value: string #环境变量的值
    resources: #资源限制和请求的设置
      limits:  #资源限制的设置
        cpu: string     #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests: #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string #内存请求,容器启动的初始可用数量
    lifecycle: #生命周期钩子
        postStart: #容器启动后立即执行此钩子,如果执行失败,会根据重启策略进行重启
        preStop: #容器终止前执行此钩子,无论结果如何,容器都会终止
    livenessProbe:  #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器
      exec:       　 #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
  restartPolicy: [Always | Never | OnFailure]  #Pod的重启策略
  nodeName: <string> #设置NodeName表示将该Pod调度到指定到名称的node节点上
  nodeSelector: obeject #设置NodeSelector表示将该Pod调度到包含这个label的node上
  imagePullSecrets: #Pull镜像时使用的secret名称，以key：secretkey格式指定
  - name: string
  hostNetwork: false   #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
  volumes:   #在该pod上定义共享存储卷列表
  - name: string    #共享存储卷名称 （volumes类型有很多种）
    emptyDir: {}       #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
    hostPath: string   #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
      path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
    secret:       　　　#类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
      scretname: string  
      items:     
      - key: string
        path: string
    configMap:         #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
      name: string
      items:
      - key: string
        path: string
```

### 属性查看

1. 查看某种资源可以配置的一级属性：kubectl explain 资源类型

- apiVersion：版本，由k8s内部定义，版本号必须可以用kubectl api-versions查询到
- kind：类型，由k8s内部定义
- metadata：元数据，主要是资源标识和说明，常用的有name、namespace、labels等
- spec：描述，里面是对各种资源配置的详细描述
- status：状态信息，里面的内容不需要定义，由k8s自动生成

2. 查看属性的子属性：kubectl explain 资源类型.属性

```shell
[root@k8s-master01 ~]# kubectl explain pod
KIND:     Pod
VERSION:  v1
FIELDS:
   apiVersion   <string>
   kind <string>
   metadata     <Object>
   spec <Object>
   status       <Object>

[root@k8s-master01 ~]# kubectl explain pod.metadata
KIND:     Pod
VERSION:  v1
RESOURCE: metadata <Object>
FIELDS:
   annotations  <map[string]string>
   clusterName  <string>
   creationTimestamp    <string>
   deletionGracePeriodSeconds   <integer>
   deletionTimestamp    <string>
   finalizers   <[]string>
   generateName <string>
   generation   <integer>
   labels       <map[string]string>
   managedFields        <[]Object>
   name <string>
   namespace    <string>
   ownerReferences      <[]Object>
   resourceVersion      <string>
   selfLink     <string>
   uid  <string>
```

## Pod配置

### 镜像拉取

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-imagepullpolicy
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    imagePullPolicy: Never # 用于设置镜像拉取策略
  - name: busybox
    image: busybox:1.30
```

imagePullPolicy：用于设置镜像拉取策略，k8s支持3种拉取策略

1. Always：总是从远程仓库拉取镜像
2. ifNotPresent：本地有则使用本地镜像，本地没有则从远程仓库拉取镜像
3. Never：只使用本地镜像，从不去远程仓库拉取，本地没有就报错

**注意**

1. 如果镜像tag为具体版本号，默认策略是ifNotPresent
2. 如果镜像tag为lastest（最终版本），默认策略是always

### 启动命令

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-command
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","touch /tmp/hello.txt;while true;do /bin/echo $(date +%T) >> /tmp/hello.txt; sleep 3; done;"]
```

command：用于在pod中的容器初始化完毕之后运行一个命令

1. 如果command和args均没有写，那么用Dockerfile的配置。
2. 如果command写了，但args没有写，那么Dockerfile默认的配置会被忽略，执行输入的command
3. 如果command没写，但args写了，那么Dockerfile中配置的ENTRYPOINT的命令会被执行，使用当前args的参数
4. 如果command和args都写了，那么Dockerfile的配置被忽略，执行command并追加上args参数

### 环境变量

env：用于在pod中的容器设置环境变量（不推荐）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do /bin/echo $(date +%T);sleep 60; done;"]
    env: # 设置环境变量列表
    - name: "username"
      value: "admin"
    - name: "password"
      value: "123456"
```

### 端口设置

1. name：端口名称，如果指定，必须保证name在pod中是唯一的
2. containerPort：容器要监听的端口（0<x<65535）
3. hostPort：容器要在主机上公开的端口，如果设置，主机上只能运行容器的一个副本
4. hostIP：要将外部端口绑定到的主机IP
5. protocol：端口协议，必须是UDP、TCP或SCTP，默认为TCP

访问容器中的程序需要使用的是PodIP:containerPort

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ports
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: # 设置容器暴露的端口列表
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

### 资源限额

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    resources: # 资源配额
      limits:  # 限制资源（上限）
        cpu: 2 # CPU限制，单位是core数
        memory: 10Gi # 内存限制
      requests: # 请求资源（下限）
        cpu: 1  # CPU限制，单位是core数
        memory: 10Mi  # 内存限制
```

如果不对某个容器的资源做限制，那么它就可能吃掉大量资源，导致其他容器无法运行。k8s提供了对内存和CPU的资源进行配额的机制，这种机制主要通过resources选项实现

1. limits：用于限制运行时容器的最大占用资源，当容器占用资源超过limits时会被终止，并进行重启
2. requests：用于设置容器需要的最小资源，如果环境资源不够，容器将无法启动

- cpu：core数，可以为整数或小数
- memory：内存大小，可以使用Gi、Mi、G、M等形式

## Pod生命周期

一般将pod对象从创建至终的这段时间范围称为pod的生命周期，主要有以下过程：

1. pod创建
2. 初始化容器（init container）
3. 运行主容器（main container）

- 容器启动后钩子（post start）、容器终止前钩子（pre stop）
- 容器的存活性探测（liveness probe）、就绪性探测（readiness probe）

4. pod终止

![](https://cdn.nlark.com/yuque/0/2023/png/12836966/1678077718921-234467b9-2ea7-401d-8052-acbba3009b64.png#averageHue=%23dadaf8&clientId=u57d41497-1054-4&from=paste&height=343&id=u962c91bc&originHeight=548&originWidth=1053&originalType=url&ratio=1.5625&rotation=0&showTitle=false&status=done&style=none&taskId=ue96a5ef7-e5b7-42cb-ae30-a0ddfe9c68a&title=&width=659.719970703125)

在整个生命周期中，pod会出现5种状态（相位)：

1. 挂起（Pending）：apiserver已经创建了pod资源对象，但它尚未被调度完成或者仍处于下载镜像的过程中
2. 运行中（Running）：pod已经被调度至某节点，并且所有容器都已经被kubelet创建完成
3. 成功（Succeeded）：pod中的所有容器都已经成功终止并且不会被重启
4. 失败（Failed）：所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非0值的退出状态
5. 未知（Unknown）：apiserver无法正常获取到pod对象的状态信息，通常由网络通信失败所导致

### 创建和终止过程

**创建过程**

1. 用户通过kubectl或其他api客户端提交需要创建的pod信息给apiServer
2. apiServer开始生成pod对象的信息，并将信息存入etcd，然后返回确认信息至客户端
3. apiServer开始反映etcd中的pod对象变化，其他组件使用watch机制来跟踪检查apiServer上的变动
4. scheduler发现有新的pod对象要创建，开始为pod分配主机并将结果信息更新至apiServer
5. node节点上的kubelet发现有pod调度过来，尝试调用docker启动容器，并将结果回送至apiServer
6. apiServer将接收到的pod状态信息存入etcd中

**终止过程**

1. 用户向apiServer发送删除pod对象的命令
2. apiServer中的pod对象信息会随着时间的推移而更新，在宽限期内（默认30s），pod被视为dead
3. 将pod标记为terminating状态
4. kubelet在监控到pod对象转为terminating状态的同时启动pod关闭过程
5. 端点控制器监控到pod对象的关闭行为时将其从所有匹配到此端点的service资源的端点列表中移除
6. 如果当前pod对象定义了preStop钩子处理器，则在其标记为terminating后即会以同步的方式启动执行
7. pod对象中的容器进程收到停止信号
8. 宽限期结束后，若pod中还存在运行的进程，那么pod对象会收到立即终止的信号
9. kubelet请求apiServer将此pod资源的宽限期设置为0，从而完成删除操作，此时pod对于用户已不可见

![image-20200406184656917](https://gitee.com/yooome/golang/raw/main/22-k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B-%E8%B0%83%E6%95%B4%E7%89%88/Kubenetes.assets/image-20200406184656917-1626782168787.png)

### 初始化容器

初始化容器是在pod的主容器启动之前要运行的容器，主要是做一些主容器的前置工作，有两大特征：

1. 初始化容器必须运行完成直至结束，若某初始化容器运行失败，那么k8s需要重启它直到成功完成
2. 初始化容器必须按照定义的顺序执行，当且仅当前一个成功之后，后面的一个才能运行

应用场景：

1. 提供主容器镜像不具备的工具程序或自定义代码
2. 初始化容器要先于应用容器串行启动并运行完成，因为可用于延后应用容器的启动直至依赖的条件得到满足

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-initcontainer
  namespace: dev
spec:
  containers:
  - name: main-container
    image: nginx:1.17.1
    ports: 
    - name: nginx-port
      containerPort: 80
  initContainers:
  - name: test-mysql
    image: busybox:1.30
    command: ['sh', '-c', 'until ping 192.168.90.14 -c 1 ; do echo waiting for mysql...; sleep 2; done;']
  - name: test-redis
    image: busybox:1.30
    command: ['sh', '-c', 'until ping 192.168.90.15 -c 1 ; do echo waiting for reids...; sleep 2; done;']
```

### 钩子函数

钩子函数能够感知自身生命周期中的事件，并在相应的时刻到来时运行用户指定的程序代码。k8s在主容器的启动之后和停止之前提供了两个钩子函数：

1. post start：容器创建之后执行，如果失败了会重启容器
2. pre stop：容器终止之前执行，执行完成之后容器将成功终止，在其完成之前会阻塞删除容器的操作

钩子处理器支持3种方式定义动作：

1. Exec：在容器内执行一次命令
2. TCPSocket：在当前容器尝试访问指定的socket
3. HTTPGet：在当前容器中向某url发起http请求

```yaml
……
  lifecycle:
    postStart: 
      exec:
        command:
        - cat
        - /tmp/healthy    
      tcpSocket:
        port: 8080
      httpGet:
        path: / #URI地址
        port: 80 #端口号
        host: 192.168.5.3 #主机地址
        scheme: HTTP #支持的协议，http或者https
……
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hook-exec
  namespace: dev
spec:
  containers:
  - name: main-container
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    lifecycle:
      postStart: 
        exec: # 在容器启动的时候执行一个命令，修改掉nginx的默认首页内容
          command: ["/bin/sh", "-c", "echo postStart... > /usr/share/nginx/html/index.html"]
      preStop:
        exec: # 在容器停止之前停止nginx服务
          command: ["/usr/sbin/nginx","-s","quit"]
```

### 容器探测

容器探测用于检测容器中的应用实例是否正常工作，是保障业务可用性的一种传统机制。如果经过探测，实例的状态不符合预期，那么k8s就会把该问题实例“摘除”，不承担业务流量。k8s提供了两种探针来实现容器探测

1. liveness probes（存活性探针）：用于检测应用实例当前是否处于正常运行状态，如果不是，k8s会重启容器
2. readiness probes（就绪性探针）：用于检测应用实例当前是否可以接收请求，如果不能，k8s不会转发流量

- Exec：在容器内执行一次命令，如果命令执行的退出码为0，则认为程序正常，否则不正常
- TCPSocket：将会尝试访问一个用户容器的端口，如果能够建立这条连接，则认为程序正常，否则不正常
- HTTPGet：调用容器内Web应用的URL，如果返回的状态码在200和399之间，则认为程序正常，否则不正常

```yaml
……
  livenessProbe:
    exec:
      command:
      - cat
      - /tmp/healthy
    tcpSocket:
      port: 8080
    httpGet:
      path: / #URI地址
      port: 80 #端口号
      host: 127.0.0.1 #主机地址
      scheme: HTTP #支持的协议，http或者https
……
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-exec
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: 
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      exec:
        command: ["/bin/cat","/tmp/hello.txt"] # 执行一个查看文件的命令
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-tcpsocket
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: 
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      tcpSocket:
        port: 8080 # 尝试访问8080端口
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-httpget
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:
        scheme: HTTP
        port: 80 
        path: /
      initialDelaySeconds: 30 # 容器启动后30s开始探测
      timeoutSeconds: 5 # 探测超时时间为5s
```

1. initialDelaySeconds：容器启动后等待多少秒执行第一次探测
2. timeoutSeconds：探测超时时间。默认1秒，最小1秒
3. periodSeconds：执行探测的频率。默认是10秒，最小1秒
4. failureThreshold：连续探测失败多少次才被认定为失败。默认是3。最小值是1
5. successThreshold：连续探测成功多少次才被认定为成功。默认是1

### 重启策略

一旦容器探测出现了问题，k8s就会对容器所在的pod进行重启

1. Always（默认值）：容器失效时，自动重启该容器
2. OnFailure：容器终止运行且退出码不为0时重启
3. Never：不论状态为何，都不重启该容器

重启策略适用于pod对象中的所有容器，首次需要重启的容器，将在其需要时立即进行重启，随后再次需要重启的操作将有kubectl延迟一段时间后进行，且反复的重启操作的延迟时间依次为10s、20s、40s、80s、160s和300s，300s是最大延迟时长

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-restartpolicy
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:
        scheme: HTTP
        port: 80
        path: /hello
  restartPolicy: Never # 设置重启策略为Never
```

## Pod调度

在默认情况下，一个Pod在哪个Node节点上运行是由Scheduler组件采用相应的算法计算出来的，这个过程是不受人工控制的，但是在实际使用中，这并不满足需求。因为很多情况下，我们想控制某些Pod到达某些节点上，那么应该怎么做呢？

### 自动调度

运行在哪个节点上完全由Scheduler经过一系列的算法计算得出

### 定向调度

利用在pod上声明nodeName或者nodeSelector，以此将Pod调度到期望的node节点上

注意：这里的调度是强调度，即调用的目标node不存在时，也会向上面进行调度，只不过pod运行失败而已

1. NodeName：用于强制约束将Pod调度到指定Name的Node节点上，这种方式其实是直接跳过Scheduler的调度逻辑，直接将Pod调度到指定名称的节点

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeName: node1 # 指定调度到node1节点上
```


2. NodeSelector：用于将pod调度到添加了指定标签的node节点上，是通过k8s的label-selector机制实现的，即在pod创建之前，会由scheduler使用MatchNodeSelector调度策略进行label匹配，找出目标node，然后将pod调度到目标节点，匹配规则是强制约束

为node节点添加标签：kubectl label nodes node名称 key=value

存在问题：如果没有满足条件的node，那么pod将不会被运行，即使在集群中还有可用node列表也不行。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselector
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeSelector: 
    nodeenv: pro # 指定调度到具有nodeenv=pro标签的节点上
```

### 亲和性调度（Affinity）

亲和性调度在NodeSelector的基础上进行了扩展，可以通过配置的形式，实现优先选择满足条件的node进行调度，如果没有，也可以调度到不满足条件的节点上，使调度更加灵活

1. NodeAffinity（node亲和性）：以node为目标，解决pod可以调度到哪些node的问题
2. PodAffinity（pod亲和性）：以pod为目标，解决pod可以和哪些已存在的pod部署在同一个拓扑域中的问题
3. PodAntiAffinity（pod反亲和性）：以pod为目标，解决pod不能和哪些已存在pod部署在同一个拓扑域中的问题

- 亲和性：如果两个应用频繁交互，那就有必要利用亲和性让两个应用尽可能地靠近，这样就可以减少因网络通信而带来的性能损耗
- 反亲和性：当应用采用多副本部署时，有必要采用反亲和性让各个实例打散分布在各个node上，这样可以提高服务的高可用性

**NodeAffinity**

```yaml
pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution  Node节点必须满足指定的所有规则才可以，相当于硬限制
    nodeSelectorTerms  节点选择列表
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operat or 关系符 支持Exists, DoesNotExist, In, NotIn, Gt, Lt
  preferredDuringSchedulingIgnoredDuringExecution 优先调度到满足指定的规则的Node，相当于软限制 (倾向)
    weight 倾向权重，在范围1-100。
    preference   一个节点选择器项，与相应的权重相关联
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist, Gt, Lt
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    nodeAffinity: #设置node亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
        nodeSelectorTerms:
        - matchExpressions: # 匹配nodeenv的值在["xxx","yyy"]中的标签
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-preferred
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    nodeAffinity: #设置node亲和性
      preferredDuringSchedulingIgnoredDuringExecution: # 软限制
      - weight: 1
        preference:
          matchExpressions: # 匹配nodeenv的值在["xxx","yyy"]中的标签(当前环境没有)
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

**注意**

1. 如果同时定义了nodeSelector和nodeAffinity，那么必须两个条件都得到满足，Pod才能运行在指定的Node上
2. 如果nodeAffinity指定了多个nodeSelectorTerms，那么只需要其中一个能够匹配成功即可
3. 如果一个nodeSelectorTerms中有多个matchExpressions ，则一个节点必须满足所有的才能匹配成功
4. 如果一个pod所在的Node在Pod运行期间其标签发生了改变，不再符合该Pod的节点亲和性需求，则系统将忽略此变化

**PodAffinity**

PodAffinity主要实现以运行的Pod为参照，实现让新创建的Pod跟参照pod在一个区域的功能

```yaml
pod.spec.affinity.podAffinity
  requiredDuringSchedulingIgnoredDuringExecution  硬限制
    namespaces       指定参照pod的namespace
    topologyKey      指定调度作用域
    labelSelector    标签选择器
      matchExpressions  按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist.
      matchLabels    指多个matchExpressions映射的内容
  preferredDuringSchedulingIgnoredDuringExecution 软限制
    podAffinityTerm  选项
      namespaces      
      topologyKey
      labelSelector
        matchExpressions  
          key    键
          values 值
          operator
        matchLabels 
    weight 倾向权重，在范围1-100
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podantiaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    podAntiAffinity: #设置pod亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
      - labelSelector:
          matchExpressions: # 匹配podenv的值在["pro"]中的标签
          - key: podenv
            operator: In
            values: ["pro"]
        topologyKey: kubernetes.io/hostname
```

上面配置表达的意思是：新Pod必须要与拥有标签podenv=pro的pod不在同一Node上

### 污点（Taints）

通过在pod上添加属性，来确定pod是否要调度到指定的node上，也可以在node上添加污点属性，来决定是否允许pod调度过来

污点格式：key=value:effect，key和value是污点的标签，effect描述污点的作用

1. PreferNoSchedule：k8s将尽量避免把pod调度到具有该污点的node上，除非没有其他节点可调度
2. NoSchedule：k8s将不会把pod调度到具有该污点的node上，但不会影响当前node已存在的pod
3. NoExecute：k8s将不会把pod调度到具有该污点的node上，同时也会将node上已存在的pod隔离

![image-20200605021606508](https://gitee.com/yooome/golang/raw/main/22-k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B-%E8%B0%83%E6%95%B4%E7%89%88/Kubenetes.assets/image-20200605021831545.png)

```yaml
# 设置污点
kubectl taint nodes node1 key=value:effect

# 去除污点
kubectl taint nodes node1 key:effect-

# 去除所有污点
kubectl taint nodes node1 key-
```

示例：

- 准备节点node1
- 为node1节点设置一个污点: tag=heima:PreferNoSchedule；然后创建pod1( pod1 可以 )
- 修改为node1节点设置一个污点: tag=heima:NoSchedule；然后创建pod2(pod1 正常 pod2 失败 )
- 修改为node1节点设置一个污点: tag=heima:NoExecute；然后创建pod3 ( 3个pod都失败 )

```yaml
# 为node1设置污点(PreferNoSchedule)
[root@k8s-master01 ~]# kubectl taint nodes node1 tag=heima:PreferNoSchedule

# 创建pod1
[root@k8s-master01 ~]# kubectl run taint1 --image=nginx:1.17.1 -n dev
[root@k8s-master01 ~]# kubectl get pods -n dev -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP           NODE   
taint1-7665f7fd85-574h4   1/1     Running   0          2m24s   10.244.1.59   node1    

# 为node1设置污点(取消PreferNoSchedule，设置NoSchedule)
[root@k8s-master01 ~]# kubectl taint nodes node1 tag:PreferNoSchedule-
[root@k8s-master01 ~]# kubectl taint nodes node1 tag=heima:NoSchedule

# 创建pod2
[root@k8s-master01 ~]# kubectl run taint2 --image=nginx:1.17.1 -n dev
[root@k8s-master01 ~]# kubectl get pods taint2 -n dev -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP            NODE
taint1-7665f7fd85-574h4   1/1     Running   0          2m24s   10.244.1.59   node1 
taint2-544694789-6zmlf    0/1     Pending   0          21s     <none>        <none>   

# 为node1设置污点(取消NoSchedule，设置NoExecute)
[root@k8s-master01 ~]# kubectl taint nodes node1 tag:NoSchedule-
[root@k8s-master01 ~]# kubectl taint nodes node1 tag=heima:NoExecute

# 创建pod3
[root@k8s-master01 ~]# kubectl run taint3 --image=nginx:1.17.1 -n dev
[root@k8s-master01 ~]# kubectl get pods -n dev -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED 
taint1-7665f7fd85-htkmp   0/1     Pending   0          35s   <none>   <none>   <none>    
taint2-544694789-bn7wb    0/1     Pending   0          35s   <none>   <none>   <none>     
taint3-6d78dbd749-tktkq   0/1     Pending   0          6s    <none>   <none>   <none>     
```

注意：使用kubeadm搭建的集群，默认就会给master节点添加一个污点标记,所以pod就不会调度到master节点上

### 容忍（Toleration）

可以在node上添加污点用于拒绝pod调度上来，但是想将一个pod调度到一个有污点的node上去，这时就需要容忍

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  tolerations:      # 添加容忍
  - key: "tag"        # 要容忍的污点的key
    operator: "Equal" # 操作符
    value: "heima"    # 容忍的污点的value
    effect: "NoExecute"   # 添加容忍的规则，这里必须和标记的污点规则相同
    #tolerationSeconds: 30   # 容忍时间, 当effect为NoExecute时生效，表示pod在Node上的停留时间
```

```yaml
# 添加容忍之前的pod
[root@k8s-master01 ~]# kubectl get pods -n dev -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED 
pod-toleration   0/1     Pending   0          3s    <none>   <none>   <none>           

# 添加容忍之后的pod
[root@k8s-master01 ~]# kubectl get pods -n dev -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED
pod-toleration   1/1     Running   0          3s    10.244.1.62   node1   <none>        
```
