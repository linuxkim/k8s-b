# Job

Job负责批量处理短暂的一次性任务 \(short lived one-off tasks\)，即仅执行一次的任务，它保证批处理任务的一个或多个Pod成功结束。

Kubernetes支持以下几种Job：

* 非并行Job：通常创建一个Pod直至其成功结束
* 固定结束次数的Job：设置.spec.completions，创建多个Pod，直到.spec.completions个Pod成功结束
* 带有工作队列的并行Job：设置.spec.Parallelism但不设置.spec.completions，当所有Pod结束并且至少一个成功时，Job就认为是成功

根据.spec.completions和.spec.Parallelism的设置，可以将Job划分为以下几种pattern：

| Job类型 | 使用示例 | 行为 | Completions | Parallelism |
| :--- | :--- | :--- | :--- | :--- |
| 一次性Job | 数据库迁移 | 创建一个Pod运行至其成功结束 | 1 | 1 |
| 固定结束次数的Job | 处理工作队列的Pod | 依次创建一个Pod运行至completions成功结束 | 2+ | 1 |
| 固定结束次数的并行Job | 多个Pod同时处理工作队列 | 依次创建一个Pod运行至completions成功结束 | 2+ | 2+ |
| 并行Job | 多个Pod同事处理工作队列 | 创建一个或多个Pod至有一个成功结束 | 1 | 2+ |

# Job Controller

Job Controller负责根据Job Spec创建Pod，并持续监控Pod的状态，直至其成功结束。如果失败，则根据restartPolicy（只支持OnFailure和Never，不支持Always）决定是否创建新的Pod再次重试任务。

# Job Spec格式

* spec.template格式同Pod
* RestartPolicy仅支持Never或OnFailure
* 单个Pod时，默认Pod成功运行后Job即结束
* .spec.completions标志Job结束需要成功运行的Pod个数，默认为1
* .spec.parallelism标志并行运行的Pod的个数，默认为1
* spec.activeDeadlineSeconds标志失败Pod的重试最大时间，超过这个时间不会继续重试

#### 示例

\#job.yaml

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    metadata:
      name: pi
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

因为是一次性任务Pod，所以Pod的重启策略只能是Never或者OnFailure。

\#创建Job

```
[root@k8s-master plugin]# kubectl create -f job.yaml
job "pi" created

[root@k8s-master plugin]# kubectl get job pi
NAME      DESIRED   SUCCESSFUL   AGE
pi        1         0            3s
```

\#Job创建成功后，会运行Pod

```
[root@k8s-master plugin]# kubectl get pod --selector app=pi
NAME       READY     STATUS    RESTARTS   AGE
pi-scfgt   1/1       Running   0          11s
```

\#Pod运行的是一次性任务，计算出圆周率就终止了

```
[root@k8s-master plugin]# kubectl logs pi-scfgt
3.1415926535897932384626433832795028841971693993751058209749445923078164062862089.....
```

\#再一次性任务Pod执行完后，显示Job已经成功执行1次

```
[root@k8s-master plugin]# kubectl get job pi
NAME      DESIRED   SUCCESSFUL   AGE
pi        1         1            3m
```



