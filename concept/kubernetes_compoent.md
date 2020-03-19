# Kubernetes各个组件介绍


## Kubernetes 集群组件交互架构
如下图所示为Kubernetes集群中各个组件交互流程以及整体集群架构    
![kubernetes-control-plane.png](https://upload-images.jianshu.io/upload_images/17904159-fb8a24a90c9faeaa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 组件功能介绍

### kube-master[控制节点]:
如图所示为master节点的工作流：
![master.png](https://upload-images.jianshu.io/upload_images/17904159-79a056a93b8b6f6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Master定义了Kubernetes 集群Master/API Server的主要声明，包括Pod Registry、Controller Registry、Service Registry、Endpoint Registry、Minion Registry、Binding Registry、RESTStorage以及Client, 是client(Kubecfg)调用Kubernetes API，管理Kubernetes主要构件Pods、Services、Minions、容器的入口。Master由API Server、Scheduler以及Registry等组成。从下图可知Master的工作流主要分以下步骤：

- **Kubecfg** 将特定的请求，比如创建Pod，发送给Kubernetes Client。
- **Kubernetes Client** 将请求发送给API server。
- **API Server**根据请求的类型，比如创建Pod时storage类型是pods，然后依此选择何种REST Storage API对请求作出处理。
- **REST Storage API** 对的请求作相应的处理。将处理的结果存入高可用键值存储系统Etcd中。在API Server响应Kubecfg的请求后。
- **Scheduler** 会根据Kubernetes Client获取集群中运行Pod及Minion信息。依据从Kubernetes Client获取的信息，Scheduler将未分发的Pod分发到可用的Minion节点上。


### API Server[资源操作入口]：
1. 提供了资源对象的唯一操作入口，其他所有组件都必须通过它提供的API来操作资源数据，只有API Server与存储通信，其他模块通过API Server访问集群状态。
第一，是为了保证集群状态访问的安全。
第二，是为了隔离集群状态访问的方式和后端存储实现的方式：API Server是状态访问的方式，不会因为后端存储技术etcd的改变而改变。

2. 作为kubernetes系统的入口，封装了核心对象的增删改查操作，以RESTful接口方式提供给外部客户和内部组件调用。对相关的资源数据“全量查询”+“变化监听”，实时完成相关的业务功能。
### Controller Manager[内部管理控制中心]：
1. 实现集群故障检测和恢复的自动化工作，负责执行各种控制器，主要有：
- **endpoint-controller：** 定期关联service和pod(关联信息由endpoint对象维护)，保证service到pod的映射总是最新的。
- **replication-controller：** 定期关联replicationController和pod，保证replicationController定义的复制数量与实际运行pod的数量总是一致的。
### Scheduler[集群分发调度器]：
1. Scheduler收集和分析当前Kubernetes集群中所有Minion/Node节点的资源(内存、CPU)负载情况，然后依此分发新建的Pod到Kubernetes集群中可用的节点。
2. 实时监测Kubernetes集群中未分发和已分发的所有运行的Pod。
3. 监测Minion/Node节点信息，由于会频繁查找Minion/Node节点，Scheduler会缓存一份最新的信息在本地。
4. 将已分发Pod到指定的Minion/Node节点，把Pod相关的信息Binding写回API Server。

### Kubelet[节点上的Pod管家]：
1. 负责Node节点上pod的创建、修改、监控、删除等全生命周期的管理，
以及定时上报本Node的状态信息给API Server。
2. kubelet是Master API Server和Minion/Node之间的桥梁，接收Master API Server分配给它的commands和work，通过kube-apiserver间接与Etcd集群交互，读取配置信息。
具体的工作如下：

- 1) 设置容器的环境变量、给容器绑定Volume、给容器绑定Port、根据指定的Pod运行一个单一容器、给指定的Pod创建network 容器。

- 2) 同步Pod的状态、从cAdvisor获取container info、 pod info、 root info、 machine info。

- 3) 在容器中运行命令、杀死容器、删除Pod的所有容器
### Proxy[负载均衡、路由转发]：
1. Proxy是为了解决外部网络能够访问跨机器集群中容器提供的应用服务而设计的，运行在每个Minion/Node上。Proxy提供TCP/UDP sockets的proxy，每创建一种Service，Proxy主要从etcd获取Services和Endpoints的配置信息（也可以从file获取），然后根据配置信息在Minion/Node上启动一个Proxy的进程并监听相应的服务端口，当外部请求发生时，Proxy会根据Load Balancer将请求分发到后端正确的容器处理。
3. Proxy不但解决了同一主宿机相同服务端口冲突的问题，还提供了Service转发服务端口对外提供服务的能力，Proxy后端使用了随机、轮循负载均衡算法。
### kubectl[集群管理命令行工具集]：
1. 通过客户端的kubectl命令集操作，API Server响应对应的命令结果，从而达到对kubernetes集群的管理。