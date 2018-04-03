# 配置 kubelet



### 创建 kubelet kubeconfig 文件

\# 配置集群  

```
$ cd /etc/kubernetes/ssl/

$ kubectl config set-cluster kubernetes \
  --certificate-authority=k8s-root-ca.pem \
  --embed-certs=true \
  --server=https://172.20.20.1:6443 \
  --kubeconfig=bootstrap.kubeconfig
```

\# 配置客户端认证

```
$ kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
```

\# 配置关联  

```
$ kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
```

\# 配置默认关联 

```
$ kubectl config use-context default --kubeconfig=bootstrap.kubeconfig  
```

\#下发 bootstrap.kubeconfig 文件

```
$ scp /etc/kubernetes/ssl/bootstrap.kubeconfig 172.20.20.2:/etc/kubernetes/ssl
$ scp /etc/kubernetes/ssl/bootstrap.kubeconfig 172.20.20.3:/etc/kubernetes/ssl
```

### 创建 kubelet.service 文件

\# 创建 kubelet 目录

```
$ mkdir /var/lib/kubelet
```

```
#k8s-node-1

cat >/lib/systemd/system/kubelet.service  <<'HERE'
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service
[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
--address=172.20.20.2 \
--hostname-override=sz-fc-cs-k8s-node-20-2 \
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



