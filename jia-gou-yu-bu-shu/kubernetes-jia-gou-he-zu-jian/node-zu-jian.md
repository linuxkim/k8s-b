# Node 组件

Kubernetes Node是运行节点，提供Kubernetes运行时环境，用于运行管理业务的容器，包含以下组件。

## kubelet

负责管控容器，Kubelet会从Kubernetes API Server接收Pod的创建请求，启动和停止容器，监控运行状态并汇报给Kubernetes API Server。

## kube-proxy

负责为Pod创建代理服务，Kubernetes Proxy会从Kubernetes API Server获取所有的Service，并根据Service信息创建代理服务，实现Service到Pod的请求路由和转发，从而实现Kubernetes层级的虚拟转发网络。

## docker

Docker用于运行容器。

