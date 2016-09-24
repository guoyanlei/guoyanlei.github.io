---
title: Internal Hive--Hive的精髓
date: 2016-09-24 18:20:51
tags:
- 大数据
- 数据仓库
- hive
categories:
- Hive
---

在公司对数据仓库Hive已经学习和使用了很长时间，觉得很有必要对Hive重要的东西做个较全面总结，帮助自己对整个知识点的梳理，也帮助其他感兴趣的亲们更快的了解和学习Hive。接下来将分为以下几点进行阐述。

+ MapReduce框架实现SQL的基本原理
+ Hive SQL的编译过程
+ Hive SQL的性能优化
+ 数据倾斜的预防和处理

## MapReduce框架实现SQL的基本原理

首先，我们先不去关心HiveSQL具体是怎么被Hive所编译的，我们只需知道一个Hive查询会生成多个MapReduce Job，一个MapReduce Job又有Map，Reduce，Shuffle，Sort等多个阶段，所以在深入理解HiveSQL编译过程之前，我们需要先了解一下MapReduce框架实现SQL基本操作的原理。

接下来以SQL中的Join、Group by和Distinct的实现原理来理解MapReduce框架：

### Join的实现原理

假如有两个表，user表和order表，它们之间进行Join操作，整个过程如下图：

    select u.name, o.orderid from order o join user u on o.uid = u.uid;

{% img /img/hive/1469784640027.jpg join %}

总结之，即
1.Map阶段将输入文件的每行按key value分割；
2.Shuffle和sort阶段会将相同的key值进行汇聚，并将value值排序；
（Shuffle的主要任务是：怎样把Map task的输出结果有效地传送到Reduce端。也可以这样理解， Shuffle描述着数据从Map task输出到Reduce task输入的这段过程，包括了Sort和Combiner操作。）
3.Reduce阶段会将相同key值的value进行处理，从而实现Join操作。

注意：这里可以思考一个问题，假如order表中uid=1的数据占了绝大多数，而其他的uid却与之相比又少很多，极端一下，uid=1的有1000多万，而其他的平均只有1千个，这样在进行Reduce时就会有个别的Reduce收到比其他Reduce多得多的数据，这样即产生了【数据倾斜】，之后会详细介绍优化方法和途径。

### Group By的实现原理

如下的Group by操作的过程如下图：

    select rank, isonline, count(*) from city group by rank, isonline;

{% img /img/hive/1469787083263.jpg GroupBy %}

分析之，各个阶段的操作和Join类似，不同的是key、value的分割问题。观察结果，可以忽略最后一列，将group by操作用来做去重处理，SQL如下。

    select rank, isonline from city group by rank, isonline;

### count(distinct id)的实现原理

考虑如下的一个SQL查询：

    select dealid, count(distinct uid) num from order group by dealid;

{% img /img/hive/1469934726261.jpg count %}

分析之，若仅仅对一个字段进行distinct处理，只需将group by 和distinct字段组合为Map输出的key，利用Shuffle阶段的排序功能汇聚到不同的Reduce，最后将group by字段作为Reduce输出的key。
若对两个或以上的字段进行distinct处理，如下SQL，则不能使用上面方法（将各个字段组合起来，借助Shuffle的排序功能）实现汇聚了。

    select dealid, count(distinct uid), count(distinct date) from order group by dealid;

在Hive中是这样实现的，如下图，对所有的distinct字段编号，每行数据生成n行数据，那么相同字段就会分别排序，这时只需要在reduce阶段记录LastKey即可去重。
这种实现方式很好的利用了MapReduce的排序，节省了reduce阶段去重的内存消耗，但是缺点是增加了shuffle的数据量。

{% img /img/hive/1469936051510.jpg reduce %}

