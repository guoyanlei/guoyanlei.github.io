title: CDH-HDFS误删HA的namenode后无法启动
date: 2018/08/10 15:30:00
tags:
- cloudera-manager
- hdfs
categories:
- cloudera-manager

---


### 背景

今天干了一件现在想还有点害怕的是，记录下。

新加了几天集群，想迁移一下Journal node，然后就顺手点了cloudera-manager上的角色迁移，在迁的过程中报错了，一系列操作后，不小心禁用了HA，禁用过程失败了，hdfs就无法启动了，报的错误如下图所示。随后又删除了3个journal node，添加了一个secondary namenode，hdfs的实例页面还是报同样的错误，重启hdfs也报这个错。突然不知道咋办有点慌了，唉。

<!--more-->

![hdfs-ha-error](/img/cm/hdfs-ha-error.png)

冷静了一段时间后，从网上找到了解决方案，还是暗自庆幸的，但是下次不该这么鲁莽了。


### 解决方法

1）删除了如下图所示的hdfs的配置文件。
因为把namenode 的ha删除之后，hdfs已经不再具有ha功能了，原来的ha设置就没有用了，但是配置上如果还存在，就会报错。

![hdfs-ha-error-conf](/img/cm/hdfs-ha-error-conf.png)

2）删除journal node，添加secondary namenode
3）先重启name node，再重启data node，到此，问题已经得到解决。
4）最后启动secondaryNamenode






