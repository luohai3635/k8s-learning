## 基础组件


### etcd
1. 分布式的kv存储介质
2. 采用`RAFT` 协议来同步集群的状态，比`zookeeper` 新、好
3. 存储 k8s 集群的状态
4. 数据存储在每个节点上，并不能通过增加节点的方式来扩容，可以通过分片的方式（使用别的工具）
5. 在k8s中只能通过`kube-apiserver`来访问

### kube-apiserver
1. k8s 中的唯一api服务，所有的集群操作，客户端操作都走这个服务提供的api接口
2. 有天然的基于TLS的https访问和http访问（http访问在生产环境中不推荐）
3. ssl访问经过：认证、授权、准入控制（`Admission control`）,认证解决的是账号对于整个系统的访问权限，授权解决的是对系统服务的访问权限，准入控制解决的是非`GET`请求是否满足系统对api输入参数的要求


### kube-scheduler
1. 通过`kube-apiserver` 对`etcd` 中Pod的配置进行监听，将 Pod调度到合适的Node中
2. 调度两阶段：
 * `predicate`:过滤不符合条件的节点
 * `priority`:优先级排序，选择优先级最高的节点
3.三种方式将Node调度到指定的Node上
 * `nodeSelector`:只调度到匹配指定label的Node上
 * `nodeAffinity`:功能更丰富的Node选择器，比如支持集合操作
 * `podAffinity`: 调度到满足条件的Pod所在的Node上
4. 两种方式用于保证Pod不被调度到不合适的Node上：
 * `Taints`:保证Pod不被调度到不合适的Node上
 * `Toleration`:保证Pod不被调度到不合适的Pod所在的Node上

5. 从`v1.8` 开始，支持优先级调度，保证高优先级的Pod有限调度
6. 多调度器：可以在集群中部署多个调度器，可以部署自定义的调度器，即调度器支持扩展
7. 影响调度的其他因素包括内存压力、磁盘压力；为了保证`
Critical Pod` 的正常运行，在异常状态时会将其调度到别的节点执行


### kube-controller-manager
1. 包含`kube-controller-manager`和`cloud-controller-manager` ,通过`kube-apiserver`来监控整个集群的状态，并室集群处于预期的工作状态
2. `kube-controller-manager`包含多种控制器，可分为必须启动、默认启动、默认不启动
3. `cloud-controller-manager`在kubernates启动`Cloud Provider`之后才需要，用来配合云服务提供商的控制
4. `Metrics`: `Controller manager metrics` 提供了控制器内部逻辑的性能度量
5. 可以通过启动时配置，使得`Controller Manager`会执行选主算法选出主节点，只有主节点才会启动所有的控制器，其他节点只执行选主算法
6. 从`v1.7`开始，所有监控资源变化情况的调用推荐使用`informer`。其提供了基于时间通知的只读缓存机制、可以祖册资源变化的回调函数，极大减少API的调用。
7. `Node eviction`:Node 控制器在节点异常之后，会按照一定的默认的速率驱逐Node。并将Node根据Zone 划分为不同的组，再跟进Zone的状态调整速率，三种状态：
 * `Normal`
 * `PartialDisruption`
 * `FullDisruption`

### kubelet
接受和执行master发来的指令，管理和监控Pods、Container，定期向Master汇报节点状态
#### 节点管理
1. 节点自注册
 若节点没有配置自注册信息，需要用户自己配置Node的资源信息，并提供apiserver的位置信息
2. 定时向apiserver发送节点信息，apiserver将信息写入etcd

#### Pod管理
##### 获取Pod清单
kubelet通过PodSpec的方式工作，PodSpec是描述一个Pod的YAML或JSON对象。可以通过如下途径向kublet提供PodSpec：
1. 文件
2. HTTP endpoint(URL)
3. Api Server
4. HTTP server

##### 通过API Server获取Pod清单
kubelet 通过`API server Client` 使用`Watch`,`List`的方式监听`/registry/nodes/$` 和`/registry/nodes` 目录，将获取到的信息同步到本地缓存中。

`kubelet` 会针对监听到的增加、删除、修改Pod的信息作出对应的动作。
修改流程：拉去Pod清单、检查指定Pod中`Pause`容器是否启动，若没有启动，则将所有的容器停掉，启动`kubernates/pause` 来接管Pod中的所有其他容器的网络；计算容器hash并与镜像列表中的做对比，若一致则不变，若不一致则删掉重新拉去镜像创建容器并停掉相关联的`Pause`容器。

> 所有不通过Apiserver创建的Pod称为 Static Pod，该类Pod的状态会被kubelet汇报给apiserver，apiserver为该Static Pod创建一个 Mirror Pod，真实反映Static Pod的状态。当Static Pod被删除的时候，Mirror Pod也同时被删除


#### 容器健康检查
Pod 通过两类探针兼容容器的健康状态
1. LivenessProb：判断容器是否健康
 * ExecAction:在容器内部执行一个命令，如果命令的退出状态为0，表示容器健康
 * TCPSocketAction:通过容器的IP地址和端口号进行TCP检查
 * HTTPGetAction：通过容器的IP地址和端口号及路径调用HTTP GET方法，返回状态码大于200小于400 ，则认为容器健康
2. ReadinessProb：判断容器是否启动完成并且准备接受请求


#### cAdvisor
kubelet 通过cAdvisor 获取节点和容器的数据，并将其汇报给heapster，heapster通过带着关联标签的Pod分类这些信心，并将其推送到一个可配置的后端进行存储和可视化。

#### Kubelet Eviction
kubelet 监控节点的状态，使用驱逐机制方式节点的资源被耗尽。

kubelet 定期检查系统资源是否达到预期配置的阈值，在达到阈值时执行如下两种驱逐：
* 软驱逐：在配置的驱逐宽限期之后驱逐
* 硬驱逐：直接驱逐

驱逐动作包括：
* 回收节点资源
* 驱逐用户Pod


#### 容器运行时
容器运行时（Container Runtime）是k8s最重要的组件之一，负责真正管理镜像和容器的生命周期。Kubelet 通过`Container Runtime Interface(CRI)`与容器运行时交互，以管理镜像和容器。

Container Runtime Interface（CRI）是 Kubernetes v1.5 引入的容器运行时接口，它将Kubelet 与容器运行时解耦，将原来完全面向 Pod 级别的内部接口拆分成面向 Sandbox 和 Container 的 gRPC 接口，并将镜像管理和容器管理分离到不同的服务。

#### Pod启动流程




