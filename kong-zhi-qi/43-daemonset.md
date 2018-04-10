# 什么是 DaemonSet？

DaemonSet 确保全部（或者某些）节点上运行一个 Pod 的副本。当有节点加入集群时，也会为他们新增一个 Pod 。 当有节点从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。



使用 DaemonSet 的一些典型用法：

* 运行集群存储 daemon，例如在每个节点上运行 glusterd、ceph。
* 在每个节点上运行日志收集 daemon，例如fluentd、logstash。
* 在每个节点上运行监控 daemon，例如 [Prometheus Node Exporter](https://github.com/prometheus/node_exporter)、collectd、Datadog 代理、New Relic 代理，或 Ganglia gmond。

一个简单的用法是在所有的节点上都启动一个 DaemonSet，将被作为每种类型的 daemon 使用。 一个稍微复杂的用法是单独对每种 daemon 类型使用多个 DaemonSet，但具有不同的标志，和/或对不同硬件类型具有不同的内存、CPU要求。

