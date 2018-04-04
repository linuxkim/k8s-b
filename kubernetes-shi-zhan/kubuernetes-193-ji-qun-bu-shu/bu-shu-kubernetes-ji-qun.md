# Kubernetes集群部署

\#创建 admin 证书

> 需要为kubectl 与 kube-apiserver 之间的https通信提供 TLS 证书和秘钥。

```
[root@k8s-master ~]# cat >/etc/kubernetes/ssl/admin-csr.json  <<'HERE'
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
HERE
```

\# 生成 admin 证书和私钥

```
[root@k8s-master ~]# cd /etc/kubernetes/ssl/
[root@k8s-master ssl]# cfssl gencert --ca k8s-root-ca.pem --ca-key k8s-root-ca-key.pem --config k8s-gencert.json --profile kubernetes admin-csr.json | cfssljson --bare admin
```

### 配置 kubectl kubeconfig 文件

> 生成集群管理员admin kubeconfig配置文件供kubectl调用

\# 配置 kubernetes 集群

```
[root@k8s-master ssl]# kubectl config set-cluster kubernetes \
    --certificate-authority=k8s-root-ca.pem\
    --embed-certs=true \
    --server=https://172.20.20.1:6443 \
    --kubeconfig=./kubeconfig
```

\# 配置 客户端认证

```
[root@k8s-master ssl]# kubectl config set-credentials kubernetes-admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=./kubeconfig
```

\# 配置关联

```
[root@k8s-master ssl]# kubectl config set-context kubernetes-admin@kubernetes \
    --cluster=kubernetes \
    --user=kubernetes-admin \
    --kubeconfig=./kubeconfig
```

\# 配置默认关联

```
[root@k8s-master ssl]# kubectl config use-context kubernetes-admin@kubernetes \
    --kubeconfig=./kubeconfig
```

### 创建 kubernetes 证书

```
[root@k8s-master ssl]# cat >/etc/kubernetes/ssl/kubernetes-csr.json  <<'HERE'
{
    "CN": "kubernetes",
    "hosts": [
    "127.0.0.1",
    "172.20.20.1",
    "172.20.20.2",
    "172.20.20.3",
    "10.6.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
     ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shenzhen",
            "L": "Shenzhen",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
HERE
```

> 注意，此处需要将dns首ip、k8s-master、node节点的ip都填上, 这里的认证IP 可以多预留几个，以备后续添加新节点能通过认证，不需要重新签发

\#生成 kubernetes 证书和私钥

```
[root@k8s-master ssl]# cfssl gencert --ca k8s-root-ca.pem --ca-key k8s-root-ca-key.pem --config k8s-gencert.json --profile kubernetes kubernetes-csr.json | cfssljson --bare kubernetes
```

\#下发证书到master,node服务器

```
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/kubernetes*.pem 172.20.20.2:/etc/kubernetes/ssl/
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/kubernetes*.pem 172.20.20.3:/etc/kubernetes/ssl/
```



