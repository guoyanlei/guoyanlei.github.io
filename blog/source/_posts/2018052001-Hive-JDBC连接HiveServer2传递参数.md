
---
title: Hive-JDBC连接HiveServer2传递参数
date: 2018/05/20 22:30:04
tags:
- hive
- HiveServer2
categories:
- hive
---

HiveServer2（以下简称：HS2）是Hive提供的一种jdbc服务，用户可以通过Hive自带的Beeline连接，也可以使用Java、Python或者PHP等通过jdbc的方式连接。下面以Java连接HiveServer2为例来介绍几种向Hive传递参数的方法。

### Java-JDBC连接HS2

连接到HS2，一般需要提供HS2的地址、端口号、连接的Hive库、用户名和密码这几个必选项，示例代码如下：

```
Class.forName("org.apache.hive.jdbc.HiveDriver");
Properties info = new Properties();
info.setProperty("user", "user_name");
info.setProperty("password", "passwd");
String JDBC_URL="jdbc:hive2://localhost:10000/default";
Connection conn = DriverManager.getConnection(JDBC_URL, info);
HiveStatement stat = (HiveStatement) conn.createStatement();

```

只要URL、用户名和密码正确的话，通过上面的示例代码，就可以连接到HS2执行操作了。但是往往这样还不够，如果我们想通过传递一些Hive的配置信息，那该怎么办呢？


### 传递参数

#### 1.类HiveClient方式

使用过Hive客户端的用户都知道，如果我们想改变Hive的某一项客户端配置的话，可以通过set hive_conf_key=hive_conf_value;的方式来修改。由此，很自然的我们会想到在获得了一个JDBC的连接后，我们执行一下上面的语句不就可以了。示例代码如下：

```
// 我们执行一些复杂的sql的时候，往往需要制定一个队列，假设队列的名字为"root.hive-server2"
stat.execute("set mapreduce.job.queuename=root.hive-server2"); 

```

NOTE：需要注意的是，execute中的set语句不能包含分号（不能是set mapreduce.job.queuename=root.hive-server2;），这是和客户端的区别，否则不生效

#### 2.在JDBC URL中传

对jdbc比较熟悉的用户，都知道可以在jdbc的连接中传递一些参数，hive也一样支持。对于上面的需求，可以把充分利用JDBC_URL。示例代码如下：

```
String JDBC_URL="jdbc:hive2://localhost:10000/default?mapreduce.job.queuename=root.hive-server2;hive.cli.print.header=false";
Connection conn = DriverManager.getConnection(JDBC_URL, info);

```

NOTE：普通的jdbc传递参数不一样的地方，那就是这里是通过使用分号来分割多个hive的配置变量的，而不是使用'&'。

NOTE：另外，这里传递hive配置和hive变量还是有区别的，Hive是通过'#'来分割Hive配置列表和Hive变量列表的。

```
// 源码HiveConnection.java有说明
// JDBC URL: jdbc:hive2://<host>:<port>/dbName;sess_var_list?hive_conf_list#hive_var_list
// each list: <key1>=<val1>;<key2>=<val2> and so on

参考：
https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients#HiveServer2Clients-UsingJDBC
```

#### 3.通过连接属性配置

如果需要传递的配置数目比较多，使用上面的方法，难免有点冗余和负杂，URL将会变得特别长。其实，我们可以像配置user和password一样来传递配置。区别于user和password配置方式的地方是，必须明确指出配置的是一个hive_conf还是hive_var，否则配置不会生效。示例代码如下：

```
Class.forName("org.apache.hive.jdbc.HiveDriver");
Properties info = new Properties();
info.setProperty("user", "user_name");
info.setProperty("password", "passwd");
// 这里传递了一个队列的hive_conf
info.setProperty("hiveconf:mapreduce.job.queuename", "root.hive-server2");
String JDBC_URL="jdbc:hive2://localhost:10000/default";
Connection conn = DriverManager.getConnection(JDBC_URL, info);
HiveStatement stat = (HiveStatement) conn.createStatement();

```


