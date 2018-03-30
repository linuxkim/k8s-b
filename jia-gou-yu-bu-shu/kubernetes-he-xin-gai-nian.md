# Kubernetes 的核心概念 {#kubernetes核心概念}



## Pod {#pod}

Pod是一组紧密关联的容器集合，Pod包含的容器运行在同一台宿主机上，这些容器使用相同的网络命名空间、IP地址和端口，相互之间能通过localhost来发现和通信。它们还可以共享Volume（存储卷空间），在Kubernetes中创建、调度和管理的最小单位是Pod，Pod通过提供更高层次的抽象，提供了更加灵活的部署和管理模式。



## Node {#node}

Node可以认为是Pod的宿主机，为了管理Pod，每个Node节点上至少要运行container runtime（比如docker或者rkt）、`kubelet`和`kube-proxy`服务。



## Service {#service}

Service是应用服务的抽象，通过labels为应用提供负载均衡和服务发现。Service对外暴露一个统一的访问接口，外部服务不需要了解后端容器的运行。



# Label

Label是识别Kubernetes对象的标签，以key/value的方式附加到对象上。Label不提供唯一性，并且实际上经常是很多对象（如Pods）都使用相同的label来标志具体的应用。

Label定义好后其他对象可以使用Label Selector来选择一组相同label的对象（比如ReplicaSet和Service用label来选择一组Pod）。Label Selector支持以下几种方式：

* 等式，如
  `app=nginx`
  和
  `env!=production`
* 集合，如
  `env in (production, qa)`
* 多个label（它们之间是AND关系），如
  `app=nginx,env=test` 



## Annotations {#annotations}

Annotations是key/value形式附加于对象的注解。不同于Labels用于标志和选择对象，Annotations则是用来记录一些附加信息，以便于外部工具进行查找。



## Namespace {#namespace}

Namespace是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。常见的pods, services, replication controllers和deployments等都是属于某一个namespace的（默认是default），而node, persistentVolumes等则不属于任何namespace。

