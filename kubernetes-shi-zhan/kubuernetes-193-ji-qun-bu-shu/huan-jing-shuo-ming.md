# 基础环境

| 环境 | 版本 |
| :--- | :--- |
| CentOS | CentOS Linux release 7.3.1611 \(Core\) |
| Kernel | 4.4.122-1.el7.elrepo.x86\_64 |
| Docker | 1.13.1 |
| Kubernetes | 1.9.3 |

# 网络

| 集群网络 | 10.8.0.0/16 |
| :--- | :--- |
| svc网络 | 10.6.0.0/16 |
| 物理网络 | 172.20.20.0/24 |

# 业务分布

| 主机 | IP | 业务 |
| :--- | :--- | :--- |
| k8s-master | 172.20.20.1 | flanneld,kube-apiserver,kube-controller-manager,kube-scheduler |
| k8s-node-1 | 172.20.20.2 | flanneld,docker,kubelet |
| k8s-node-2 | 172.20.20.3 | flanneld,docker,kubelet |
| k8s-etcd-1 | 172.20.20.4 | etcd |
| k8s-etcd-2 | 172.20.20.5 | etcd |
| k8s-etcd-3 | 172.20.20.6 | etcd |





