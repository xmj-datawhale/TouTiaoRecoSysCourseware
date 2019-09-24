# 2.3 用户行为收集到HIVE

## 学习目标

- 目标
  - 知道收集用户日志形式、流程
  - 知道flume收集相关配置、hive相关配置
  - 知道supervisor开启flume收集进程管理
- 应用
  - 应用supervisor管理flume实时收集点击日志

### 2.3.1 为什么要收集用户点击行为日志

用户行为对于黑马头条文章推荐来说，至关重要。用户的行为代表的每一次的喜好反馈，我们需要收集起来并且存到HIVE

* 便于了解分析用户的行为、喜好变化
* 为用户建立画像提供依据

![](../images/黑马头条宣传.png)

### 2.3.2 用户日志如何收集

####2.3.2.1 埋点开发测试流程

一般用户有很多日志，我们当前黑马头条推荐场景统一到行为日志中，还有其它业务场景如(下单日志、支付日志)

* 埋点参数
  * **就是在应用中特定的流程收集一些信息，用来跟踪应用使用的状况，后续用来进一步优化产品或是提供运营的数据支撑**
  * 重要性：埋点数据是推荐系统的基石，模型训练和效果数据统计都基于埋点数据，需保证埋点数据的正确无误
* 流程：
  * 1、PM（项目经理）、算法推荐工程师一起指定埋点需求文档
  * 2、后端、客户端 APP集成
  * 3、推荐人员基于文档埋点测试与梳理

#### 2.3.2.2 黑马头条文章推荐埋点需求整理

文章行为特点：点击、浏览、收藏、分享等行为，基于这些我们制定需求

- 埋点场景

- - 首页中的各频道推荐

* 埋点事件号

* - 停留时间

  - - read

  - 点击事件

  - - click

  - 曝光事件(相当于刷新一次请求推荐新文章)

  - - exposure

  - 收藏事件

  - - collect

  - 分享事件

  - - share

* 埋点参数文件结构

```python
# 曝光的参数，
{"actionTime":"2019-04-10 18:15:35","readTime":"","channelId":0,"param":{"action": "exposure", "userId": "2", "articleId": "[18577, 14299]", "algorithmCombine": "C2"}}

# 对文章发生行为的参数
{"actionTime":"2019-04-10 18:12:11","readTime":"2886","channelId":18,"param":{"action": "read", "userId": "2", "articleId": "18005", "algorithmCombine": "C2"}}
{"actionTime":"2019-04-10 18:15:32","readTime":"","channelId":18,"param":{"action": "click", "userId": "2", "articleId": "18005", "algorithmCombine": "C2"}}
{"actionTime":"2019-04-10 18:15:34","readTime":"1053","channelId":18,"param":{"action": "read", "userId": "2", "articleId": "18005", "algorithmCombine": "C2"}}
{"actionTime":"2019-04-10 18:15:36","readTime":"","channelId":18,"param":{"action": "click", "userId": "2", "articleId": "18577", "algorithmCombine": "C2"}}
{"actionTime":"2019-04-10 18:15:38","readTime":"1621","channelId":18,"param":{"action": "read", "userId": "2", "articleId": "18577", "algorithmCombine": "C2"}}
{"actionTime":"2019-04-10 18:15:39","readTime":"","channelId":18,"param":{"action": "click", "userId": "1", "articleId": "14299", "algorithmCombine": "C2"}}
{"actionTime":"2019-04-10 18:15:39","readTime":"","channelId":18,"param":{"action": "click", "userId": "2", "articleId": "14299", "algorithmCombine": "C2"}}
{"actionTime":"2019-04-10 18:15:41","readTime":"914","channelId":18,"param":{"action": "read", "userId": "2", "articleId": "14299", "algorithmCombine": "C2"}}
{"actionTime":"2019-04-10 18:15:47","readTime":"7256","channelId":18,"param":{"action": "read", "userId": "1", "articleId": "14299", "algorithmCombine": "C2"}}
```

我们将埋点参数设计成一个固定格式的json字符串，它包含了事件发生事件、算法推荐号、获取行为的频道号、帖子id列表、帖子id、用户id、事件号字段。

### 2.3.3 离线部分-用户日志收集

