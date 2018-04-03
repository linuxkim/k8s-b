### 配置 kube-apiserver

> kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证 kubelet 请求中的 token 是否与它配置的 token 一致，如果一致则自动为 kubelet生成证书和秘钥。

\# 生成 token

```
$ export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
```

\# 创建 token.csv 文件

```
$ cat >/etc/kubernetes/ssl/token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

\# 生成高级审核配置文件

```
$ cat >>/etc/kubernetes/ssl/audit-policy.yaml <<EOF
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
- level: Metadata
EOF
```

### 创建 kube-apiserver.service 文件

```
cat >/lib/systemd/system/kube-apiserver.service  <<'HERE'
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
[Service]
ExecStart=/usr/local/bin/kube-apiserver \
--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
--advertise-address=172.20.20.1 \
--bind-address=172.20.20.1 \
--insecure-bind-address=127.0.0.1 \
--kubelet-https=true \
--runtime-config=rbac.authorization.k8s.io/v1beta1 \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth \
--token-auth-file=/etc/kubernetes/ssl/token.csv \
--service-cluster-ip-range=10.6.0.0/16 \
--service-node-port-range=300-9000 \
--tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
--tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
--client-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
--service-account-key-file=/etc/kubernetes/ssl/k8s-root-ca-key.pem \
--etcd-cafile=/etc/kubernetes/ssl/k8s-root-ca.pem \
--etcd-certfile=/etc/kubernetes/ssl/etcd.pem \
--etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \
--etcd-servers=https://172.20.20.4:2379,https://172.20.20.5:2379,https://172.20.20.6:2379 \
--enable-swagger-ui=true \
--allow-privileged=true \
--audit-policy-file=/etc/kubernetes/ssl/audit-policy.yaml \
--apiserver-count=3 \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/var/lib/audit.log \
--event-ttl=1h \
--v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
HERE
```

> k8s 1.8 开始需要 添加 ：
>
> --authorization-mode=Node
>
> --admission-control=NodeRestriction
>
> --audit-policy-file=/etc/kubernetes/ssl/audit-policy.yaml
>
> 这里面要注意的是 --service-node-port-range=300-9000 这个地方是 映射外部端口时 的端口范围，随机映射也在这个范围内映射，指定映射端口必须也在这个范围内。

### 配置 kube-controller-manager

\# 创建 kube-controller-manager.service 文件

```
cat >/lib/systemd/system/kube-controller-manager.service  <<'HERE'
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
--address=127.0.0.1 \
--master=http://127.0.0.1:8080 \
--allocate-node-cidrs=true \
--service-cluster-ip-range=10.6.0.0/16 \
--cluster-cidr=10.8.0.0/16 \
--cluster-name=kubernetes \
--cluster-signing-cert-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
--cluster-signing-key-file=/etc/kubernetes/ssl/k8s-root-ca-key.pem \
--service-account-private-key-file=/etc/kubernetes/ssl/k8s-root-ca-key.pem \
--root-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
--leader-elect=true \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
HERE
```

### 配置 kube-scheduler

\# 创建 kube-cheduler.service 文件

```
cat >/lib/systemd/system/kube-scheduler.service  <<'HERE'
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-scheduler \
--address=127.0.0.1 \
--master=http://127.0.0.1:8080 \
--leader-elect=true \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
HERE

#flanneld.service
cat >/lib/systemd/system/flanneld.service  <<'HERE'
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

\#创建master /root/.kube 目录,复制超级admin授权config

```
$ mkdir -p /root/.kube
$ cp -f /etc/kubernetes/ssl/kubeconfig  /root/.kube/config
```

### 启动master节点服务

```
systemctl daemon-reload && systemctl start flanneld docker kube-apiserver kube-controller-manager kube-scheduler && systemctl enable flanneld docker kube-apiserver kube-controller-manager kube-scheduler
```

\#验证 Master 节点

```
$ kubectl get componentstatuses
```



