# 初始化环境（所有节点）

### 关闭防火墙

```
$ systemctl stop firewalld
$ systemctl disable firewalld
$ systemctl status firewalld
$ systemctl stop iptables
```

### 关闭SELINUX

```
$ setenforce  0 
$ sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
$ sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
$ getenforce
```

### 安装docker 1.13.1（master，node）

> 官方1.9 中提示， 目前 k8s 支持最高 Docker versions 1.11.2, 1.12.6, 1.13.1, and 17.03.1

\#安装163yum源

```
$ wget http://mirrors.163.com/.help/CentOS7-Base-163.repo -P /etc/yum.repos.d/
$ yum clean all
$ yum makecache
```

\#移除其它版本Docker

```
$ yum remove docker -y
```

\#安装docker

```
$ yum install docker -y
```

\#更改docker 配置

```
$ cat >/lib/systemd/system/docker.service  <<'HERE'
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target
Wants=docker-storage-setup.service
Requires=docker-cleanup.timer

[Service]
Type=notify
NotifyAccess=all
KillMode=process
EnvironmentFile=-/etc/sysconfig/docker
EnvironmentFile=-/etc/sysconfig/docker-storage
EnvironmentFile=-/etc/sysconfig/docker-network
EnvironmentFile=/run/flannel/docker
Environment=GOTRACEBACK=crash
Environment=DOCKER_HTTP_HOST_COMPAT=1
Environment=PATH=/usr/libexec/docker:/usr/bin:/usr/sbin
ExecStart=/usr/bin/dockerd-current  $DOCKER_NETWORK_OPTIONS \
          --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
          --default-runtime=docker-runc \
          --exec-opt native.cgroupdriver=systemd \
          --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
          --graph=/mnt --storage-driver=overlay2 \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0
Restart=on-abnormal
MountFlags=slave

[Install]
WantedBy=multi-user.target
HERE
```

> --graph=/mnt --storage-driver=overlay2  \#修改文件系统为ovelay2驱动

### 安装ipvsadm（node）

```
$ yum install ipvsadm -y
```