#### 2.3.3.1目的：通过flume将业务数据服务器A的日志收集到hadoop服务器hdfs的hive中

注意：这里我们都在hadoop-master上操作

#### 2.3.3.2 收集步骤

* 创建HIVE对应日志收集表
  * 收集到新的数据库中
* flume收集日志配置
* 开启收集命令

#### 2.3.3.3 实现

##### 1、flume读取设置

* 进入flume/conf目录

创建一个collect_click.conf的文件，写入flume的配置

* sources：为实时查看文件末尾，interceptors解析json文件
* channels：指定内存存储，并且制定batchData的大小，PutList和TakeList的大小见参数，Channel总容量大小见参数
* 指定sink：形式直接到hdfs，以及路径，文件大小策略默认1024、event数量策略、文件闲置时间

```python
a1.sources = s1
a1.sinks = k1
a1.channels = c1

a1.sources.s1.channels= c1
a1.sources.s1.type = exec
a1.sources.s1.command = tail -F /root/logs/userClick.log
a1.sources.s1.interceptors=i1 i2
a1.sources.s1.interceptors.i1.type=regex_filter
a1.sources.s1.interceptors.i1.regex=\\{.*\\}
a1.sources.s1.interceptors.i2.type=timestamp

# channel1
a1.channels.c1.type=memory
a1.channels.c1.capacity=30000
a1.channels.c1.transactionCapacity=1000

# k1
a1.sinks.k1.type=hdfs
a1.sinks.k1.channel=c1
a1.sinks.k1.hdfs.path=hdfs://192.168.19.137:9000/user/hive/warehouse/profile.db/user_action/%Y-%m-%d
a1.sinks.k1.hdfs.useLocalTimeStamp = true
a1.sinks.k1.hdfs.fileType=DataStream
a1.sinks.k1.hdfs.writeFormat=Text
a1.sinks.k1.hdfs.rollInterval=0
a1.sinks.k1.hdfs.rollSize=10240
a1.sinks.k1.hdfs.rollCount=0
a1.sinks.k1.hdfs.idleTimeout=60
```

##### 2、HIVE设置

解决办法可以按照日期分区

- 修改表结构
- 修改flume

在这里我们创建一个新的数据库profile，表示用户相关数据，画像存储到这里

```mysql
create database if not exists profile comment "use action" location '/user/hive/warehouse/profile.db/';
```

在profile数据库中创建user_action表，指定格式

```mysql
create table user_action(
actionTime STRING comment "user actions time",
readTime STRING comment "user reading time",
channelId INT comment "article channel id",
param map<string, string> comment "action parameter")
COMMENT "user primitive action"
PARTITIONED BY(dt STRING)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION '/user/hive/warehouse/profile.db/user_action';
```

- ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe':添加一个json格式匹配

##### 3. 开启收集命令

```
/root/bigdata/flume/bin/flume-ng agent -c /root/bigdata/flume/conf -f /root/bigdata/flume/conf/collect_click.conf -Dflume.root.logger=INFO,console -name a1
```

如果运行成功：

![](../images/flume第一次收集.png)

同时我们可以通过向userClick.log数据进行测试

```
echo {\"actionTime\":\"2019-04-10 21:04:39\",\"readTime\":\"\",\"channelId\":18,\"param\":{\"action\": \"click\", \"userId\": \"2\", \"articleId\": \"14299\", \"algorithmCombine\": \"C2\"}} >> userClick.log
```

重要：**这样就有HIVE的user_action表，并且hadoop有相应目录**,flume会自动生成目录，但是如果想要通过spark sql 获取内容，每天每次还是要主动关联，后面知识点会提及)

```python
select * from user_action where dt='yy-mm-dd'

# 如果flume自动生成目录后，需要手动关联分区
alter table user_action add partition (dt='2018-12-11') location "/user/hive/warehouse/profile.db/user_action/2018-12-11/"

select * from user_action where dt='yy-mm-dd' limit 1;
```

### 2.3.3 Supervisor进程管理

Supervisor作为进程管理工具，很方便的监听、启动、停止、重启一个或**多个**进程。用Supervisor管理的进程，当一个进程意外被杀死，supervisort监听到进程死后，会自动将它重新拉起，很方便的做到进程自动恢复的功能，不再需要自己写shell脚本来控制。

