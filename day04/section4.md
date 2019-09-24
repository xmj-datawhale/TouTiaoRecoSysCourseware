# 5.5 召回集读取与推荐中心对接

## 学习目标

- 目标
  - 无
- 应用
  - 无

### 5.5.1 召回集读取服务

![](../images/召回读取服务.png)

* 添加一个server的目录
  * 召回读取服务
  * **添加一个召回集的结果读取服务recall_service.py**

### 5.5.2 多路召回结果读取

* 目的：读取离线和在线存储的召回结果
  * hbase的存储：cb_recall, als, content, online

* 步骤：
  * 1、初始化redis,hbase相关工具
  * 2、在线画像召回，离线画像召回，离线协同召回数据的读取
  * 3、redis新文章和热门文章结果读取
  * 4、相似文章读取接口

初始化redis,hbase相关工具

```python
import os
import sys
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
sys.path.insert(0, os.path.join(BASE_DIR))

from server import redis_client
from server import pool
import logging
from datetime import datetime
from server.utils import HBaseUtils

logger = logging.getLogger('recommend')


class ReadRecall(object):
    """读取召回集的结果
    """
    def __init__(self):
        self.client = redis_client
        self.hbu = HBaseUtils(pool)
```

在init文件中添加相关初始化数据库变量

```python
import redis
import happybase
from setting.default import DefaultConfig
from pyspark import SparkConf
from pyspark.sql import SparkSession


pool = happybase.ConnectionPool(size=10, host="hadoop-master", port=9090)

# 加上decode_responses=True，写入的键值对中的value为str类型，不加这个参数写入的则为字节类型。
redis_client = redis.StrictRedis(host=DefaultConfig.REDIS_HOST,
                                 port=DefaultConfig.REDIS_PORT,
                                 db=10,
                                 decode_responses=True)
```

**2、在线画像召回，离线画像召回，离线协同召回数据的读取**

* 读取用户的指定列族的召回数据，并且读取之后要删除原来的推荐召回结果'cb_recall'

```python
    def read_hbase_recall_data(self, table_name, key_format, column_format):
        """
        读取cb_recall当中的推荐数据
        读取的时候可以选择列族进行读取als, online, content

        :return:
        """
        recall_list = []
        try:
            data = self.hbu.get_table_cells(table_name, key_format, column_format)

            # data是多个版本的推荐结果[[],[],[],]
            for _ in data:
                recall_list = list(set(recall_list).union(set(eval(_))))

            # self.hbu.get_table_delete(table_name, key_format, column_format)
        except Exception as e:
            logger.warning("{} WARN read {} recall exception:{}".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                                                                     table_name, e))
        return recall_list
```

测试：

```python
if __name__ == '__main__':
    rr = ReadRecall()
    # 召回结果的读取封装
    print(rr.read_hbase_recall_data('cb_recall', b'recall:user:1114864874141253632', b'als:18'))
```

**3、redis新文章和热门文章结果读取**

```python
    def read_redis_new_article(self, channel_id):
        """
        读取新闻章召回结果
        :param channel_id: 提供频道
        :return:
        """
        logger.warning("{} WARN read channel {} redis new article".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                                                                 channel_id))
        _key = "ch:{}:new".format(channel_id)
        try:
            res = self.client.zrevrange(_key, 0, -1)
        except Exception as e:
            logger.warning("{} WARN read new article exception:{}".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), e))
            res = []

        return list(map(int, res))
```

热门文章读取:热门文章记录了很多，可以选取前K个

```python
    def read_redis_hot_article(self, channel_id):
        """
        读取新闻章召回结果
        :param channel_id: 提供频道
        :return:
        """
        logger.warning("{} WARN read channel {} redis hot article".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), channel_id))
        _key = "ch:{}:hot".format(channel_id)
        try:
            res = self.client.zrevrange(_key, 0, -1)

        except Exception as e:
            logger.warning("{} WARN read new article exception:{}".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), e))
            res = []

        # 由于每个频道的热门文章有很多，因为保留文章点击次数
        res = list(map(int, res))
        if len(res) > self.hot_num:
            res = res[:self.hot_num]
        return res
```

