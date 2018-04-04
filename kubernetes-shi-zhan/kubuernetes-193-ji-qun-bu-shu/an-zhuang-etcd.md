# Etcd集群部署

\#创建 etcd 证书配置

```
[root@k8s-master ssl]# cat >/etc/kubernetes/ssl/etcd-csr.json  <<'HERE'
{
    "CN": "etcd",
    "hosts": [
    "127.0.0.1",
    "172.20.20.1",
    "172.20.20.2",
    "172.20.20.3",
    "172.20.20.4",
    "172.20.20.5",
    "172.20.20.6"
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

> “hosts”中包含了所有需要连接etcd的节点ip，可以多预留几个，以备后续添加新的节点能直接认证，不需要重新签发证书。

\# 生成 etcd 密钥

```
[root@k8s-master ssl]# cfssl gencert --ca k8s-root-ca.pem --ca-key k8s-root-ca-key.pem --config k8s-gencert.json --profile kubernetes etcd-csr.json | cfssljson --bare etcd
```

\#下发证书到etcd各节点及包含运行着flanneld的节点。

```
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/etcd*.pem 172.20.20.4:/etc/kubernetes/ssl/
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/etcd*.pem 172.20.20.5:/etc/kubernetes/ssl/
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/etcd*.pem 172.20.20.6:/etc/kubernetes/ssl/

[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/etcd*.pem 172.20.20.2:/etc/kubernetes/ssl/
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/etcd*.pem 172.20.20.3:/etc/kubernetes/ssl/
```

\# 创建 etcd各节点 data 目录

```
[root@k8s-master ssl]# ssh 172.20.20.4 "mkdir -p /var/lib/etcd/"
[root@k8s-master ssl]# ssh 172.20.20.5 "mkdir -p /var/lib/etcd/"
[root@k8s-master ssl]# ssh 172.20.20.6 "mkdir -p /var/lib/etcd/"
```

\# 创建etcd所需service文件（看清楚主机名）

```
#k8s-etcd-1

cat >/lib/systemd/system/etcd.service  <<'HERE'
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos
[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \
--name=k8s-etcd-1 \
--cert-file=/etc/kubernetes/ssl/etcd.pem \
--key-file=/etc/kubernetes/ssl/etcd-key.pem \
--peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
--peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
--trusted-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
--peer-trusted-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
--initial-advertise-peer-urls=https://172.20.20.4:2380 \
--listen-peer-urls=https://172.20.20.4:2380 \
--listen-client-urls=https://172.20.20.4:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://172.20.20.4:2379 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster=k8s-etcd-1=https://172.20.20.4:2380,k8s-etcd-2=https://172.20.20.5:2380,k8s-etcd-3=https://172.20.20.6:2380 \
--initial-cluster-state=new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
HERE
```

```
#etcd-2

cat >/lib/systemd/system/etcd.service  <<'HERE'
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos
[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \
--name=k8s-etcd-2 \
--cert-file=/etc/kubernetes/ssl/etcd.pem \
--key-file=/etc/kubernetes/ssl/etcd-key.pem \
--peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
--peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
--trusted-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
--peer-trusted-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
--initial-advertise-peer-urls=https://172.20.20.5:2380 \
--listen-peer-urls=https://172.20.20.5:2380 \
--listen-client-urls=https://172.20.20.5:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://172.20.20.5:2379 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster=k8s-etcd-1=https://172.20.20.4:2380,k8s-etcd-2=https://172.20.20.5:2380,k8s-etcd-3=https://172.20.20.6:2380 \
--initial-cluster-state=new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
HERE
```

```
#etcd-3

cat >/lib/systemd/system/etcd.service  <<'HERE'
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos
[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \
--name=k8s-etcd-3 \
--cert-file=/etc/kubernetes/ssl/etcd.pem \
--key-file=/etc/kubernetes/ssl/etcd-key.pem \
--peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
--peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
--trusted-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
--peer-trusted-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
--initial-advertise-peer-urls=https://172.20.20.6:2380 \
--listen-peer-urls=https://172.20.20.6:2380 \
--listen-client-urls=https://172.20.20.6:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://172.20.20.6:2379 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster=k8s-etcd-1=https://172.20.20.4:2380,k8s-etcd-2=https://172.20.20.5:2380,k8s-etcd-3=https://172.20.20.6:2380 \
--initial-cluster-state=new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
HERE
```

\#启动etcd集群

```
$ systemctl daemon-reload && systemctl start etcd && systemctl enable etcd
```

\#检查集群健康

```
etcdctl --endpoints=https://172.20.20.4:2379,https://172.20.20.5:2379,https://172.20.20.6:2379 \
  --ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  cluster-health
```

\#查看 etcd 集群成员

```
etcdctl --endpoints=https://172.20.20.4:2379,https://172.20.20.5:2379,https://172.20.20.6:2379 \
  --ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  member list
```

\#配置 flanneld 网络

```
etcdctl --endpoints=https://172.20.20.4:2379,https://172.20.20.5:2379,https://172.20.20.6:2379 \
  --ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  mkdir /kubernetes/network
```

```
etcdctl --endpoints=https://172.20.20.4:2379,https://172.20.20.5:2379,https://172.20.20.6:2379 \
  --ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  mk /kubernetes/network/config '{ "Network": "10.8.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}'
```

\#查看网络设置

```
etcdctl --endpoints=https://172.20.20.4:2379,https://172.20.20.5:2379,https://172.20.20.6:2379 \
  --ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  get /kubernetes/network/config
```



