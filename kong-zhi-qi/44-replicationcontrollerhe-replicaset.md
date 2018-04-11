# ReplicationController 是什么？ {#replicationcontroller和replicaset}

Replication Controller简称RC，它能确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的Pod来替代，在此基础上并提供一些高级特性，比如滚动升级和弹性伸缩。

# 主要功能

* **确保pod数量**：它会确保Kubernetes中有指定数量的Pod在运行。如果少于指定数量的pod，Replication Controller会创建新的，反之则会删除掉多余的以保证Pod数量不变。
* **确保pod健康**：当pod不健康，运行出错或者无法提供服务时，Replication Controller也会杀死不健康的pod，重新创建新的。
* **弹性伸缩 **：在业务高峰或者低峰期的时候，可以通过Replication Controller动态的调整pod的数量来提高资源的利用率。同时，配置相应的监控功能（Hroizontal Pod Autoscaler），会定时自动从监控平台获取Replication Controller关联pod的整体资源使用情况，做到自动伸缩。
* **滚动升级**：滚动升级为一种平滑的升级方式，通过逐步替换的策略，保证整体系统的稳定，在初始化升级的时候就可以及时发现和解决问题，避免问题不断扩大。

# 创建 ReplicationController

\#nginx-rc-v1.yaml

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc-v1
spec:
  replicas: 2
  selector:
    app: nginx
    version: v1
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: 172.20.88.6/test/nginx:v1
        ports:
        - containerPort: 80
```

\#创建rc

```
[root@k8s-master plugin]# kubectl create -f nginx-rc-v1.yaml 
replicationcontroller "nginx-rc-v1" created
```

\#查看rc和pod

```
[root@k8s-master plugin]# kubectl get rc nginx-rc-v1
NAME          DESIRED   CURRENT   READY     AGE
nginx-rc-v1   2         2         2         21s

[root@k8s-master plugin]# kubectl get pod --selector app=nginx
NAME                READY     STATUS    RESTARTS   AGE
nginx-rc-v1-7bj2x   1/1       Running   0          29s
nginx-rc-v1-7tfd5   1/1       Running   0          29s

```

## 滚动升级

> 滚动升级是一种平滑过渡的升级方式，通过逐步替换的策略，保证整体系统的稳定，在初始升级的时候可以及时发现，调整问题，以保证问题影响度不会扩大。

\#将nginx v1版本升级到v2

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc-v2
spec:
  replicas: 2
  selector:
    app: nginx
    version: v2
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
        version: v2
    spec:
      containers:
      - name: nginx
        image: 172.20.88.6/test/nginx:v2
        ports:
        - containerPort: 80
```

\#开始滚动升级

```
[root@k8s-master plugin]# kubectl rolling-update nginx-rc-v1 -f nginx-rc-v2.yaml --update-period=10s
Created nginx-rc-v2
Scaling up nginx-rc-v2 from 0 to 2, scaling down nginx-rc-v1 from 2 to 0 (keep 2 pods available, don't exceed 3 pods)
Scaling nginx-rc-v2 up to 1
Scaling nginx-rc-v1 down to 1
Scaling nginx-rc-v2 up to 2
Scaling nginx-rc-v1 down to 0
Update succeeded. Deleting nginx-rc-v1
replicationcontroller "nginx-rc-v1" rolling updated to "nginx-rc-v2"
```

升级开始后，导入v2的yaml文件，每隔10s逐步增加v2版本的Replication Controller的Pod副本数，逐步减少v1版本的Replication Controller的Pod副本数。升级完成后删除v1版本的Replication Controller，保留v2版本的Replication Controller，即实现滚动升级。



