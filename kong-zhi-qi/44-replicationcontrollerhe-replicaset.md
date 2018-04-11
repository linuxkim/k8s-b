# ReplicationController 是什么？ {#replicationcontroller和replicaset}

Replication Controller简称RC，它能确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的Pod来替代，在此基础上并提供一些高级特性，比如滚动升级和弹性伸缩。