注意：再仔细观察一下整个过程，发现最终的去重、求和处理都是在最后的Reduce阶段完成的，我们知道对于整个Job处理，Reducer的个数往往是小于Maper的，当处理的数据量非常大Reducer的个数又非常少，或者个别group by字段的数据较为突出时，就会引发两个问题：处理效率非常慢，数据倾斜。

解决办法：
+ 针对第一个问题，尽量避免使用count(distinct)来去重求和，应多使用group by 或 row number 先去重，然后再使用sum来求和，主要是为了借助group by 或 row number在Map阶段先实现去重，进而分担Reduce的压力，从而提高效率。
+ 针对数据倾斜，之后详细介绍

## Hive SQL的编译过程

了解完MapReduce框架实现SQL的基本原理之后，下面分析Hive SQL的编译过程。我们可以先从全局的角度，了解一下Hive的整体架构，别的不多说，其中的重点就是Driver模块，它由编译器（Compiler）、优化器（Optimizer）、解释器（Executor）等完成HQL查询语句从词法分析、语法分析、编译、优化以及查询计划的生成。生成的查询计划存储在HDFS中，并在随后有MapReduce调用执行。

{% img /img/hive/1469954620861.jpg scheme %}

下面重点介绍下编译器的重要过程。
+ 编译器将Hive SQL 转换成一组操作符(Operator)
+ 操作符是Hive的最小处理单元
+ 每个操作符处理代表一道HDFS操作或MapReduce作业

主要的操作符有：

|  操作符   |  说明   |
| --------- | ---------- |
| TableScanOperator    |  扫描hive表数据   |
| ReduceSinkOperator    | 创建将发送到Reducer端的<Key,Value>对    |
| JoinOperator    | Join两份数据    |
| SelectOperator    | 选择输出列    |
| FileSinkOperator    | 建立结果数据,输出至文件    |
| FilterOperator    | 过滤输入数据    |
| GroupByOperator    | GroupBy语句    |
| MapJoinOperator    | /*+mapjoin(t) */    |
| LimitOperator    | Limit语句    |
| UnionOperator    | Union语句    |

整个编译过程分为六个阶段：

1. Parser阶段，Hive会根据Antlr定义SQL的语法规则，完成SQL词法，语法解析，将SQL转化为抽象语法树AST Tree；
其中ANTLR (ANother Tool for Language Recognition) 是一个语言识别工具，它可以根据输入自动生成语法树并可视化的显示出来。这里不再详述，具体可参看文献1,2。

2. Semantic Analyzer阶段，遍历AST Tree，抽象出查询的基本组成单元QueryBlock；
由于AST Tree仍然非常复杂，不够结构化，不方便直接翻译为MapReduce程序，AST Tree转化为QueryBlock就是将SQL进一部抽象和结构化。QueryBlock是一条SQL最基本的组成单元，包括三个部分：输入源，计算过程，输出。简单来讲一个QueryBlock就是一个子查询。

3. Logic Plan Generator阶段，遍历QueryBlock，翻译为逻辑查询计划，即执行操作树OperatorTree；
Hive最终生成的MapReduce任务中的Map阶段和Reduce阶段均由OperatorTree组成。逻辑操作符，就是在Map阶段或者Reduce阶段完成单一特定的操作。主要包括TableScanOperator，SelectOperator，FilterOperator，JoinOperator，GroupByOperator，ReduceSinkOperator等，如上个表所示。

4. Logical Optimizer阶段，逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量；

5. Physical Plan Generator阶段，将逻辑查询计划转成物理查询计划，即遍历OperatorTree，翻译为MapReduce任务；

6. Physical Optimizer阶段，物理层优化器进行MapReduce任务的变换，生成最终的执行计划。

以如下的查询为例进行详细描述：

    INSERT OVERWRITE TABLE access_log_temp2
    SELECT a.user, a.prono, p.maker, p.price
      FROM access_log_hbase as a
      JOIN product_hbase as p
        ON (a.prono= p.prono);

