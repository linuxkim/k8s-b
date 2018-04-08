### 创建 yaml文件

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        log stdout
        health
        kubernetes cluster.local 10.6.0.0/16
        prometheus
        proxy . /etc/resolv.conf
        cache 30
    }
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      serviceAccountName: coredns
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
      - name: coredns
        image: 172.20.88.6/test/coredns:0.9.10
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: 10.6.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
```

> kubernetes cluster.local \#创建 svc 的 IP 段
>
> clusterIP  \#指定 DNS 的 IP

### 导入 yaml文件

```
[root@k8s-master plugin]# kubectl create -f coredns.yaml 
serviceaccount "coredns" created
clusterrole "system:coredns" created
clusterrolebinding "system:coredns" created
configmap "coredns" created
deployment "coredns" created
service "coredns" created
```

### 查看 kubedns 服务

```
[root@k8s-master plugin]# kubectl get pod,svc -n kube-system
NAME                          READY     STATUS              RESTARTS   AGE
po/coredns-859dd66bb5-b6cx2   0/1       ContainerCreating   0          11m

NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
svc/coredns   ClusterIP   10.6.0.2     <none>        53/UDP,53/TCP,9153/TCP   11m
```

### 验证dns服务

\#创建一个service

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
     app: nginx    
spec:
     containers:
        - name: nginx
          image: 172.20.88.6/test/nginx
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 80
     restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  sessionAffinity: ClientIP
  selector:
    app: nginx
  ports:
    - port: 80
      nodePort: 3080
```

```
[root@k8s-master plugin]# kubectl create -f nginx.yaml 
pod "nginx" created
service "nginx-service" created
```

```
[root@k8s-master plugin]# kubectl get pod,svc
NAME        READY     STATUS    RESTARTS   AGE
po/nginx    1/1       Running   0          55s

NAME                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)       AGE
svc/kubernetes      ClusterIP   10.6.0.1     <none>        443/TCP       3d
svc/nginx-service   NodePort    10.6.64.42   <none>        80:3080/TCP   54s
```

\#创建一个pod

```
apiVersion: v1
kind: Pod
metadata:
  name: alpine
spec:
  containers:
  - name: alpine
    image: 172.20.88.6/xnol-k8s/alpine
    command:
    - sh
    - -c
    - while true; do sleep 1; done
```

```
[root@k8s-master plugin]# kubectl create -f alpine.yaml 
pod "alpine" created
```

\#查看所有service及pod

```
[root@k8s-master plugin]# kubectl get pod,svc 
NAME        READY     STATUS              RESTARTS   AGE
po/alpine   0/1       ContainerCreating   0          23s
po/nginx    0/1       ContainerCreating   0          4m

NAME                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)       AGE
svc/kubernetes      ClusterIP   10.6.0.1     <none>        443/TCP       3d
svc/nginx-service   NodePort    10.6.3.133   <none>        80:3080/TCP   4m
```

\#测试dns

