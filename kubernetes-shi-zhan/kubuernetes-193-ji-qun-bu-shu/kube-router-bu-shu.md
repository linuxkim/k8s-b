# Kube-router 是什么？

**Kube-router**是基于Kubernetes网络设计的一个集负载均衡器、防火墙和容器网络的综合方案。

### 主要功能

##### 1. 基于IPVS/LVS的负载均衡器

> --run-service-proxy

kube-router采用Linux内核的IPVS模块为K8s提供Service的代理。

更多参考：

[Kubernetes network services prox with IPVS/LVS](https://cloudnativelabs.github.io/post/2017-05-10-kube-network-service-proxy/)

[Kernel Load-Balancing for Docker Containers Using IPVS](https://blog.codeship.com/kernel-load-balancing-for-docker-containers-using-ipvs/)

### 创建 yaml文件

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-router-cfg
  namespace: kube-system
  labels:
    tier: node
    k8s-app: kube-router
data:
  cni-conf.json: |
    {
      "name":"kubernetes",
      "type":"bridge",
      "bridge":"kube-bridge",
      "isDefaultGateway":true,
      "ipam": {
        "type":"host-local"
      }
    }
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    k8s-app: kube-router
    tier: node
  name: kube-router
  namespace: kube-system
spec:
  template:
    metadata:
      labels:
        k8s-app: kube-router
        tier: node
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      serviceAccountName: kube-router
      serviceAccount: kube-router
      containers:
      - name: kube-router
        image: 172.20.88.6/test/kube-router:latest
        imagePullPolicy: Always
        args:
        - --run-router=true
        - --run-firewall=true
        - --run-service-proxy=true
        - --kubeconfig=/var/lib/kube-router/kubeconfig
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          requests:
            cpu: 250m
            memory: 250Mi
        securityContext:
          privileged: true
        volumeMounts:
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true
        - name: cni-conf-dir
          mountPath: /etc/cni/net.d
        - name: kubeconfig
          mountPath: /var/lib/kube-router/kubeconfig
        - name: run
          mountPath: /var/run/docker.sock
          readOnly: true
      initContainers:
      - name: install-cni
        image: 172.20.88.6/test/busybox:latest
        imagePullPolicy: Always
        command:
        - /bin/sh
        - -c
        - set -e -x;
          if [ ! -f /etc/cni/net.d/10-kuberouter.conf ]; then
            TMP=/etc/cni/net.d/.tmp-kuberouter-cfg;
            cp /etc/kube-router/cni-conf.json ${TMP};
            mv ${TMP} /etc/cni/net.d/10-kuberouter.conf;
          fi
        volumeMounts:
        - name: cni-conf-dir
          mountPath: /etc/cni/net.d
        - name: kube-router-cfg
          mountPath: /etc/kube-router
      hostNetwork: true
      hostIPC: true
      hostPID: true
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Exists
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: cni-conf-dir
        hostPath:
          path: /etc/cni/net.d
      - name: run
        hostPath:
          path: /var/run/docker.sock
      - name: kube-router-cfg
        configMap:
          name: kube-router-cfg
      - name: kubeconfig
        hostPath:
          path: /etc/kubernetes/ssl/kubeconfig
       # configMap:
        #  name: kube-proxy
         # items:
         # - key: kubeconfig.conf
         #   path: kubeconfig
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-router
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: kube-router
  namespace: kube-system
rules:
  - apiGroups:
    - ""
    resources:
      - namespaces
      - pods
      - services
      - nodes
      - endpoints
    verbs:
      - list
      - get
      - watch
  - apiGroups:
    - "networking.k8s.io"
    resources:
      - networkpolicies
    verbs:
      - list
      - get
      - watch
  - apiGroups:
    - extensions
    resources:
      - networkpolicies
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: kube-router
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-router
subjects:
- kind: ServiceAccount
  name: kube-router
  namespace: kube-system
```

### 导入yaml文件

```
[root@k8s-master plugin]# kubectl create -f kube-router.yaml 
configmap "kube-router-cfg" created
daemonset "kube-router" created
serviceaccount "kube-router" created
clusterrole "kube-router" created
clusterrolebinding "kube-router" created
```

```
[root@k8s-master plugin]# kubectl get pod -n kube-system
NAME                   READY     STATUS    RESTARTS   AGE
po/kube-router-fhc75   1/1       Running   0          27s
po/kube-router-gvp2x   1/1       Running   0          27s
```

### 验证

```
[root@k8s-node-1 ~]# ping 10.6.0.1
PING 10.6.0.1 (10.6.0.1) 56(84) bytes of data.
64 bytes from 10.6.0.1: icmp_seq=1 ttl=64 time=0.106 ms
64 bytes from 10.6.0.1: icmp_seq=2 ttl=64 time=0.086 ms
64 bytes from 10.6.0.1: icmp_seq=3 ttl=64 time=0.076 ms
```



