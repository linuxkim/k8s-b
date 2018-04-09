# 创建yaml文件

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      serviceAccountName: kubernetes-dashboard
      containers:
      - name: kubernetes-dashboard
        image: 172.20.88.6/test/kubernetes-dashboard-amd64:head
        resources:
          # keep request = limit to keep this container in guaranteed class
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 9090
        livenessProbe:
          httpGet:
            path: /
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
---
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 8100
```

> 官方目录：[https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dashboard](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dashboard)

# 导入yaml文件

```
[root@k8s-master plugin]# kubectl create -f dashboard.yaml 
serviceaccount "kubernetes-dashboard" created
clusterrolebinding "kubernetes-dashboard" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

```
[root@k8s-master plugin]# kubectl get pod,svc -n kube-system
NAME                                       READY     STATUS    RESTARTS   AGE
po/coredns-859dd66bb5-2nzgd                1/1       Running   0          18h
po/kube-router-fhc75                       1/1       Running   0          19h
po/kube-router-gvp2x                       1/1       Running   0          19h
po/kubernetes-dashboard-74f769cbb4-6zshp   1/1       Running   0          12s

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
svc/coredns                ClusterIP   10.6.0.2       <none>        53/UDP,53/TCP,9153/TCP   18h
svc/kubernetes-dashboard   NodePort    10.6.145.157   <none>        9090:8100/TCP            13s
```

# 测试

> 浏览器访问：[http://NodeIP:8100，任意一个NodeIP即可。](http://NodeIP:8100，任意一个NodeIP即可。)

![](/assets/123123121231.jpg)

> 由于缺少 Heapster 插件，当前 dashboard 不能展示 Pod、Nodes 的 CPU、内存等 metric 图形。



