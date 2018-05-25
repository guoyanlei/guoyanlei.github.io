

title: Hive跨集群访问hbase和phoenix表
date: 2018/05/22 15:30:04
tags:
- hive
- 跨集群
- hbase
- phoenix
categories:
- hive

---

在这里记录下公司前段时间对集群做的调整：

1）背景：由于之前集群是基于虚拟化的方式搭建的集群，一个物理节点上虚拟出了十多个节点，由于每个物理节点的磁盘是固定的，随着我们数据量的增长，以及数据计算频度的增加，这套集群的磁盘io已经达到瓶颈，急需搭建新的物理节点集群。

2）问题：这套老集群上部署着数据平台所用到的hbase和phoenix等，由于不能影响业务，所以短时间内不好进行迁移，但是集群的计算资源又不够用，继续把计算迁移出去。

<!--more-->

3）方案：一方面对新老集群实时日志收集进行双写，另一方面把hive表中的统计数据跨集群迁移，然后在新集群上进行统计操作，把统计后的数据跨集群写入老集群的hbase和phoenix中。

本文记录下跨集群写入Hbase和Phoenix的方法。

### 跨集群写入 hbase 表

在新建Hbas外部表时，指定老集群hbase所在的zk

```
-- hbase外部表（hive执行）
set hbase.zookeeper.quorum=10.10.95.131:2181,10.10.34.92:2181,10.10.48.188:2181;
create EXTERNAL table hbase_ads_dmp_uid_module_pv_1d (
    key         String
   ,stat_date   string
   ,pv          bigint
   ,clk         bigint
   ,page_view   bigint
)
stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
with serdeproperties (
"hbase.table.name"="hbase_ads_dmp_uid_module_pv_1d",
"hbase.columns.mapping" = ":key,T:stat_date,T:pv#b,T:clk#b,T:page_view#b"
);

```

在执行hive写入Hbase SQL时需要加上老hbase的zk

```
set hbase.zookeeper.quorum=10.10.95.131:2181,10.10.34.92:2181,10.10.48.188:2181;
insert overwrite table prod.ads_hbase_dmp_ad_mapper_pv_1d
select pv
      ,uv
  from prod.ads_hive_dmp_ad_mapper_pv_1d
  distribute by rand()
;
```

### phoenix 表

可在建外部表时直接指定phoenix所在的zk

```
-- phoenix外部表（hive执行）
add jar hdfs://nameservice:8020/user/udf/phoenix-hive-4.2.2-jar-with-dependencies.jar;
CREATE EXTERNAL TABLE PHOENIX_ADS_DMP_USER_START_1D(
     stat_date      string
    ,app_key        string
    ,start_type     string
    ,rn             int
    ,start_num_sum  bigint
    ,uv             bigint
)
STORED BY  "org.apache.phoenix.hive.PhoenixStorageHandler"
TBLPROPERTIES(
'phoenix.zookeeper.quorum'='10.10.95.131,10.10.34.92,10.10.48.188:2181',
'phoenix.hbase.table.name'='PHOENIX_ADS_DMP_USER_START_1D',
'phoenix.zookeeper.znode.parent'='hbase',
'phoenix.rowkeys'='stat_date,app_key,start_type,rn',
'phoenix.column.mapping'='start_num_sum:A.start_num_sum,uv:A.uv'
);

```

还需要再hivesever2所在的机器添加hbase配置

```
/etc/hbase/conf/hbase-site.xml

<property>
    <name>hbase.zookeeper.quorum</name>
    <value>10.10.95.131:2181,10.10.34.92:2181,10.10.48.188:2181</value>
</property>
<property>
    <name>hbase.client.scanner.caching</name>
    <value>10000</value>
</property>
<property>
    <name>hbase.rpc.timeout</name>
    <value>150000</value>
</property>

```

