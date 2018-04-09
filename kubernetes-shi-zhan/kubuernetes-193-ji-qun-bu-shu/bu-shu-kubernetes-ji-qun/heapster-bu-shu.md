# 创建yaml文件

\#influxdb

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: monitoring-influxdb
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: influxdb
    spec:
      containers:
      - name: influxdb
        image: 172.20.88.6/xnol-k8s/heapster_influxdb:v1.3.3
        volumeMounts:
        - mountPath: /data
          name: influxdb-storage
      volumes:
      - name: influxdb-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-influxdb
  name: monitoring-influxdb
  namespace: kube-system
spec:
  selector:
    k8s-app: influxdb
  ports:
  - port: 8086
    targetPort: 8086
```

\#heapster-rbac

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:heapster
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
```

\#heapster

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: heapster
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: heapster
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: heapster
    spec:
      serviceAccountName: heapster
      containers:
      - name: heapster
        image: 172.20.88.6/xnol-k8s/heapster-amd64:v1.5.2
        imagePullPolicy: IfNotPresent
        command:
        - /heapster
        - --source=kubernetes:https://kubernetes.default
        - --sink=influxdb:http://monitoring-influxdb.kube-system.svc:8086
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: Heapster
  name: heapster
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 8082
  selector:
    k8s-app: heapster
```

# 导入yaml文件

```
[root@k8s-master plugin]# kubectl create -f influxdb.yaml 
deployment "monitoring-influxdb" created
service "monitoring-influxdb" created
[root@k8s-master influxdb]# kubectl create -f heapster-rbac.yaml 
clusterrolebinding "heapster" created
[root@k8s-master influxdb]# kubectl create -f heapster.yaml 
serviceaccount "heapster" created
deployment "heapster" created
service "heapster" created
```

```
[root@k8s-master plugin]# kubectl get pod,svc -n kube-system
NAME                                       READY     STATUS    RESTARTS   AGE
po/coredns-859dd66bb5-2nzgd                1/1       Running   0          22h
po/heapster-cfff9f6c4-dqkn4                1/1       Running   0          1m
po/kube-router-fhc75                       1/1       Running   0          1d
po/kube-router-gvp2x                       1/1       Running   0          1d
po/kubernetes-dashboard-74f769cbb4-6zshp   1/1       Running   0          4h
po/monitoring-influxdb-58b6f7974d-jtwng    1/1       Running   0          1m

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
svc/coredns                ClusterIP   10.6.0.2       <none>        53/UDP,53/TCP,9153/TCP   22h
svc/heapster               ClusterIP   10.6.123.22    <none>        80/TCP                   1m
svc/kubernetes-dashboard   NodePort    10.6.145.157   <none>        9090:8100/TCP            4h
svc/monitoring-influxdb    ClusterIP   10.6.93.47     <none>        8086/TCP                 1m
```

# 验证

![](/assets/N7$5}0K~[65N%28223HUMS]]8.png)

