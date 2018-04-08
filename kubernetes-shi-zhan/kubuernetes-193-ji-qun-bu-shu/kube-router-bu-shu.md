# Kube-router 是什么？

**Kube-router**是基于Kubernetes网络设计的一个集负载均衡器、防火墙和容器网络的综合方案。

### 主要功能

##### 1. 基于IPVS/LVS的负载均衡器

> --run-service-proxy

kube-router采用Linux内核的IPVS模块为K8s提供Service的代理。

更多参考：

* [Kubernetes network services prox with IPVS/LVS](https://cloudnativelabs.github.io/post/2017-05-10-kube-network-service-proxy/)
* [Kernel Load-Balancing for Docker Containers Using IPVS](https://blog.codeship.com/kernel-load-balancing-for-docker-containers-using-ipvs/)

##### 2. 容器网络

kube-router利用BGP协议和Go的GoBGP库和为容器网络提供直连的方案。因为用了原生的Kubernetes API去构建容器网络，意味着在使用kube-router时，不需要在你的集群里面引入其他依赖。

同样的，kube-router在引入容器CNI时也没有其它的依赖，官方的“bridge”插件就能满足kube-rouetr的需求。

更多关于BGP协议在Kubernetes中的使用可以参考：

* [Kubernetes pod networking and beyond with BGP](https://cloudnativelabs.github.io/post/2017-05-22-kube-pod-networking/)

##### 3. 网络策略管理

> --run-firewall

采用了kube-router的Kubernetes很容易通过添加标签到kube-router的方式使用[网路策略](https://link.jianshu.com?t=https://kubernetes.io/docs/concepts/services-networking/network-policies/)功能。kube-router使用了ipset操作iptables，以保证防火墙的规则对系统性能有较低的影响。

Kube-router支持networking.k8s.io/NetworkPolicy的API或者其他基于网络策略的V1/GA语义。

更多关于kube-router防火墙的功能可以参考：

* [Enforcing Kubernetes network policies with iptables](https://cloudnativelabs.github.io/post/2017-05-1-kube-network-policies/)

#### 负载均衡器

kube-router的负载均衡器功能，会在物理机上创建一个虚拟的kube-dummy-if网卡，然后利用k8s的watch APi实时更新svc和ep的信息。svc的cluster\_ip会绑定在kube-dummy-if网卡上，作为lvs的virtual server的地址。realserver的ip则通过ep获取到容器的IP地址。

# 部署 Kube-router

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

> --run-router=true 
> \#启用Pod网络 - 通过iBGP发布并学习到Pod的路由。 （默认为true）
>
> --run-firewall=true
> \#启用网络策略 - 设置iptables为pod提供入口防火墙。 （默认为true）
>
> --run-service-proxy=true 
> \#启用服务代理 - 为Kubernetes服务设置IPVS。 （默认为true）

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



