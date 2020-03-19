# Kubernetes容器云平台技术栈

Kubernetes 搭建的容器云平台通过Docker及分布式存储等技术提供应用运行平台，从而实现运维自动化，快速部署应用、弹性伸缩和动态调整应用环境资源，提高研发运营效率。

|功能组成部分|工具|
|----|----|
|应用载体|	Docker|
|编排容器管理工具|Kubernetes|
|参数配置管理|ETCD|
|网络管理|Flannel、Calico 等|
|存储管理|Ceph、Gluster Fs 等|
|底层实现|Linux内核的Namespace[资源隔离]和CGroups[资源控制]|

- **Namespace[资源隔离]**
Namespaces机制提供一种资源隔离方案。PID,IPC,Network等系统资源不再是全局性的，而是属于某个特定的Namespace。每个namespace下的资源对于其他namespace下的资源都是透明，不可见的。
- **CGroups[资源控制]**
CGroup（control group）是将任意进程进行分组化管理的Linux内核功能。CGroup本身是提供将进程进行分组化管理的功能和接口的基础结构，I/O或内存的分配控制等具体的资源管理功能是通过这个功能来实现的。CGroups可以限制、记录、隔离进程组所使用的物理资源（包括：CPU、memory、IO等），为容器实现虚拟化提供了基本保证。CGroups本质是内核附加在程序上的一系列钩子（hooks），通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。
