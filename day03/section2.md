# 4.3 实时召回集业务

## 学习目标

- 目标
  - 实时内容召回的作用
- 应用
  - 应用spark streaming完成实时召回集的创建

### 4.3.1 实时召回实现

实时召回会用基于画像相似的文章推荐

创建online文件夹，建立在线实时处理程序

![](../images/在线目录.png)

* 目的：对用户日志进行处理，实时达到求出相似文章，放入用户召回集合中
* 步骤：
  * 1、配置spark streaming信息
  * 2、读取点击行为日志数据，获取相似文章列表
  * 3、过滤历史文章集合
  * 4、存入召回结果以及历史记录结果
  * 5、写入日志进行测试

####1、创建spark streaming配置信息以及happybase

导入默认的配置，SPARK_ONLINE_CONFIG

```python
# 增加spark online 启动配置
class DefaultConfig(object):
    """默认的一些配置信息
    """
    SPARK_ONLINE_CONFIG = (
        ("spark.app.name", "onlineUpdate"),  # 设置启动的spark的app名称，没有提供，将随机产生一个名称
        ("spark.master", "yarn"),
        ("spark.executor.instances", 4)
    )
```

配置StreamingContext，在online的\_\_init\_\_.py文件添加，导入模块时直接使用

```python
# 添加sparkstreaming启动对接kafka的配置

from pyspark import SparkConf
from pyspark.sql import SparkSession
from pyspark import SparkContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils
from setting.default import DefaultConfig

import happybase

#  用于读取hbase缓存结果配置
pool = happybase.ConnectionPool(size=10, host='hadoop-master', port=9090)
# 1、创建conf
conf = SparkConf()
conf.setAll(DefaultConfig.SPARK_ONLINE_CONFIG)
# 建立spark session以及spark streaming context
sc = SparkContext(conf=conf)
# 创建Streaming Context
stream_c = StreamingContext(sc, 60)
```

配置streaming 读取Kafka的配置,在配置文件中增加KAFKAIP和端口

```
# KAFKA配置
KAFKA_SERVER = "192.168.19.137:9092"
```

```python
# 基于内容召回配置，用于收集用户行为，获取相似文章实时推荐
similar_kafkaParams = {"metadata.broker.list": DefaultConfig.KAFKA_SERVER, "group.id": 'similar'}
SIMILAR_DS = KafkaUtils.createDirectStream(stream_c, ['click-trace'], similar_kafkaParams)
```

创建online_update文件，建立在线召回类

```python
import os
import sys
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
sys.path.insert(0, os.path.join(BASE_DIR))
from online import stream_sc, SIMILAR_DS, pool
from datetime import datetime
from setting.default import DefaultConfig
import redis
import json
import time
import setting.logging as lg
import logging
```

#### 注意添加运行时环境

```python
# 注意，如果是使用jupyter或ipython中，利用spark streaming链接kafka的话，必须加上下面语句
# 同时注意：spark version>2.2.2的话，pyspark中的kafka对应模块已被遗弃，因此这里暂时只能用2.2.2版本的spark
os.environ["PYSPARK_SUBMIT_ARGS"] = "--packages org.apache.spark:spark-streaming-kafka-0-8_2.11:2.2.2 pyspark-shell"
```

- **2、Kafka读取点击行为日志数据，获取相似文章列表**

传入Kafka的数据：

```
OK
{"actionTime":"2019-04-10 21:04:39","readTime":"","channelId":18,"param":{"action": "click", "userId": "2", "articleId": "116644", "algorithmCombine": "C2"}}
Time taken: 3.72 seconds, Fetched: 1 row(s)
```

- 3、过滤历史文章集合
- 4、存入召回结果以及历史记录结果

