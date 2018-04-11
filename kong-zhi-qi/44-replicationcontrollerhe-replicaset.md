# ReplicationController 是什么？ {#replicationcontroller和replicaset}

Replication Controller简称RC，它能确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的Pod来替代，在此基础上并提供一些高级特性，比如滚动升级和弹性伸缩。

# 主要功能

* **确保pod数量**：它会确保Kubernetes中有指定数量的Pod在运行。如果少于指定数量的pod，Replication Controller会创建新的，反之则会删除掉多余的以保证Pod数量不变。
* **确保pod健康**：当pod不健康，运行出错或者无法提供服务时，Replication Controller也会杀死不健康的pod，重新创建新的。
* **弹性伸缩 **：在业务高峰或者低峰期的时候，可以通过Replication Controller动态的调整pod的数量来提高资源的利用率。同时，配置相应的监控功能（Hroizontal Pod Autoscaler），会定时自动从监控平台获取Replication Controller关联pod的整体资源使用情况，做到自动伸缩。
* **滚动升级**：滚动升级为一种平滑的升级方式，通过逐步替换的策略，保证整体系统的稳定，在初始化升级的时候就可以及时发现和解决问题，避免问题不断扩大。

# 创建 ReplicationController

\#nginx-rc.yaml

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 2
  selector:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: 172.20.88.6/test/nginx:v1
        ports:
        - containerPort: 80
```

\#创建rc

```
[root@k8s-master plugin]# kubectl create -f nginx-rc.yaml 
replicationcontroller "nginx-rc" created
```

\#查看rc和pod

```
[root@k8s-master plugin]# kubectl get rc nginx-rc
NAME       DESIRED   CURRENT   READY     AGE
nginx-rc   2         2         2         48s

[root@k8s-master plugin]# kubectl get pod --selector app=nginx
NAME             READY     STATUS    RESTARTS   AGE
nginx-rc-qp4dc   1/1       Running   0          1m
nginx-rc-v4k8r   1/1       Running   0          1m
```



