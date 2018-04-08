# CoreDNS 是什么？

Kubernetes包括用于服务发现的DNS服务器Kube-DNS。 该DNS服务器利用SkyDNS的库来为Kubernetes pod和服务提供DNS请求。SkyDNS2的作者，Miek Gieben，创建了一个新的DNS服务器，CoreDNS，它采用更模块化，可扩展的框架构建。 Infoblox已经与Miek合作，将此DNS服务器作为Kube-DNS的替代品。



      CoreDNS利用作为Web服务器Caddy的一部分而开发的服务器框架。该框架具有非常灵活，可扩展的模型，用于通过各种中间件组件传递请求。这些中间件组件根据请求提供不同的操作，例如记录，重定向，修改或维护。虽然它一开始作为Web服务器，但是Caddy并不是专门针对HTTP协议的，而是构建了一个基于CoreDNS的理想框架。



      在这种灵活的模型中添加对Kubernetes的支持，相当于创建了一个Kubernetes中间件。该中间件使用Kubernetes API来满足针对特定Kubernetes pod或服务的DNS请求。而且由于Kube-DNS作为Kubernetes的另一项服务，kubelet和Kube-DNS之间没有紧密的绑定。您只需要将DNS服务的IP地址和域名传递给kubelet，而Kubernetes并不关心谁在实际处理该IP请求。

# CoreDNS 部署

#### 创建 yaml文件

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

#### 导入 yaml文件

```
[root@k8s-master plugin]# kubectl create -f coredns.yaml 
serviceaccount "coredns" created
clusterrole "system:coredns" created
clusterrolebinding "system:coredns" created
configmap "coredns" created
deployment "coredns" created
service "coredns" created
```

#### 查看 kubedns 服务

```
[root@k8s-master plugin]# kubectl get pod,svc -n kube-system
NAME                          READY     STATUS    RESTARTS   AGE
po/coredns-859dd66bb5-2nzgd   1/1       Running   0          6m
po/kube-router-fhc75          1/1       Running   0          1h
po/kube-router-gvp2x          1/1       Running   0          1h

NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
svc/coredns   ClusterIP   10.6.0.2     <none>        53/UDP,53/TCP,9153/TCP   6m
```

#### 验证dns服务

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
[root@k8s-master plugin]# kubectl get pod,svc --all-namespaces
NAMESPACE     NAME                          READY     STATUS    RESTARTS   AGE
default       po/alpine                     1/1       Running   0          6s
default       po/nginx                      1/1       Running   0          37s
kube-system   po/coredns-859dd66bb5-2nzgd   1/1       Running   0          5m
kube-system   po/kube-router-fhc75          1/1       Running   0          1h
kube-system   po/kube-router-gvp2x          1/1       Running   0          1h

NAMESPACE     NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
default       svc/kubernetes      ClusterIP   10.6.0.1      <none>        443/TCP                  4d
default       svc/nginx-service   ClusterIP   10.6.163.29   <none>        80/TCP                   37s
kube-system   svc/coredns         ClusterIP   10.6.0.2      <none>        53/UDP,53/TCP,9153/TCP   5m
```

\#测试dns

```
[root@k8s-master plugin]# kubectl exec -it alpine nslookup nginx-service
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-service
Address 1: 10.6.163.29 nginx-service.default.svc.cluster.local

[root@k8s-master plugin]# kubectl exec -it alpine nslookup kubernetes
nslookup: can't resolve '(null)': Name does not resolve

Name:      kubernetes
Address 1: 10.6.0.1 kubernetes.default.svc.cluster.local
```

> 在验证 dns 之前，coredns 未部署之前创建的 pod 与 deployment 等，都必须删除，重新部署，否则无法解析



