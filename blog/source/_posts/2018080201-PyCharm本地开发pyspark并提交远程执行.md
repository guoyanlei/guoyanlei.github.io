title: PyCharm本地开发pyspark并提交远程执行
date: 2018/08/01 15:30:00
tags:
- pyspark
categories:
- pyspark

---


最近在学习pyspark的开发，遇到些问题记录下。

我们在开发pyspark时经常需要进行测试，自己电脑上安装搭建一个spark环境代价有点高，目前有的同事在开发时，通常是开发完把代码贴出到本地测试集群进行测试，因此，能不能借助pycharm里的一个功能，连接本地测试集群的pyspark进行执行呢，经过一番搜索终于实现了这一个功能。

<!--more-->


### 新建带有Virtualenv的工程


Virtualenv是什么？

Python 的第三方包成千上万，在一个 Python 环境下开发时间越久、安装依赖越多，就越容易出现依赖包冲突的问题。为了解决这个问题，开发者们开发出了 virtualenv，可以搭建虚拟且独立的 Python 环境。这样就可以使每个项目环境与其他项目独立开来，保持环境的干净，解决包冲突问题。

下面我们先见一个project，在pycharm中默认的是Virtualenv管理环境，当然还有conda，功能和Virtualenv类似。

![Virtualenv的工程](/img/pycharm/new-project.png)


### 开发&导入pyspark

新建一个py文件

```
from pyspark import SparkContext
from pyspark.sql import HiveContext

APP_NAME = 'APP_TEST'

sc = SparkContext(appName=APP_NAME)
sqlContext = HiveContext(sc)

sqlContext.sql("show databases").show()

```

引入未安装的包时，pycharm会提示安装，安装需借助pip安装，若为安装pip需先安装，安装完成后会在项目目录：venv/lib/python2.7/site-packages 中出现安装好的第三方库。

至此，我们就可以开发和使用pyspark中的一些文件了。

开发是方便，但是我们还想变开发变测试，由于本地并没有装大数据相关环境，因此，我们还需要做些配置，来远程提交我们的代码并执行测试。

### 提交远程执行

#### 配置远程project interpreter（程序解释器）

打开pycharm的prefrences配置，从中找到我们项目的project interpreter。

![project interpreter](/img/pycharm/interpreter.png)

图中展示的是我们本机中导入的程序解释器，我们还可以点击右上方配置按钮，添加远程的程序解释器。

![ssh-interpreter](/img/pycharm/ssh-interpreter.png)

在配置中添加远程服务器的主机ip和用户、密码

![ssh-interpreter-2](/img/pycharm/ssh-interpreter-2.png)

在最后一页，可以看到，远程服务器中python的位置。我们在执行程序时，pycharm会自动的将我们的最新代码提交的远程的目录下（/tmp/pycharm_project_237）

当我们切换到远程的project interpreter时，就会看到远程的一些库。我们在开发时，远程服务器缺少哪些库，就需要先到服务器上安装好那些库，之后才可以提交到远程执行，否则会报no module find。

![project interpreter remote](/img/pycharm/interpreter-2.png)


#### 远程执行

上面的配置完成后，就可以在当前文件的Edit Configuration中进行执行配置，在project interpreter选项中可以选择是使用本地的程序解释器还是远程程序解释器执行了。

![Edit Configuration](/img/pycharm/run-configuration.png)


执行上面定义好的的程序。

![Running](/img/pycharm/running.png)

可以看出，实际程序执行的是同步到远程服务器的代码。并借助ssh登录到远程服务器执行，并返回执行的结果。

```
ssh://root@node102.bigdata.dmp.local.com:22/usr/bin/python -u /tmp/pycharm_project_568/suishen/uid_to_mongo.py
```

这样就很方便了，不需要每次边写代码，边复制粘贴进行测试了。

### CDH-spark在执行时遇到的问题

由于服务器安装的是CDH版本的spark，因此，在执行pyspark程序时需要指定SPARK_HOME。

解决方案有很多种，可参考（https://blog.csdn.net/syani/article/details/72851425）

我们使用的是第二种，在代码中灵活的配置SPARK_HOME和其他的环境地址。

```
import os
import sys

os.environ['SPARK_HOME'] = "/data/dmp/cloudera/parcels/CDH-5.12.2-1.cdh5.12.2.p0.4/lib/spark"
sys.path.append("/data/dmp/cloudera/parcels/CDH-5.12.2-1.cdh5.12.2.p0.4/lib/spark/python")
sys.path.append("/data/dmp/cloudera/parcels/CDH-5.12.2-1.cdh5.12.2.p0.4/lib/spark/python/lib/py4j-0.9-src.zip")

```