```python
   class OnlineRecall(object):
    """在线处理计算平台
    """
    def __init__(self):
        pass

    def _update_content_recall(self):
        """
        在线内容召回计算
        :return:
        """
        # {"actionTime":"2019-04-10 21:04:39","readTime":"","channelId":18,
        # "param":{"action": "click", "userId": "2", "articleId": "116644", "algorithmCombine": "C2"}}
        # x [,'json.....']
        def get_similar_online_recall(rdd):
            """
            解析rdd中的内容，然后进行获取计算
            :param rdd:
            :return:
            """
            # rdd---> 数据本身
            # [row(1,2,3), row(4,5,6)]----->[[1,2,3], [4,5,6]]
            import happybase
            # 初始化happybase连接
            pool = happybase.ConnectionPool(size=10, host='hadoop-master', port=9090)
            for data in rdd.collect():

                # 进行data字典处理过滤
                if data['param']['action'] in ["click", "collect", "share"]:

                    logger.info(
                        "{} INFO: get user_id:{} action:{}  log".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                                                                        data['param']['userId'], data['param']['action']))

                    # 读取param当中articleId，相似的文章
                    with pool.connection() as conn:

                        sim_table = conn.table("article_similar")

                        # 根据用户点击流日志涉及文章找出与之最相似文章(基于内容的相似)，选取TOP-k相似的作为召回推荐结果
                        _dic = sim_table.row(str(data["param"]["articleId"]).encode(), columns=[b"similar"])
                        _srt = sorted(_dic.items(), key=lambda obj: obj[1], reverse=True)  # 按相似度排序
                        if _srt:

                            topKSimIds = [int(i[0].split(b":")[1]) for i in _srt[:10]]

                            # 根据历史推荐集过滤，已经给用户推荐过的文章
                            history_table = conn.table("history_recall")

                            _history_data = history_table.cells(
                                b"reco:his:%s" % data["param"]["userId"].encode(),
                                b"channel:%d" % data["channelId"]
                            )

                            history = []
                            if len(_history_data) > 1:
                                for l in _history_data:
                                    history.extend(l)

                            # 根据历史召回记录，过滤召回结果
                            recall_list = list(set(topKSimIds) - set(history))

                            # 如果有推荐结果集，那么将数据添加到cb_recall表中，同时记录到历史记录表中
                            logger.info(
                                "{} INFO: store online recall data:{}".format(
                                    datetime.now().strftime('%Y-%m-%d %H:%M:%S'), str(recall_list)))

                            if recall_list:

                                recall_table = conn.table("cb_recall")

                                recall_table.put(
                                    b"recall:user:%s" % data["param"]["userId"].encode(),
                                    {b"online:%d" % data["channelId"]: str(recall_list).encode()}
                                )

                                history_table.put(
                                    b"reco:his:%s" % data["param"]["userId"].encode(),
                                    {b"channel:%d" % data["channelId"]: str(recall_list).encode()}
                                )
                        conn.close()

        # x可以是多次点击行为数据，同时拿到多条数据
        SIMILAR_DS.map(lambda x: json.loads(x[1])).foreachRDD(get_similar_online_recall)
```

开启实时运行,同时增加日志打印

```python
if __name__ == '__main__':
    # 启动日志配置
    lg.create_logger()
    op = OnlineRecall()
    op._update_online_cb()
    stream_c.start()
    # 使用 ctrl+c 可以退出服务
    _ONE_DAY_IN_SECONDS = 60 * 60 * 24
    try:
        while True:
            time.sleep(_ONE_DAY_IN_SECONDS)
    except KeyboardInterrupt:
        pass
```

#### 添加文件打印日志

```python
# 添加到需要打印日志内容的文件中
logger = logging.getLogger('online')


# 在线更新日志
# 离线处理更新打印日志
trace_file_handler = logging.FileHandler(
  os.path.join(logging_file_dir, 'online.log')
)
trace_file_handler.setFormatter(logging.Formatter('%(message)s'))
log_trace = logging.getLogger('online')
log_trace.addHandler(trace_file_handler)
log_trace.setLevel(logging.INFO)
```

**添加日志数据，进行测试**

```
echo {\"actionTime\":\"2019-04-10 21:04:39\",\"readTime\":\"\",\"channelId\":18,\"param\":{\"action\": \"click\", \"userId\": \"2\", \"articleId\": \"116644\", \"algorithmCombine\": \"C2\"}} >> userClick.log
```

```
2019-05-29 07:37:00 INFO: get user_id:2 action:click  log
2019-05-29 07:37:02 INFO: store online recall data:[118560, 18530, 59747, 118370, 49962, 17613, 117199, 15315, 118550, 140376]
```

结果查询：

```python
hbase(main):028:0> get 'cb_recall', 'recall:user:2'
COLUMN                     CELL                                                                        
 als:13                    timestamp=1558041569201, value=[141431]                                     
 als:18                    timestamp=1558041572176, value=[19200, 17665, 19476, 16151, 16411, 19233, 13
                           090, 15140, 16421, 19494, 14381, 17966, 17454, 14125, 16174, 14899, 44339, 1
                           6437, 18743, 44090, 18238, 13890, 14915, 15429, 15944, 44371, 18005, 15196, 
                           13410, 13672, 44137, 18795, 19052, 44652, 44654, 44657, 14961, 17522, 43894,
                            44412, 16000, 14208, 44419, 17802, 14223, 18836, 140956, 18335, 13728, 1449
                           8, 44451, 44456, 18609, 18353, 44468, 18103, 13757, 14015, 13249, 44739, 444
                           83, 17605, 14021, 15309, 18127, 43983, 44754, 43986, 19413, 14805, 18904, 44
                           761, 17114, 13272, 14810, 18907, 13022, 14299, 17120, 17632, 43997, 17889, 1
                           7385, 18156, 15085, 13295, 44020, 14839, 44024, 14585, 18172, 44541]        
 als:5                     timestamp=1558041564163, value=[141440]                                     
 als:7                     timestamp=1558041568776, value=[141437]                                     
 online:18                 timestamp=1559086622107, value=[118560, 18530, 59747, 118370, 49962, 17613, 
                           117199, 15315, 118550, 140376]
```