测试：

```python
print(rr.read_redis_new_article(18))
print(rr.read_redis_hot_article(18))
```

**4、相似文章读取接口**

最后相似文章读取接口代码

* 会有接口获取固定的文章数量(用在黑马头条APP中的猜你喜欢接口)

```python
    def read_hbase_article_similar(self, table_name, key_format, article_num):
        """获取文章相似结果
        :param article_id: 文章id
        :param article_num: 文章数量
        :return:
        """
        # 第一种表结构方式测试：
        # create 'article_similar', 'similar'
        # put 'article_similar', '1', 'similar:1', 0.2
        # put 'article_similar', '1', 'similar:2', 0.34
        try:
            _dic = self.hbu.get_table_row(table_name, key_format)

            res = []
            _srt = sorted(_dic.items(), key=lambda obj: obj[1], reverse=True)
            if len(_srt) > article_num:
                _srt = _srt[:article_num]
            for _ in _srt:
                res.append(int(_[0].decode().split(':')[1]))
        except Exception as e:
            logger.error(
                "{} ERROR read similar article exception: {}".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), e))
            res = []
        return res
```

测试：

```python
print(rr.read_hbase_article_similar('article_similar', b'116644', 10))
```

完整代码：

```python
import os
import sys
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
sys.path.insert(0, os.path.join(BASE_DIR))

from server import redis_client
from server import pool
import logging
from datetime import datetime
from server.utils import HBaseUtils

logger = logging.getLogger('recommend')


class ReadRecall(object):
    """读取召回集的结果
    """
    def __init__(self):
        self.client = redis_client
        self.hbu = HBaseUtils(pool)

    def read_hbase_recall_data(self, table_name, key_format, column_format):
        """获取指定用户的对应频道的召回结果,在线画像召回，离线画像召回，离线协同召回
        :return:
        """
        # 获取family对应的值
        # 数据库中的键都是bytes类型，所以需要进行编码相加
        # 读取召回结果多个版本合并
        recall_list = []
        try:

            data = self.hbu.get_table_cells(table_name, key_format, column_format)
            for _ in data:
                recall_list = list(set(recall_list).union(set(eval(_))))

            # 读取所有这个用户的在线推荐的版本，清空该频道的数据
            # self.hbu.get_table_delete(table_name, key_format, column_format)
        except Exception as e:
            logger.warning(
                "{} WARN read recall data exception:{}".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), e))
        return recall_list

    def read_redis_new_data(self, channel_id):
        """获取redis新文章结果
        :param channel_id:
        :return:
        """
        # format结果
        logger.info("{} INFO read channel:{} new recommend data".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), channel_id))
        _key = "ch:{}:new".format(channel_id)
        try:
            res = self.client.zrevrange(_key, 0, -1)
        except redis.exceptions.ResponseError as e:
            logger.warning("{} WARN read new article exception:{}".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), e))
            res = []
        return list(map(int, res))

    def read_redis_hot_data(self, channel_id):
        """获取redis热门文章结果
        :param channel_id:
        :return:
        """
        # format结果
        logger.info("{} INFO read channel:{} hot recommend data".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), channel_id))
        _key = "ch:{}:hot".format(channel_id)
        try:
            _res = self.client.zrevrange(_key, 0, -1)
        except redis.exceptions.ResponseError as e:
            logger.warning("{} WARN read hot article exception:{}".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), e))
            _res = []
        # 每次返回前50热门文章
        res = list(map(int, _res))
        if len(res) > 50:
            res = res[:50]
        return res

    def read_hbase_article_similar(self, table_name, key_format, article_num):
        """获取文章相似结果
        :param article_id: 文章id
        :param article_num: 文章数量
        :return:
        """
        # 第一种表结构方式测试：
        # create 'article_similar', 'similar'
        # put 'article_similar', '1', 'similar:1', 0.2
        # put 'article_similar', '1', 'similar:2', 0.34
        try:
            _dic = self.hbu.get_table_row(table_name, key_format)

            res = []
            _srt = sorted(_dic.items(), key=lambda obj: obj[1], reverse=True)
            if len(_srt) > article_num:
                _srt = _srt[:article_num]
            for _ in _srt:
                res.append(int(_[0].decode().split(':')[1]))
        except Exception as e:
            logger.error("{} ERROR read similar article exception: {}".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), e))
            res = []
        return res


if __name__ == '__main__':

    rr = ReadRecall()
    print(rr.read_hbase_article_similar('article_similar', b'13342', 10))
    print(rr.read_hbase_recall_data('cb_recall', b'recall:user:1115629498121846784', b'als:18'))

    # rr = ReadRecall()
    # print(rr.read_redis_new_data(18))
```

