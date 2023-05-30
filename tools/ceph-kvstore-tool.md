# 工具介绍
ceph-kvstore-tool是一个命令行工具，用于管理Ceph集群中的KV（key-value）存储。KV存储是一种轻量级的分布式数据存储机制，适用于存储小型的键值对和元数据信息。

在Ceph集群中，KV存储通常用于存储集群状态、元数据、配置信息等。ceph-kvstore-tool命令行工具提供了一系列操作KV存储的命令，包括查看、添加、删除、修改等操作。

# 注意
1. 当mon正在运行中，有锁保护，不能直接用ceph-kvstore-tool，因此需要将mon进程停掉，或者可以把数据库拷贝到另一个目录


```
cp -r /var/lib/ceph/mon/ceph-b/store.db /tmp/

ceph-kvstore-tool rocksdb /tmp/store.db list

ceph-kvstore-tool rocksdb /tmp/store.db get mon_config_key  mgr/dashboard/accessdb_v2 out file
```
