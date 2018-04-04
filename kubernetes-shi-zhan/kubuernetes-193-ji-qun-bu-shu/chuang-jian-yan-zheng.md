# 创建集群证书文件

> 这里使用 CloudFlare 的 PKI 工具集 cfssl 来生成 Certificate Authority \(CA\) 证书和秘钥文件。

\#安装证书工具CFSSL

```
[root@k8s-master ~]# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -P /opt
[root@k8s-master ~]# chmod +x /opt/cfssl_linux-amd64
[root@k8s-master ~]# mv /opt/cfssl_linux-amd64 /usr/local/bin/cfssl

[root@k8s-master ~]# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -P /opt
[root@k8s-master ~]# chmod +x /opt/cfssljson_linux-amd64
[root@k8s-master ~]# mv /opt/cfssljson_linux-amd64 /usr/local/bin/cfssljson

[root@k8s-master ~]# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -P /opt
[root@k8s-master ~]# chmod +x /opt/cfssl-certinfo_linux-amd64
[root@k8s-master ~]# mv /opt/cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

[root@k8s-master ~]# export PATH=/usr/local/bin:$PATH
```

\#创建存储证书文件夹

```
[root@k8s-master ~]# mkdir -p /etc/kubernetes/ssl
[root@k8s-master ~]# ssh 172.20.20.2 "mkdir -p /etc/kubernetes/ssl/"
[root@k8s-master ~]# ssh 172.20.20.3 "mkdir -p /etc/kubernetes/ssl/"
[root@k8s-master ~]# ssh 172.20.20.4 "mkdir -p /etc/kubernetes/ssl/"
[root@k8s-master ~]# ssh 172.20.20.5 "mkdir -p /etc/kubernetes/ssl/"
[root@k8s-master ~]# ssh 172.20.20.6 "mkdir -p /etc/kubernetes/ssl/"
```

\#创建 CA 证书配置

```
[root@k8s-master ~]# cat >/etc/kubernetes/ssl/k8s-gencert.json  <<'HERE'
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
HERE
```

```
[root@k8s-master ~]# cat >/etc/kubernetes/ssl/k8s-root-ca-csr.json  <<'HERE'
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 4096
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

\#生成 CA 证书和私钥

```
[root@k8s-master ~]# cd /etc/kubernetes/ssl/
[root@k8s-master ssl]# cfssl gencert --initca=true k8s-root-ca-csr.json | cfssljson --bare k8s-root-ca
```

\#下发证书

```
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/k8s-root-ca*.pem 172.20.20.2:/etc/kubernetes/ssl/
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/k8s-root-ca*.pem 172.20.20.3:/etc/kubernetes/ssl/
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/k8s-root-ca*.pem 172.20.20.4:/etc/kubernetes/ssl/
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/k8s-root-ca*.pem 172.20.20.5:/etc/kubernetes/ssl/
[root@k8s-master ssl]# scp -r /etc/kubernetes/ssl/k8s-root-ca*.pem 172.20.20.6:/etc/kubernetes/ssl/
```