### 5.5.3 推荐中心---获取多路召回结果，过滤历史推荐记录逻辑

* 目的：在推荐中加入召回文章结果以及过滤逻辑
* 步骤：
  * 1、循环算法组合参数，遍历不同召回结果进行过滤
  * 2、过滤当前该请求频道推荐历史结果，需要过滤0频道推荐结果，防止出现推荐频道与25个频道有重复推荐
  * 3、过滤之后，推荐出去指定个数的文章列表，写入历史记录，剩下多的写入待推荐结果

定义一个user_reco_list函数中实现读取用户的召回结果

- 1、循环算法组合参数，遍历不同召回结果进行过滤

```python
# 所有合并的结果
        reco_set = []
        # -  1、循环算法组合参数，遍历不同召回结果合并，18
        for _num in RAParam.COMBINE[temp.algo][1]:
            if _num == 103:
                # 读取新文章的结果，temp.channel_id
                _res = self.recall_service.read_redis_new_article(temp.channel_id)
                reco_set = list(set(reco_set).union(set(_res)))
            elif _num == 104:
                # 读取热门文章的数据
                _res = self.recall_service.read_redis_hot_article(temp.channel_id)
                reco_set = list(set(reco_set).union(set(_res)))
            else:
                # 读取具体编号的对应表cb_recall,对应召回算法的召回结果als, content, online
                _res = self.recall_service.\
                    read_hbase_recall_data(RAParam.RECALL[_num][0],
                                           'recall:user:{}'.format(temp.user_id).encode(),
                                           '{}:{}'.format(RAParam.RECALL[_num][1], temp.channel_id).encode())
                reco_set = list(set(reco_set).union(set(_res)))
```

- 2、过滤当前该请求频道推荐历史结果，如果不是0频道需要过滤0频道推荐结果，防止出现
  - 比如Python频道和0频道相同的推荐结果

```python
# - 2、过滤，该请求频道(18)的历史推荐记录过滤），推荐频道0频道
        #   - 0：APP  推荐(循环所有的频道召回结果)，0 频道也有历史记录
        # temp.channel_id频道这个用户历史记录进行过滤
        history_list = []

        try:
            data = self.hbu.get_table_cells('history_recommend',
                                            'reco:his:{}'.format(temp.user_id).encode(),
                                            'channel:{}'.format(temp.channel_id).encode())
            for _ in data:
                history_list = list(set(history_list).union(set(eval(_))))

            logger.info("{} INFO read user_id:{} channel_id:{} history data".format(
                datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))

        except Exception as e:
            # 打印日志
            logger.warning(
                "{} WARN filter history article exception:{}".format(datetime.now().
                                                                     strftime('%Y-%m-%d %H:%M:%S'), e))
        # 获取0频道的结果
        try:
            data = self.hbu.get_table_cells('history_recommend',
                                            'reco:his:{}'.format(temp.user_id).encode(),
                                            'channel:{}'.format(0).encode())
            for _ in data:
                history_list = list(set(history_list).union(set(eval(_))))
            logger.info("{} INFO filter user_id:{} channel:{} history data".format(
                datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, 0))

        except Exception as e:
            logger.warning(
                "{} WARN filter history article exception:{}".format(datetime.now().
                                                                     strftime('%Y-%m-%d %H:%M:%S'), e))

        reco_set = list(set(reco_set).difference(set(history_list)))
```