第1步：Parser成抽象语法树AST Tree，先不看下面的图片，自己可以先想想一下整个SQL会被如何解析。首先按照常理的思路，可以把上面的查询分为如下面三个步骤：
1）从表access_log_hbase和表product_hbase去查，关联的字段是prono；
2）查哪些字段，即a.user, a.prono, p.maker, p.price；
3）插入到新的表access_log_temp2中
由此得到的真实的语法树如下图。

{% img /img/hive/1469961401357.jpg ASTTree %}

第2步：遍历AST Tree，抽象出查询的基本组成单元QueryBlock
下图给出语法树的一部分抽象成查询块的过程，其他类似。

{% img /img/hive/1469967315586.jpg QueryBlock %}

第3步：遍历QueryBlock，翻译为逻辑查询计划，即执行操作树OperatorTree（由一个和多个操作符构成）；

{% img /img/hive/1469968200881.jpg QueryBlock %}

第4步：对OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量。
经过第3步之后生成的执行操作树如图中最右所示，这颗执行操作树没什么可优化的了，但是若我们要执行的是如下的SQL，则生成的执行语法树是图中第二个，此时逻辑层优化器就会对这颗语法树进行优化，结果如最右图所示。

    INSERT OVERWRITE TABLE access_log_temp2
    SELECT a.user, a.prono, p.maker, p.price
      FROM access_log_hbase as a
      JOIN product_hbase as p
        ON (a.prono= p.prono)
     WHERE p.maker= 'honda';

{% img /img/hive/1469969995476.jpg OperatorTree %}

第5步：遍历OperatorTree，翻译为MapReduce任务，将上步生成的OperatorTree转换成的MapReduce任务如下图。

{% img /img/hive/1469970118344.jpg MapReduce %}

第6步：物理层优化器进行MapReduce任务的变换，包括：
MapJoinResolver 处理MapJoin
SkewJoinResolver  处理倾斜Join
CommonJoinResolver  处理普通Join 。等

## Hive SQL的性能优化

### Map阶段的优化

Map阶段的优化，主要是确定合适的map数。在这之前首先要了解的就是map的数量是怎么计算的。

    num_map_tasks = max[
                        ${mapred.min.split.size}, //数据的最小分割单元大小
                        min(
                            ${dfs.block.size}, //HDFS设置的数据块大小
                            ${mapred.max.split.size}  //数据的最大分割单元大小
                        )
                    ]

一般来说dfs.block.size是由HDFS决定的，Hive无权干预。
所以实际上只有`mapred.min.split.size和mapred.max.split.size`这两个参数来决定map数量。
在hive中min的默认值是1B，max的默认值是256MB。
通过公式可得，如果不修改这些默认值的话，1个map task处理的就是256MB的数据，因此，通过调整max可以起到调整map数的作用，减小max可以增加map数，增大max可以减少map数。需要提醒的是，直接调整mapred.map.tasks这个参数是没有效果的。

调整大小的时机根据查询的不同而不同，主要来讲还是看map task执行的时间来确定，若执行时间很长，那么往往增加map数量，使得每个map task处理的数据量减少，能够让map task更快完成。若map task的运行时间已经很少了，比如10-20秒，这个时候增加map不太可能让map task更快完成，反而可能因为map需要的初始化时间反而让job总体速度变慢，这个时候反而需要考虑是否可以把map的数量减少，这样可以节省更多资源给其他Job。


### Reduce阶段的优化

跟上面的map优化类似，Reduce阶段的优化也是选择合适的reduce task数量。
与map优化不同的是，reduce优化时，可以直接设置`mapred.reduce.tasks`参数从而直接指定reduce的个数。当然直接指定reduce个数虽然比较方便，但是不利于自动扩展。Reduce数的设置虽然相较map更灵活，但是也可以像map一样设定一个自动生成规则，这样运行定时job的时候就不用担心原来设置的固定reduce数会由于数据量的变化而不合适。

