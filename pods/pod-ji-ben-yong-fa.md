# 创建 Pod



使用kubectl create命令创建Pod，Kubernet中大部分API对象都是通过kubectl create命令创建的。

示例：

```
$ kubectl create -f nginx.yaml
pod "nginx" created
```

Yaml文件详解：

```
apiVersion: v1 #指定api版本，目前是v1，此值必须在kubectl apiversion中。
kind: Pod #指定创建资源的角色/类型，这里类型是Pod。
metadata: #设置Pod的元数据。
  name: nginx #指定Pod的名字，在同一个namespace中必须唯一。
spec: #配置Pod的具体规格。
  containers: #设置Pod中容器的规格，数组形式，每一项定义一个容器。
  - name: nginx #指定容器的名称，在Pod的定义中唯一。
    image: 172.20.20.1/test/nginx #设置容器镜像
    ports:
    - containerPort: 80 #必选项，设置在容器内的端口。
```

# 查询Pod

最常用的查询命令就是kubectl get，可以查询一个或者多个Pod信息。

示例：

```
$　kubectl get pod nginx
NAME      READY     STATUS    RESTARTS   AGE
nginx     1/1       Running   0          25m
```

查询显示的字段含义如下：

* NAME：Pod的名称。
* READY：Pod的准备状况，右边的数字表示Pod包含的容器总数目，左边的数字表示准备就绪的容器数目。
* STATUS：Pod的状态。
* RESTARTS：Pod的重启次数。
* AGE：Pod的运行时间。



# 删除Pod

可以通过kubectl delete 命令删除Pod。

示例：

```
$ kubectl delete pod nginx
pod "nginx" deleted
```

另外，kubectl delete 命令可以批量删除全部Pod。

示例：

```
$ kubectl delete pod --all
```



# 更新Pod

Pod在创建之后如果希望更新Pod，可以在修改Pod的定义文件后执行。

示例：

```
$ kubectl replace nginx.yaml
```

但是因为Pod的很多属性是没法修改的，比如容器镜像，这时候可以通过kubectl replace命令设置--force参数，等效于重建Pod。

示例：

```
$ kubectl replace --force nginx.yaml
```



