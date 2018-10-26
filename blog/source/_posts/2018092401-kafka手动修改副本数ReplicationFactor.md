title: kafka手动修改副本数ReplicationFactor
date: 2018/09/24 15:30:00
tags:
- kafka
categories:
- kafka

---

### kafka手动修改副本数ReplicationFactor

#### 问题描述

上周对kafka集群添加了一个broker（104），然后又删掉了，但是这个被删掉的broker一直存在于一个topic的ReplicationFactor中。

在kafka-manager中的表现是：

![kafka-rep-factor-1](/img/kafka/kafka-rep-factor-1.png)

使用如下命令查看这个topic

```
./kafka-topics.sh --zookeeper node101.zk.dmp.dmp.com:2181,node102.zk.dmp.dmp.com:2181,node103.zk.dmp.dmp.com:2181 --describe --topic pv-event
```
![kafka-rep-factor-2](/img/kafka/kafka-rep-factor-2.png)

<!--more-->

#### 尝试

在kafka-manager中（Manual Partition Assignments）进行手动修改，发现并不能实现，如图所示：

![kafka-rep-factor-3](/img/kafka/kafka-rep-factor-3.png)

因此只能选择其他方案，经搜索发现好多是增加副本数的方法，但是不确定减少副本数是不是生效。

[增加副本参考方案](https://www.iteblog.com/archives/1614.html)

将验证参考文章的方法是可以的，特此总结如下。

#### 解决方案：借助配置文件手动修改副本数

首先，新建一个json配置文件replication.json

```
{
    "version": 1, 
    "partitions": [
        {
            "topic": "pv-event", 
            "partition": 0, 
            "replicas": [
                103,102,101
            ]
        },
        {
            "topic": "pv-event", 
            "partition": 1, 
            "replicas": [
                102,103,101
            ]
        },
        {
            "topic": "pv-event", 
            "partition": 2, 
            "replicas": [
                103,102,101
            ]
        },
        ... ...
        {
            "topic": "pv-event", 
            "partition": 17, 
            "replicas": [
                103,104,102,101
            ]
        }
    ]
}
```

其次，使用kafka-reassign-partitions.sh工具来执行

```
bin/kafka-reassign-partitions.sh --zookeeper node101.zk.dmp.dmp.com:2181,node102.zk.dmp.dmp.com:2181,node103.zk.dmp.dmp.com:2181 --reassignment-json-file replication.json --execute

# 运行过程中可以通过以下命令查看
bin/kafka-reassign-partitions.sh --zookeeper node101.zk.dmp.dmp.com:2181,node102.zk.dmp.dmp.com:2181,node103.zk.dmp.dmp.com:2181 --reassignment-json-file replication.json --verify
```

运行过程：

![kafka-rep-factor-4](/img/kafka/kafka-rep-factor-4.png)

运行结果：

![kafka-rep-factor-5](/img/kafka/kafka-rep-factor-5.png)