Hive估算reduce数量的时候，使用的是下面的公式：

    num_reduce_tasks = min[
                            ${hive.exec.reducers.max},
                            (
                                ${input.size} / ${ hive.exec.reducers.bytes.per.reducer}
                            )
                       ]

其中，`hive.exec.reducers.bytes.per.reducer`默认为1G，也就是每个reduce处理相当于job输入文件中1G大小的对应数据量，而且reduce个数不能超过一个上限参数值，这个参数的默认取值为999。所以我们也可以用调整这个公式的方式调整reduce数量，在灵活性和定制性上取得一个平衡。


### Shuffle阶段的优化

Shuffle阶段包括了：spill, copy, sort等。
其中，spill阶段，由于内存不够，数据可能没办法在内存中一次性排序完成，那么就只能把局部排序的文件先保存到磁盘上，这个动作叫spill，然后spill出来的多个文件可以在最后进行merge。如果发生spill，可以通过设置`io.sort.mb`来增大mapper输出buffer的大小，避免spill的发生。

另外合并时可以通过设置`io.sort.factor`来使得一次性能够合并更多的数据。
Reduce端的merge也是一样可以用io.sort.factor。一般情况下这两个参数很少需要调整，除非很明确知道这个地方是瓶颈。如果map端的输出太大，考虑到map数不一定能很方便的调整，那么这个时候就要考虑调大io.sort.mb（不过即使调大也要注意不能超过jvm heap size）。

copy阶段是把文件从map端copy到reduce端。默认情况下在5%的map完成的情况下reduce就开始启动copy，这个有时候是很浪费资源的，因为reduce一旦启动就被占用，一直等到map全部完成，收集到所有数据才可以进行后面的动作，所以我们可以等比较多的map完成之后再启动reduce流程，这个比例可以通`mapred.reduce.slowstart.completed.maps`去调整，他的默认值就是5%。

### 文件格式的优化

文件格式方面有两个问题，一个是给输入和输出选择合适的文件格式，另一个则是小文件问题。小文件问题在目前的hive环境下已经得到了比较好的解决，hive的默认配置中就可以在小文件输入时自动把多个文件合并给1个map处理，输出时如果文件很小也会进行一轮单独的合并。下面重点介绍文件格式。

Hive中文件格式有三种：textfile，sequencefile和rcfile。总体上来说，rcfile的压缩比例和查询时间稍好一点，所以推荐使用。
关于使用方法，可以在建表结构时可以指定格式，然后指定压缩插入：

    create table rc_file_test( col int ) stored as rcfile;
       set hive.exec.compress.output = true;
    insert overwrite table rc_file_test
    select * from source_table;

另外时也可以指定输出格式，也可以通过hive.default.fileformat来设定输出格式，适用于create table as select的情况：

    set hive.default.fileformat = SequenceFile;
    set hive.exec.compress.output = true; /*对于sequencefile，有record和block两种压缩方式可选，block压缩比更高*/
    set mapred.output.compression.type = BLOCK;
    create table seq_file_test as select * from source_table;


### SQL的整体优化

#### 让多个Job并行处理

在有些情况下Job之间是可以并行的，典型的就是子查询。当需要执行多个子查询union all或者join操作的时候，Job间并行就可以使用。如下代码：

    select * from
    (
        select count(*) from logs
         where log_date = 20130801 and item_id = 1
         union all
        select count(*) from logs
         where log_date = 20130802 and item_id = 2
         union all
        select count(*) from logs
         where log_date = 20130803 and item_id = 3
    )t


设置Job间并行的参数是`hive.exec.parallel`，将其设为true即可。默认的并行度为8，也就是最多允许sql中8个Job并行。如果想要更高的并行度，可以通过`hive.exec.parallel. thread.number`参数进行设置，但要避免设置过大而占用过多资源。

#### 减少Job数量

