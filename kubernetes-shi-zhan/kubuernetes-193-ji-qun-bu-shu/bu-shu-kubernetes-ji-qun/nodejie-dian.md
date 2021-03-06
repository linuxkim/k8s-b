# 配置 kubelet

### 创建 kubelet kubeconfig 文件

\# 配置集群

```
[root@k8s-master ~]#  cd /etc/kubernetes/ssl/
[root@k8s-master ssl]#  kubectl config set-cluster kubernetes \
  --certificate-authority=k8s-root-ca.pem \
  --embed-certs=true \
  --server=https://172.20.20.1:6443 \
  --kubeconfig=bootstrap.kubeconfig
```

\# 配置客户端认证

```
[root@k8s-master ssl]#  kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
```

\# 配置关联

```
[root@k8s-master ssl]#  kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
```

\# 配置默认关联

```
[root@k8s-master ssl]#  kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

\#下发 bootstrap.kubeconfig 文件

```
[root@k8s-master ssl]#  scp /etc/kubernetes/ssl/bootstrap.kubeconfig 172.20.20.2:/etc/kubernetes/ssl
[root@k8s-master ssl]#  scp /etc/kubernetes/ssl/bootstrap.kubeconfig 172.20.20.3:/etc/kubernetes/ssl
```

### 创建 kubelet.service 文件

\# 创建 kubelet 目录

```
[root@k8s-node-1 ~]# mkdir /var/lib/kubelet
[root@k8s-node-2 ~]# mkdir /var/lib/kubelet
```

```
#k8s-node-1