- 3、过滤之后，推荐出去指定个数的文章列表，写入历史记录，剩下多的写入待推荐结果

```python
# - 4、返回结果：
        if not reco_set:
            return reco_set
        else:
            # - 如果有数据，小于需要推荐文章的数量N之后，放入历史推荐记录中history_recommend，返回结果给用
            if len(reco_set) <= temp.article_num:
                res = reco_set
            else:
                #   - 如果有，350篇，取出N个，进行返回推荐，放入历史记录history_recommend
                #     - (350- N)个文章，放入wait_recommend
                res = reco_set[:temp.article_num]

                logger.info(
                    "{} INFO put user_id:{} channel:{} wait data".format(
                        datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))

                # 放入剩下多余的数据到wait_recommend当中
                self.hbu.get_table_put('wait_recommend',
                                       'reco:{}'.format(temp.user_id).encode(),
                                       'channel:{}'.format(temp.channel_id).encode(),
                                       str(reco_set[temp.article_num:]).encode(),
                                       timestamp=temp.time_stamp)

            # 放入历史记录
            self.hbu.get_table_put('history_recommend',
                                   'reco:his:{}'.format(temp.user_id).encode(),
                                   'channel:{}'.format(temp.channel_id).encode(),
                                   str(res).encode(),
                                   timestamp=temp.time_stamp)
            # 放入历史记录日志
            logger.info(
                "{} INFO store recall/sorted user_id:{} channel:{} history_recommend data".format(
                    datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))

            return res
```

**修改调用读取召回数据的部分**

```python
# 2、不开启缓存
res = self.user_reco_list(temp)
temp.time_stamp = int(last_stamp)
track = add_track(res, temp)
```

**运行grpc服务之后，测试结果**

```python
hbase(main):007:0> get "history_recommend", 'reco:his:1115629498121846784', {COLUMN=>'channel:18',VERSIONS=>1000}
COLUMN                     CELL                                                                        
 channel:18                timestamp=1558189615378, value=[13890, 14915, 13891, 15429, 15944, 44371, 18
                           005, 15196, 13410, 13672]                                                   
 channel:18                timestamp=1558189317342, value=[17966, 17454, 14125, 16174, 14899, 44339, 16
                           437, 18743, 44090, 18238]                                                   
 channel:18                timestamp=1558143073173, value=[19200, 17665, 16151, 16411, 19233, 13090, 15
                           140, 16421, 19494, 14381]
```

待推荐表中有

```python
hbase(main):008:0> scan 'wait_recommend'
ROW                        COLUMN+CELL                                                                 
 reco:1115629498121846784  column=channel:18, timestamp=1558189615378, value=[44137, 18795, 19052, 4465
                           2, 44654, 44657, 14961, 17522, 43894, 44412, 16000, 14208, 44419, 17802, 142
                           23, 18836, 140956, 18335, 13728, 14498, 44451, 44456, 18609, 18353, 44468, 1
                           8103, 135869, 16062, 14015, 13757, 13249, 44483, 17605, 14021, 15309, 18127,
                            43983, 44754, 43986, 19413, 14805, 18904, 44761, 17114, 13272, 14810, 18907
                           , 13022, 14300, 17120, 17632, 14299, 43997, 17889, 17385, 18156, 15085, 1329
                           5, 44020, 14839, 44024, 14585, 18172, 44541]
```

完整代码：