#### 安装

```shell
sudo yum install python-pip     # python2 pip   apt-get
```

**supervisor对python3支持不好，须使用python2环境**

```bash
sudo pip install supervisor
```

#### 配置

运行**echo\_supervisord\_conf**命令输出默认的配置项，可以如下操作将默认配置保存到文件中

```bash
echo_supervisord_conf > supervisord.conf
```

vim 打开编辑supervisord.conf文件，修改

```bash
[include]
files = relative/directory/*.ini
```

为

```bash
[include]
files = /etc/supervisor/*.conf
```

include选项指明包含的其他配置文件。

* 将编辑后的supervisord.conf文件复制到/etc/目录下

```bash
sudo cp supervisord.conf /etc/
```

* 然后我们在/etc目录下新建子目录supervisor（与配置文件里的选项相同），并在/etc/supervisor/中新建头条推荐管理的配置文件reco.conf

**加入配置模板如下(模板)**：

```
[program:recogrpc]
command=/root/anaconda3/envs/reco_sys/bin/python /root/headlines_project/recommend_system/ABTest/routing.py
directory=/root/headlines_project/recommend_system/ABTest
user=root
autorestart=true
redirect_stderr=true
stdout_logfile=/root/logs/reco.log
loglevel=info
stopsignal=KILL
stopasgroup=true
killasgroup=true

[program:kafka]
command=/bin/bash /root/headlines_project/scripts/startKafka.sh
directory=/root/headlines_project/scripts
user=root
autorestart=true
redirect_stderr=true
stdout_logfile=/root/logs/kafka.log
loglevel=info
stopsignal=KILL
stopasgroup=true
killasgroup=true
```

##### 启动

```bash
supervisord -c /etc/supervisord.conf
```

查看 supervisord 是否在运行：

```bash
ps aux | grep supervisord
```

##### supervisorctl

我们可以利用supervisorctl来管理supervisor。

```bash
supervisorctl

> status    # 查看程序状态
> start apscheduler  # 启动 apscheduler 单一程序
> stop toutiao:*   # 关闭 toutiao组 程序
> start toutiao:*  # 启动 toutiao组 程序
> restart toutiao:*    # 重启 toutiao组 程序
> update    ＃ 重启配置文件修改过的程序
```

执行status命令时，显示如下信息说明程序运行正常：

```bash
supervisor> status
toutiao:toutiao-app RUNNING pid 32091, uptime 00:00:02
```

### 2.3.4 启动监听flume收集日志程序

目的： 启动监听flume收集日志

* 我们将启动flume的程序建立成collect_click.sh脚本

flume启动需要相关hadoop,java环境，可以在shell程序汇总添加

```shell
#!/usr/bin/env bash

export JAVA_HOME=/root/bigdata/jdk
export HADOOP_HOME=/root/bigdata/hadoop
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin

/root/bigdata/flume/bin/flume-ng agent -c /root/bigdata/flume/conf -f /root/bigdata/flume/conf/collect_click.conf -Dflume.root.logger=INFO,console -name a1
```

* 并在supervisor的reco.conf添加

```python
[program:collect-click]
command=/bin/bash /root/toutiao_project/scripts/collect_click.sh
user=root
autorestart=true
redirect_stderr=true
stdout_logfile=/root/logs/collect.log
loglevel=info
stopsignal=KILL
stopasgroup=true
killasgroup=true
```

### 2.3.5 黑马头条HIVE历史点击数据导入

收集今天的点击行为数据，并且也只能生成今天一个日志目录，后面为了更多的日志数据处理，需要进行历史数据导入，这里我拷贝了历史数据在本地

这样就有

![](../images/历史日志信息导入.png)

* 步骤
  * 之前的Hadoop收集的若干时间之前的数据拷贝到hadoop对应hive数据目录中

* 实现

我们选择全部覆盖了，测试数据不要了

```shell
hadoop dfs -put /root/bak/hadoopbak/profile.db/user_action/ /user/hive/warehouse/profile.db/
```

### 2.3.6 总结

* 用户行为日志收集的相关工作流程
* flume收集到hive配置
* supervisor进程管理工具使用