[root@k8s-node-1 ~]# cat >/lib/systemd/system/kubelet.service  <<'HERE'
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service
[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
--address=172.20.20.2 \
--hostname-override=k8s-node-1 \
#--pod-infra-container-image=k8s-registry.local/public/pod-infrastructure:sfv1 \
--pod-infra-container-image=172.20.88.6/test/pod-infrastructure:sfv1 \
--experimental-bootstrap-kubeconfig=/etc/kubernetes/ssl/bootstrap.kubeconfig \
--kubeconfig=/etc/kubernetes/ssl/kubelet.kubeconfig \
--cert-dir=/etc/kubernetes/ssl \
--runtime-cgroups=/systemd/system.slice \
--kubelet-cgroups=/systemd/system.slice \
--hairpin-mode promiscuous-bridge \
--allow-privileged=true \
--serialize-image-pulls=false \
--logtostderr=true \
--cgroup-driver=systemd \
--cluster_dns=10.6.0.2 \
--cluster_domain=cluster.local \
--fail-swap-on=false \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
HERE
```

> k8s-node-1    本机hostname
>
> 10.6.0.2       预分配的 dns 地址
>
> cluster.local   为 kubernetes 集群的 domain
>
> pod-infrastructure:sfv1  这个是 pod 的基础镜像，既 gcr 的 gcr.io/google\_containers/pause-amd64:3.0 镜像， 下载下来修改为自己的仓库中的比较快。

```
#k8s-node-1

[root@k8s-node-1 ~]# cat >/lib/systemd/system/flanneld.service  <<'HERE'
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service
[Service]
Type=notify
ExecStart=/usr/local/bin/flanneld \
-etcd-cafile=/etc/kubernetes/ssl/k8s-root-ca.pem \
-etcd-certfile=/etc/kubernetes/ssl/etcd.pem \
-etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \
-etcd-endpoints=https://172.20.20.4:2379,https://172.20.20.5:2379,https://172.20.20.6:2379 \
-etcd-prefix=/kubernetes/network \
-iface=ens160
ExecStartPost=/usr/local/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure
[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
HERE
```

```
#k8s-node-2

[root@k8s-node-2 ~]# cat >/lib/systemd/system/kubelet.service  <<'HERE'
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service
[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
--address=172.20.20.3 \
--hostname-override=k8s-node-2 \
#--pod-infra-container-image=k8s-registry.local/public/pod-infrastructure:sfv1 \
--pod-infra-container-image=172.20.88.6/test/pod-infrastructure:sfv1 \
--experimental-bootstrap-kubeconfig=/etc/kubernetes/ssl/bootstrap.kubeconfig \
--kubeconfig=/etc/kubernetes/ssl/kubelet.kubeconfig \
--cert-dir=/etc/kubernetes/ssl \
--runtime-cgroups=/systemd/system.slice \
--kubelet-cgroups=/systemd/system.slice \
--hairpin-mode promiscuous-bridge \
--allow-privileged=true \
--serialize-image-pulls=false \
--logtostderr=true \
--cgroup-driver=systemd \
--cluster_dns=10.6.0.2 \
--cluster_domain=cluster.local \
--fail-swap-on=false \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
HERE
```

```
#k8s-node-2

[root@k8s-node-2 ~]# cat >/lib/systemd/system/flanneld.service  <<'HERE'
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service
[Service]
Type=notify
ExecStart=/usr/local/bin/flanneld \
-etcd-cafile=/etc/kubernetes/ssl/k8s-root-ca.pem \
-etcd-certfile=/etc/kubernetes/ssl/etcd.pem \
-etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \
-etcd-endpoints=https://172.20.20.4:2379,https://172.20.20.5:2379,https://172.20.20.6:2379 \
-etcd-prefix=/kubernetes/network \
-iface=ens160
ExecStartPost=/usr/local/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure
[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
HERE
```

### 启动node节点服务

```
[root@k8s-node-1 ~]# systemctl daemon-reload && systemctl start flanneld docker kubelet && systemctl enable flanneld docker kubelet
[root@k8s-node-2 ~]# systemctl daemon-reload && systemctl start flanneld docker kubelet && systemctl enable flanneld docker kubelet
```

> kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper 角色，然后 kubelet 才有权限创建认证请求\(certificatesigningrequests\)。

\# 在master机器上执行，授权kubelet-bootstrap角色

```
[root@k8s-master ~]# kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

\# 查看 csr 的名称

```
[root@k8s-master ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-DRSFk1kmkeaIVuPW5WCjSgMrX61DTmpZ_lC0L6Ms0Y0   1m        kubelet-bootstrap   Pending
node-csr-nzvjyp4vNcTGyXRPKUEFiT3D0li2ytWxjFKWkLwqh4U   1m        kubelet-bootstrap   Pending
```

\# 增加 认证

```
[root@k8s-master ~]# kubectl get csr | awk '/Pending/ {print $1}' | xargs kubectl certificate approve

[root@k8s-master ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-DRSFk1kmkeaIVuPW5WCjSgMrX61DTmpZ_lC0L6Ms0Y0   2m        kubelet-bootstrap   Approved,Issued
node-csr-nzvjyp4vNcTGyXRPKUEFiT3D0li2ytWxjFKWkLwqh4U   1m        kubelet-bootstrap   Approved,Issued
```

\#验证 nodes

```
[root@k8s-master ~]# kubectl get nodes
NAME                     STATUS     ROLES     AGE       VERSION
k8s-node-1               Ready      <none>    40s       v1.9.3
k8s-node-2               Ready      <none>    40s       v1.9.3
```

\#kubelet启动成功以后会自动生成配置文件与密钥

```
[root@k8s-node-1 ~]# ls -l /etc/kubernetes/ssl/kubelet*
-rw-r--r-- 1 root root 1407 4月   4 17:42 /etc/kubernetes/ssl/kubelet-client.crt
-rw------- 1 root root  227 4月   4 17:40 /etc/kubernetes/ssl/kubelet-client.key
-rw-r--r-- 1 root root 1164 4月   4 16:46 /etc/kubernetes/ssl/kubelet.crt
-rw------- 1 root root 1675 4月   4 16:46 /etc/kubernetes/ssl/kubelet.key
-rw------- 1 root root 3210 4月   4 17:42 /etc/kubernetes/ssl/kubelet.kubeconfig
```

> 如果 csr 被删除了，请删除如上文件，并重启 kubelet 服务



