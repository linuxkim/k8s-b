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

\#grafana

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
      containers:
      - name: grafana
        image: 172.20.88.6/xnol-k8s/heapster-grafana-amd64:v4.4.3
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certificates
          readOnly: true
        - mountPath: /var
          name: grafana-storage
        env:
        - name: INFLUXDB_HOST
          value: monitoring-influxdb
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
          # The following env variables are required to make Grafana accessible via
          # the kubernetes api-server proxy. On production clusters, we recommend
          # removing these env variables, setup auth for grafana, and expose the grafana
          # service using a LoadBalancer or a public IP.
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_SERVER_ROOT_URL
          # If you're only using the API Server proxy, set this value instead:
          # value: /api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
          value: /
      volumes:
      - name: ca-certificates
        hostPath:
          path: /etc/ssl/certs
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-grafana
  name: monitoring-grafana
  namespace: kube-system
spec:
  # In a production setup, we recommend accessing Grafana through an external Loadbalancer
  # or through a public IP.
  # type: LoadBalancer
  # You could also use NodePort to expose the service at a randomly-generated port
  # type: NodePort
  type: NodePort
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 3333
  selector:
    k8s-app: grafana
```

> 官方目录:[https://github.com/kubernetes/heapster/tree/master/deploy/kube-config/influxdb](https://github.com/kubernetes/heapster/tree/master/deploy/kube-config/influxdb)

# 导入yaml文件

```
[root@k8s-master plugin]# kubectl create -f influxdb.yaml 
deployment "monitoring-influxdb" created
service "monitoring-influxdb" created

[root@k8s-master plugin]# kubectl create -f heapster-rbac.yaml 
clusterrolebinding "heapster" created

[root@k8s-master plugin]# kubectl create -f heapster.yaml 
serviceaccount "heapster" created
deployment "heapster" created
service "heapster" created

[root@k8s-master plugin]# kubectl create -f grafana.yaml 
deployment "monitoring-grafana" created
service "monitoring-grafana" created
```

```
[root@k8s-master plugin]# kubectl get pod,svc -n kube-system
NAME                                       READY     STATUS    RESTARTS   AGE
po/coredns-859dd66bb5-2nzgd                1/1       Running   0          1d
po/heapster-cfff9f6c4-ngqnk                1/1       Running   0          38s
po/kube-router-fhc75                       1/1       Running   0          1d
po/kube-router-gvp2x                       1/1       Running   0          1d
po/kubernetes-dashboard-74f769cbb4-6zshp   1/1       Running   0          6h
po/monitoring-grafana-bc5d7784c-tf4vz      1/1       Running   0          33s
po/monitoring-influxdb-58b6f7974d-wplcz    1/1       Running   0          47s

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
svc/coredns                ClusterIP   10.6.0.2       <none>        53/UDP,53/TCP,9153/TCP   1d
svc/heapster               ClusterIP   10.6.149.21    <none>        80/TCP                   39s
svc/kubernetes-dashboard   NodePort    10.6.145.157   <none>        9090:8100/TCP            6h
svc/monitoring-grafana     NodePort    10.6.161.66    <none>        80:3333/TCP              33s
svc/monitoring-influxdb    ClusterIP   10.6.138.246   <none>        8086/TCP                 47s
```

# 验证

\#heapster验证

![](/assets/N7$5}0K~[65N%28223HUMS]]8.png)

> 如上图： dashboard 中已展示出 Pod、Nodes 的 CPU、内存等 metric 图形。

\#grafana验证

> 浏览器访问：[http://NodeIP:3333，任意一个NodeIP即可。](http://NodeIP:8100，任意一个NodeIP即可。)

![](/assets/[D92H]0NA%BK_{8LO7NW3YF.png)\#数据配置

![](/assets/UYATKJ~GZ%PG$W$B@G22]8Q.png)

