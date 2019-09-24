# 4.4 热门与新文章召回

## 学习目标

- 目标
  - 了解热门与新文章召回作用
- 应用
  - 应用spark streaming完成召回创建

### 4.4.1 热门文章与新文章

* 热门文章

通过对日志数据的处理，来实时增加文章的点击次数等信息

* 新文章由头条后台审核通过的文章传入kafka

* redis:10

| 新文章召回  |   结构    | 示例      |
| ----------- | :-------: | --------- |
| new_article | ch:{}:new | ch:18:new |

| 热门文章召回   | 结构      | 示例      |
| -------------- | --------- | --------- |
| popular_recall | ch:{}:hot | ch:18:hot |

```python
# 新文章存储
# ZADD ZRANGE
# ZADD key score member [[score member] [score member] ...]
# ZRANGE page_rank 0 -1
client.zadd("ch:{}:new".format(channel_id), {article_id: time.time()})

# 热门文章存储
# ZINCRBY key increment member
# ZSCORE
# 为有序集 key 的成员 member 的 score 值加上增量 increment 。
client.zincrby("ch:{}:hot".format(row['channelId']), 1, row['param']['articleId'])

# ZREVRANGE key start stop [WITHSCORES]
client.zrevrange(ch:{}:new, 0, -1)
```

### 4.4.2 添加热门以及新文章kafka配置信息

```python
# 添加sparkstreaming启动对接kafka的配置
# 配置KAFKA相关，用于热门文章KAFKA读取
click_kafkaParams = {"metadata.broker.list": DefaultConfig.KAFKA_SERVER}
HOT_DS = KafkaUtils.createDirectStream(stream_c, ['click-trace'], click_kafkaParams)

# new-article，新文章的读取  KAFKA配置
NEW_ARTICLE_DS = KafkaUtils.createDirectStream(stream_c, ['new-article'], click_kafkaParams)
```

并且导入相关包

```
from online import HOT_DS, NEW_ARTICLE_DS
```

然后，并且在kafka启动脚本中添加，**先在supervisor中关闭flume与kafka的进程**

```
/root/bigdata/kafka/bin/kafka-topics.sh --zookeeper 192.168.19.137:2181 --create --replication-factor 1 --topic new-article --partitions 1
```

然后supervisor start重新启动

增加一个新文章的topic，这里会与后台对接

### 4.4.3 编写热门文章收集程序

* 在线实时进行redis读取存储

在setting中增加redis的配置

```
# redis的IP和端口配置
REDIS_HOST = "192.168.19.137"
REDIS_PORT = 6379
```

然后添加初始化信息

```python
class OnlineRecall(object):
    """实时处理（流式计算）部分
    """

    def __init__(self):

        self.client = redis.StrictRedis(host=DefaultConfig.REDIS_HOST,
                                        port=DefaultConfig.REDIS_PORT,
                                        db=10)
        # 在线召回筛选TOP-k个结果
        self.k = 20
```

#### 收集热门文章代码：

```python
    def _update_hot_redis(self):
        """更新热门文章  click-trace
        :return:
        """
        client = self.client

        def updateHotArt(rdd):
            for row in rdd.collect():
                logger.info("{}, INFO: {}".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), row))
                # 如果是曝光参数，和阅读时长选择过滤
                if row['param']['action'] == 'exposure' or row['param']['action'] == 'read':
                    pass
                else:
                    # 解析每条行为日志，然后进行分析保存点击，喜欢，分享次数，这里所有行为都自增1
                    client.zincrby("ch:{}:hot".format(row['channelId']), 1, row['param']['articleId'])

        HOT_DS.map(lambda x: json.loads(x[1])).foreachRDD(updateHotArt)

        return None
```

结果，进行测试

```
[root@hadoop-master logs]# echo {\"actionTime\":\"2019-04-10 21:04:39\",\"readTime\":\"\",\"channelId\":18,\"param\":{\"action\": \"click\", \"userId\": \"2\", \"articleId\": \"14299\", \"algorithmCombine\": \"C2\"}} >> userClick.log
```

然后打印日志结果

```python
2019-05-18 03:24:01, INFO: {'actionTime': '2019-04-10 21:04:39', 'readTime': '', 'channelId': 18, 'param': {'action': 'click', 'userId': '2', 'articleId': '14299', 'algorithmCombine': 'C2'}}
```

最后查询redis当中是否存入结果热门文章

```python
127.0.0.1:6379[10]> keys *
1) "ch:18:hot"
127.0.0.1:6379[10]> ZRANGE "ch:18:hot" 0 -1
1) "14299"
127.0.0.1:6379[10]> 
  
# ZREM 'ch:18:hot' 0, -1  可删除之前的结果
```

### 4.4.4 编写新文章收集程序

新文章如何而来，黑马头条后台在文章发布之后，会将新文章ID以固定格式传到KAFKA的new-article topic当中

#### 新文章代码

```python
    def _update_new_redis(self):
        """更新频道新文章 new-article
        :return:
        """
        client = self.client

        def computeFunction(rdd):
            for row in rdd.collect():
                channel_id, article_id = row.split(',')
                logger.info("{}, INFO: get kafka new_article each data:channel_id{}, article_id{}".format(
                    datetime.now().strftime('%Y-%m-%d %H:%M:%S'), channel_id, article_id))
                client.zadd("ch:{}:new".format(channel_id), {article_id: time.time()})

        NEW_ARTICLE_DS.map(lambda x: x[1]).foreachRDD(computeFunction)

        return None
```

测试:pip install kafka-python

查看所有本地topic情况

```python
from kafka import KafkaClient
client = KafkaClient(hosts="127.0.0.1:9092")
for topic in client.topics:
    print(topic)
```

```python

from kafka import KafkaProducer 

# kafka消息生产者
kafka_producer = KafkaProducer(bootstrap_servers=['192.168.19.137:9092'])

# 构造消息并发送
msg = '{},{}'.format(18, 13891)
kafka_producer.send('new-article', msg.encode())
```

可以得到redis结果

```python
127.0.0.1:6379[10]> keys *
1) "ch:18:hot"
2) "ch:18:new"
127.0.0.1:6379[10]> ZRANGE "ch:18:new" 0 -1
1) "13890"
2) "13891"
```

### 4.4.5 添加supervisor在线实时运行进程管理

增加以下配置

```python
[program:online]
environment=JAVA_HOME=/root/bigdata/jdk,SPARK_HOME=/root/bigdata/spark,HADOOP_HOME=/root/bigdata/hadoop,PYSPARK_PYTHON=/miniconda2/envs/reco_sys/bin/python ,PYSPARK_DRIVER_PYTHON=/miniconda2/envs/reco_sys/bin/python,PYSPARK_SUBMIT_ARGS='--packages org.apache.spark:spark-streaming-kafka-0-8_2.11:2.2.2 pyspark-shell'
command=/miniconda2/envs/reco_sys/bin/python /root/toutiao_project/reco_sys/online/online_update.py
directory=/root/toutiao_project/reco_sys/online
user=root
autorestart=true
redirect_stderr=true
stdout_logfile=/root/logs/onlinesuper.log
loglevel=info
stopsignal=KILL
stopasgroup=true
killasgroup=true
```

```python
supervisor> update
online: added process group
supervisor> status
collect-click                    RUNNING   pid 97209, uptime 6:46:53
kafka                            RUNNING   pid 105159, uptime 6:20:09
offline                          STOPPED   Apr 16 04:31 PM
online                           RUNNING   pid 124591, uptime 0:00:02
supervisor> 
```