# 下载及下发二进制文件

\#下载

```
$ wget https://dl.k8s.io/v1.9.3/kubernetes-server-linux-amd64.tar.gz -P /opt
$ wget https://github.com/coreos/etcd/releases/download/v3.2.16/etcd-v3.2.16-linux-amd64.tar.gz -P /opt
$ wget https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz -P /opt
```

\#解压

```
$ cd /opt
$ tar zxvf kubernetes-server-linux-amd64.tar.gz
$ tar zxvf etcd-v3.2.16-linux-amd64.tar.gz
$ tar zxvf flannel-v0.10.0-linux-amd64.tar.gz
```

\#归类二进制文件

```
$ mkdir -p  /data/kubernetes/{node,master,etcd}

$ cp -f /opt/kubernetes/server/bin/kubelet /data/kubernetes/node/
$ cp -f /opt/mk-docker-opts.sh /data/kubernetes/node/
$ cp -f /opt/flanneld /data/kubernetes/node/

$ cp -f /opt/kubernetes/server/bin/kube-* /data/kubernetes/master/
$ cp -f /opt/kubernetes/server/bin/kubelet /data/kubernetes/master/
$ cp -f /opt/kubernetes/server/bin/kubectl /data/kubernetes/master/
$ cp -f /opt/flanneld /data/kubernetes/master/
$ cp -f /opt/mk-docker-opts.sh /data/kubernetes/master/

$ cp -f /opt/etcd-v3.2.16-linux-amd64/etcd* /data/kubernetes/etcd/
```

\#下发二进制文件

```
$ scp -r /data/kubernetes/node/* 172.20.20.2:/usr/local/bin/
$ scp -r /data/kubernetes/node/* 172.20.20.3:/usr/local/bin/

$ cp -r /data/kubernetes/master/* /usr/local/bin/

$ scp -r /data/kubernetes/etcd/* 172.20.20.4:/usr/local/bin/
$ scp -r /data/kubernetes/etcd/* 172.20.20.5:/usr/local/bin/
$ scp -r /data/kubernetes/etcd/* 172.20.20.6:/usr/local/bin/
```



