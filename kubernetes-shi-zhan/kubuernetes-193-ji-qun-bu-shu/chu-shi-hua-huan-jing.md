# 初始化环境（所有节点）

\#关闭防火墙

```
$ systemctl stop firewalld
$ systemctl disable firewalld
$ systemctl status firewalld
$ systemctl stop iptables
```

\#关闭SELINUX

```
$ setenforce  0 
$ sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
$ sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
$ getenforce
```



