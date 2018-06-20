title: hive调优之控制mapper和reducer数
date: 2018/06/19 17:30:04
tags:
- hive
categories:
- hive

---

### hive调优之控制mapper和reducer数

由于mapreduce中没有办法直接控制map数量，所以只能曲线救国，通过设置每个map中处理的数据量进行设置；reduce是可以直接设置的。

<!--more-->

### 控制mapper数

```
set mapred.max.split.size=256000000;        -- 决定每个map处理的最大的文件大小，单位为B
set mapred.min.split.size.per.node=1;         -- 节点中可以处理的最小的文件大小
set mapred.min.split.size.per.rack=1;         -- 机架中可以处理的最小的文件大小

```

#### mapper划分文件处理逻辑

- 执行HQL时，首先会读取表分区下到文件，然后分发到各个节点，此时是根据mapred.max.split.size确认要启动多少个map数。

- 假设有两个文件大小分别为(256M,280M)被分配到节点A，那么会启动两个map，剩余的文件大小为10MB和35MB因为每个大小都不足241MB会先做保留

- 根据参数set mapred.min.split.size.per.node看剩余的大小情况并进行合并。

- 如果值为1，表示a中每个剩余文件都会自己起一个map，这里会起两个，如果设置为大于45 * 1024 * 1024则会合并成一个块，并产生一个map。如果mapred.min.split.size.per.node为10 * 1024 * 1024，那么在这个节点上一共会有4个map，处理的大小为(245MB，245MB，10MB，10MB，10MB，10MB)，余下9MB。如果mapred.min.split.size.per.node为45 * 1024 * 1024，那么会有三个map，处理的大小为(245MB，245MB，45MB) 

- 实际中mapred.min.split.size.per.node无法准确地设置成45 * 1024 * 1024，会有剩余并保留带下一步进行判断处理 

- 对b中余出来的文件与其它节点余出来的文件根据mapred.min.split.size.per.rack大小进行判断是否合并，对再次余出来的文件独自产生一个map处理


#### 场景1：控制文件生成个数

通常在执行HQL时，分配的mapper和reducer个数都是根据文件的大小和任务的情况自动计算的，但有时，我们想去控制最终生成文件的个数。

前段时间在做hive表的跨集群迁移，需要将老集群上的ODS层表迁移到新集群，由于ODS表中的分区时按天和小时的，分区下小文件较多，在进行hive export和discp操作时，由于要频繁的创建和到导出小文件，因此非常慢。

能想到的解决方法：合并分区数据，减少小文件，然后再迁移

由于合并分区通常只有map阶段，没有reduce，因此有多少个mapper就会生成多少个文件，这里就可以通过设置mapper数间接控制文件个数，即加大map分配到文件大小，减少mapper个数。

```
set mapred.max.split.size=1024000000;
set mapred.min.split.size.per.node=1024000000;
set mapred.min.split.size.per.rack=1024000000;
insert into table xxx
select *
  from xxx
 where xxx
;
```

### 控制reducer数

修改reduce的个数就简单很多，直接根据可能的情况作个简单的判断确认需要的reduce数量，如果无法判断，根据当前map数减小10倍，保持在1~100个reduce即可。

```
方法1
set mapred.reduce.tasks=10;  -- 设置reduce的数量
方法2
set hive.exec.reducers.bytes.per.reducer=1073741824 -- 每个reduce处理的数据量,默认1GB
```

可通过以下两个参数配合使用：

- set hive.merge.mapredfiles=true
- distribute by rand()


小文件太多的话可以打开reduce端的小文件合并，即第一个set，后面的distribute by rand()保证了记录随机分配到50个文件，不管里数据量有多小，最后这50个文件的大小应该是一致的.