```python
    def user_reco_list(self, temp):
        """
        获取用户的召回结果进行推荐
        :param temp:
        :return:
        """
        reco_set = []
        # 1、循环算法组合参数，遍历不同召回结果进行过滤
        for _num in RAParam.COMBINE[temp.algo][1]:
            # 进行每个召回结果的读取100,101,102,103,104
            if _num == 103:
                # 新文章召回读取
                _res = self.recall_service.read_redis_new_article(temp.channel_id)
                reco_set = list(set(reco_set).union(set(_res)))
            elif _num == 104:
                # 热门文章召回读取
                _res = self.recall_service.read_redis_hot_article(temp.channel_id)
                reco_set = list(set(reco_set).union(set(_res)))
            else:
                _res = self.recall_service.\
                    read_hbase_recall_data(RAParam.RECALL[_num][0],
                                           'recall:user:{}'.format(temp.user_id).encode(),
                                           '{}:{}'.format(RAParam.RECALL[_num][1], temp.channel_id).encode())
                # 进行合并某个协同过滤召回的结果
                reco_set = list(set(reco_set).union(set(_res)))

        # reco_set都是新推荐的结果，进行过滤
        history_list = []
        try:
            data = self.hbu.get_table_cells('history_recommend',
                                            'reco:his:{}'.format(temp.user_id).encode(),
                                            'channel:{}'.format(temp.channel_id).encode())
            for _ in data:
                history_list = list(set(history_list).union(set(eval(_))))

            logger.info("{} INFO filter user_id:{} channel:{} history data".format(
                datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))
        except Exception as e:
            logger.warning(
                "{} WARN filter history article exception:{}".format(datetime.now().
                                                                     strftime('%Y-%m-%d %H:%M:%S'), e))

        # 如果0号频道有历史记录，也需要过滤

        try:
            data = self.hbu.get_table_cells('history_recommend',
                                            'reco:his:{}'.format(temp.user_id).encode(),
                                            'channel:{}'.format(0).encode())
            for _ in data:
                history_list = list(set(history_list).union(set(eval(_))))

            logger.info("{} INFO filter user_id:{} channel:{} history data".format(
                datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, 0))
        except Exception as e:
            logger.warning(
                "{} WARN filter history article exception:{}".format(datetime.now().
                                                                     strftime('%Y-%m-%d %H:%M:%S'), e))

        # 过滤操作 reco_set 与history_list进行过滤
        reco_set = list(set(reco_set).difference(set(history_list)))

        # 排序代码逻辑
        # _sort_num = RAParam.COMBINE[temp.algo][2][0]
        # reco_set = sort_dict[RAParam.SORT[_sort_num]](reco_set, temp, self.hbu)

        # 如果没有内容，直接返回
        if not reco_set:
            return reco_set
        else:

            # 类型进行转换
            reco_set = list(map(int, reco_set))

            # 跟后端需要推荐的文章数量进行比对 article_num
            # article_num > reco_set
            if len(reco_set) <= temp.article_num:
                res = reco_set
            else:
                # 之取出推荐出去的内容
                res = reco_set[:temp.article_num]
                # 剩下的推荐结果放入wait_recommend等待下次帅新的时候直接推荐
                self.hbu.get_table_put('wait_recommend',
                                       'reco:{}'.format(temp.user_id).encode(),
                                       'channel:{}'.format(temp.channel_id).encode(),
                                       str(reco_set[temp.article_num:]).encode(),
                                       timestamp=temp.time_stamp)
                logger.info(
                    "{} INFO put user_id:{} channel:{} wait data".format(
                        datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))

            # 放入历史记录表当中
            self.hbu.get_table_put('history_recommend',
                                   'reco:his:{}'.format(temp.user_id).encode(),
                                   'channel:{}'.format(temp.channel_id).encode(),
                                   str(res).encode(),
                                   timestamp=temp.time_stamp)
            # 放入历史记录日志
            logger.info(
                "{} INFO store recall/sorted user_id:{} channel:{} history_recommend data".format(
                    datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))

            return res
```