针对如下的查询：查询某网站日志中访问过页面a和页面b的用户数量。
一般想到的代码是这样的：

    select count(*)
      from
          (
           select distinct user_id
             from logs where page_name = ‘a’
          ) a
      join
          (
           select distinct user_id
             from logs where blog_owner = ‘b’
          ) b
        on a.user_id = b.user_id;
这样一来，就要产生2个求子查询的Job，一个用于关联的Job，还有一个计数的Job，一共有4个Job。
更好的查询应该是这样的，只需要用一个Job就能跑完：

    select count(*)
      from logs group by user_id
    having
          (
            count(case when page_name = ‘a’ then 1 end) > 0
            and count(case when page_name = ‘b’ then 1 end) > 0
          )

## 数据倾斜的预防和处理

所谓数据倾斜，说的是由于数据分布不均匀，个别值集中占据大部分数据量，加上hadoop的计算模式，导致计算资源不均匀引起性能下降。
表现症状：
任务进度长时间维持在99%（或100%），查看任务监控页面，发现只有少量（1个或几个）reduce子任务未完成。因为其处理的数据量和其他reduce差异过大。
单一reduce的记录数与平均记录数差异过大，通常可能达到3倍甚至更多。 最长时长远大于平均时长。如下图即为一个数据倾斜的例子。

{% img /img/hive/1469973820250.jpg qingxie %}

主要原因：
1. key分布不均匀
2. 业务数据本身的特性
3. 建表时考虑不周
4. 某些SQL语句本身就有数据倾斜

关于数据倾斜的问题在本文开头部分已经有举例，常见的倾斜分为group by造成的倾斜和join造成的倾斜。

首先Join造成的倾斜。
Hive给出的解决方案叫skew join，其原理把某些会产生倾斜的的特殊值先不在Reduce端计算掉，而是先写入hdfs，然后启动一轮Map join专门做这个特殊值的计算，期望能提高计算这部分值的处理速度。当然你要告诉Hive这个join是个skew join，即：

    set Hive.optimize.skewjoin = true;

还有要告诉Hive如何判断特殊值，根据`hive.skewjoin.key`设置的数量Hive可以知道，比如默认值是100000，那么超过100000条记录的值就是特殊值。

skew join的处理流程如下图所示：

{% img /img/hive/1469974304534.jpg skew %}

其次，group by造成的倾斜有两个参数可以解决，一个是
`hive.Map.aggr`，默认值已经为true，意思是会做Map端的combiner。
另一个参数是
`hive.groupby.skewindata`。这个参数的意思是做Reduce操作的时候，拿到的key并不是所有相同值给同一个Reduce，而是随机分发，然后Reduce做聚合，做完之后再做一轮MR，拿前面聚合过的数据再算结果。所以这个参数其实跟Hive.Map.aggr做的是类似的事情，只是拿到Reduce端来做，而且要额外启动一轮Job，所以其实不怎么推荐用，效果不明显。

如果说要改写SQL来优化的话，可以按照下面这么做：

    /*改写前*/
    select a， count(distinct b) as c from tbl group by a;
    /*改写后*/
    select a， count(*) as c
      from (select distinct a， b from tbl)
     group by a;

总结起来，Hive确实好用，上手也容易，但是真正做到 “用好” 却不容易，且用且珍惜。


【参考文献】
1. Hive SQL 编译过程 http://my.oschina.net/leejun2005/blog/267219
2. [Hive A Warehousing Solution Over a MapReduce Framework](http://www.vldb.org/pvldb/2/vldb09-938.pdf)
3. Internal Hive http://www.slideshare.net/recruitcojp/internal-hive
4. 深入浅出数据仓库中SQL性能优化之Hive篇 http://www.csdn.net/article/2015-01-13/2823530
5. http://my.oschina.net/leejun2005/blog/77701
6. http://blog.csdn.net/longshenlmj/article/details/17304437
7. http://blog.csdn.net/bitcarmanlee/article/details/51694101

