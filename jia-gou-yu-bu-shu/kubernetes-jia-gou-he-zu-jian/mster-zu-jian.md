# Master 组件

Kubernetes Mster 作为控制节点，调度管理整个系统，包含以下组件。

## kube-apiserver

作为Kubernetes系统的入口，其封装了核心对象的增删改查操作，何的资源请求，调用操作都是通过kube-apiserver提供的接口进行。

## kube-scheduler

负责集群的资源调度，为新建的Pod选择一个Node。

## kube-controller-manager

* 负责执行各种控制器，目前已经实现很多控制器来保证Kubernetes的正常运行，主要包含的控制器如下：
* 节点（Node）控制器。
* 副本（Replication）控制器：负责维护系统中每个副本中的Pod。
* 端点（Endpoints）控制器：填充Endpoints对象（即连接Services＆Pods）。
* Service Account和Token控制器：为新的Namespace创建默认账户访问API Token。



