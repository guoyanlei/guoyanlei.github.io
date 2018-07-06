

title: cloudera-manager中使用sqoop
date: 2018/05/21 11:30:04
tags:
- sqoop
categories:
- 数据同步

---

### 场景描述

- sqoop 与 hdfs和hive 没有安装在一起，sqoop单独一台机器。
- 使用cloudera-manager管理

### 使用问题

- 如何将sqoop使用集群的hdfs和hive

<!--more-->

### 解决方案

#### 环境变量配置

```
export HADOOP_COMMON_HOME=/data/dmp/cloudera/parcels/CDH/lib/hadoop
export HADOOP_MAPRED_HOME=/data/dmp/cloudera/parcels/CDH/lib/hadoop-mapreduce

```

#### 配置sqoop

```
/data/dmp/sqoop/conf.cloudera.sqoop_client/sqoop-env.sh
export HADOOP_COMMON_HOME=/data/dmp/cloudera/parcels/CDH/lib/hadoop
export HADOOP_MAPRED_HOME=/data/dmp/cloudera/parcels/CDH/lib/hadoop-mapreduce

```


#### 添加hdfs 和yarn的配置

```
/data/dmp/hadoop/conf.cloudera.hdfs
scp core-site.xml root@node1.azkaban.bigdata.dmp.com:/data/dmp/cloudera/parcels/CDH/etc/hadoop/conf.dist 
scp hdfs-site.xml root@node1.azkaban.bigdata.dmp.com:/data/dmp/cloudera/parcels/CDH/etc/hadoop/conf.dist 
scp hadoop-env.sh root@node1.azkaban.bigdata.dmp.com:/data/dmp/cloudera/parcels/CDH/etc/hadoop/conf.dist 
scp ssl-client.xml root@node1.azkaban.bigdata.dmp.com:/data/dmp/cloudera/parcels/CDH/etc/hadoop/conf.dist 

/data/dmp/hadoop/conf.cloudera.yarn
scp mapred-site.xml root@node1.azkaban.bigdata.dmp.com:/data/dmp/cloudera/parcels/CDH/etc/hadoop/conf.dist
scp yarn-site.xml root@node1.azkaban.bigdata.dmp.com:/data/dmp/cloudera/parcels/CDH/etc/hadoop/conf.dist
```


#### 添加hive配置

```
cd /data/dmp/hive/conf.cloudera.hive
scp hive-site.xml root@node1.azkaban.bigdata.dmp.com:/data/dmp/cloudera/parcels/CDH/etc/hive/conf.dist
scp hive-env.sh root@node1.azkaban.bigdata.dmp.com:/data/dmp/cloudera/parcels/CDH/etc/hive/conf.dist
scp log4j.properties root@node1.azkaban.bigdata.dmp.com:/data/dmp/cloudera/parcels/CDH/etc/hive/conf.dist
```

### 使用实例

```
su impala

hdfs dfs -rm -r /user/impala/tag_item_relation

sqoop import --connect jdbc:mysql://xxx:3306/ss_peacock \
--table tag_item_relation \
--username xxxx \
--password 'xxxx' \
--hive-import \
--hive-overwrite \
--hive-table prod.ods_dim_peacock_tag_item_relation \
--fields-terminated-by '\t' -m 1

```

