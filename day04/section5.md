# 5.6 推荐缓存服务

## 学习目标

- 目标
  - 无
- 应用
  - 无

### 5.6.1 待推荐结果的redis缓存

* 目的：对待推荐结果进行二级缓存，多级缓存减少数据库读取压力
* 步骤：
  * 1、获取redis结果，进行判断
    * 如果redis有，读取需要推荐的文章数量放回，并删除这些文章，并且放入推荐历史推荐结果中
    * 如果redis当中不存在，则从wait_recommend中读取
      * 如果wait_recommend中也没有，直接返回
      * 如果wait_recommend有，从wait_recommend取出所有结果，定一个数量(如100篇)存入redis,剩下放回wait_recommend,不够100，全部放入redis，然后清空wait_recommend
      * 从redis中拿出要推荐的文章结果，然后放入历史推荐结果中

增加一个缓存数据库

```python
# 缓存在8号当中
cache_client = redis.StrictRedis(host=DefaultConfig.REDIS_HOST,
                                 port=DefaultConfig.REDIS_PORT,
                                 db=8,
                                 decode_responses=True)
```

1、redis 8 号数据库读取

```python
# 1、直接去redis拿取对应的键，如果为空
 # - 首先从获取redis结果，进行判断(缓存拿)
    key = 'reco:{}:{}:art'.format(temp.user_id, temp.channel_id)
    res = cache_client.zrevrange(key, 0, temp.article_num - 1)
    # - 如果redis有，读取需要推荐的文章数量放回，并删除这些文章，并且放入推荐历史推荐结果中
    if res:
        cache_client.zrem(key, *res)
```

2、redis没有数据，进行wait_recommend读取，放入redis中

```python
# - 如果redis当中不存在，则从wait_recommend中读取
        # 删除redis这个键
        cache_client.delete(key)
        try:
            # 字符串编程列表
            hbase_cache = eval(hbu.get_table_row('wait_recommend',
                                                 'reco:{}'.format(temp.user_id).encode(),
                                                 'channel:{}'.format(temp.channel_id).encode()))

        except Exception as e:
            logger.warning("{} WARN read user_id:{} wait_recommend exception:{} not exist".format(
                datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, e))

            hbase_cache = []
        if not hbase_cache:
            #   - 如果wait_recommend中也没有，直接返回空，去进行一次召回读取、排序然后推荐
            return hbase_cache
        #   - wait_recommend有，从wait_recommend取出所有结果，定一个数量(如100篇)的文章存入redis
        if len(hbase_cache) > 100:
            logger.info(
                "{} INFO reduce user_id:{} channel:{} wait_recommend data".format(
                    datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))
            # 拿出固定数量（100）给redis
            cache_redis = hbase_cache[:100]
            # 放入redis缓存
            cache_client.zadd(key, dict(zip(cache_redis, range(len(cache_redis)))))
            # 剩下的放回wait hbase结果
            hbu.get_table_put('wait_recommend',
                              'reco:{}'.format(temp.user_id).encode(),
                              'channel:{}'.format(temp.channel_id).encode(),
                              str(hbase_cache[100:]).encode())
        else:
            logger.info(
                "{} INFO delete user_id:{} channel:{} wait_recommend data".format(
                    datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))
            # - wait_recommend不够一定数量，全部取出放入redis当中，直接推荐出去
            # 清空wait_recommend
            hbu.get_table_put('wait_recommend',
                              'reco:{}'.format(temp.user_id).encode(),
                              'channel:{}'.format(temp.channel_id).encode(),
                              str([]).encode())

            # 放入redis缓存
            cache_client.zadd(key, dict(zip(hbase_cache, range(len(hbase_cache)))))

        res = cache_client.zrevrange(key, 0, temp.article_num - 1)
        if res:
            cache_client.zrem(key, *res)
```

3、推荐出去的结果放入历史结果

```python
# 进行类型转换
    res = list(map(int, res))
    logger.info("{} INFO get cache data and store user_id:{} channel:{} cache history data".format(
        datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))

    # 放入历史记录
    hbu.get_table_put('history_recommend',
                      'reco:his:{}'.format(temp.user_id).encode(),
                      'channel:{}'.format(temp.channel_id).encode(),
                      str(res).encode(),
                      timestamp=temp.time_stamp)
    return res
```

完整逻辑代码：

```python
from server import cache_client
import logging
from datetime import datetime

logger = logging.getLogger('recommend')


def get_cache_from_redis_hbase(temp, hbu):
    """
    进行用户频道推荐缓存结果的读取
    :param temp: 用户请求的参数
    :param hbu: hbase工具
    :return:
    """

    # - 首先从获取redis结果，进行判断(缓存拿)
    key = 'reco:{}:{}:art'.format(temp.user_id, temp.channel_id)
    res = cache_client.zrevrange(key, 0, temp.article_num - 1)
    # - 如果redis有，读取需要推荐的文章数量放回，并删除这些文章，并且放入推荐历史推荐结果中
    if res:
        cache_client.zrem(key, *res)
    else:
        # - 如果redis当中不存在，则从wait_recommend中读取
        # 删除redis这个键
        cache_client.delete(key)
        try:
            # 字符串编程列表
            hbase_cache = eval(hbu.get_table_row('wait_recommend',
                                                 'reco:{}'.format(temp.user_id).encode(),
                                                 'channel:{}'.format(temp.channel_id).encode()))

        except Exception as e:
            logger.warning("{} WARN read user_id:{} wait_recommend exception:{} not exist".format(
                datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, e))

            hbase_cache = []
        if not hbase_cache:
            #   - 如果wait_recommend中也没有，直接返回空，去进行一次召回读取、排序然后推荐
            return hbase_cache
        #   - wait_recommend有，从wait_recommend取出所有结果，定一个数量(如100篇)的文章存入redis
        if len(hbase_cache) > 100:
            logger.info(
                "{} INFO reduce user_id:{} channel:{} wait_recommend data".format(
                    datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))
            # 拿出固定数量（100）给redis
            cache_redis = hbase_cache[:100]
            # 放入redis缓存
            cache_client.zadd(key, dict(zip(cache_redis, range(len(cache_redis)))))
            # 剩下的放回wait hbase结果
            hbu.get_table_put('wait_recommend',
                              'reco:{}'.format(temp.user_id).encode(),
                              'channel:{}'.format(temp.channel_id).encode(),
                              str(hbase_cache[100:]).encode())
        else:
            logger.info(
                "{} INFO delete user_id:{} channel:{} wait_recommend data".format(
                    datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))
            # - wait_recommend不够一定数量，全部取出放入redis当中，直接推荐出去
            # 清空wait_recommend
            hbu.get_table_put('wait_recommend',
                              'reco:{}'.format(temp.user_id).encode(),
                              'channel:{}'.format(temp.channel_id).encode(),
                              str([]).encode())

            # 放入redis缓存
            cache_client.zadd(key, dict(zip(hbase_cache, range(len(hbase_cache)))))

        res = cache_client.zrevrange(key, 0, temp.article_num - 1)
        if res:
            cache_client.zrem(key, *res)

    # 进行执行PL，然后写入历史推荐结果
    # 进行类型转换
    res = list(map(int, res))
    logger.info("{} INFO get cache data and store user_id:{} channel:{} cache history data".format(
        datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))

    # 放入历史记录
    hbu.get_table_put('history_recommend',
                      'reco:his:{}'.format(temp.user_id).encode(),
                      'channel:{}'.format(temp.channel_id).encode(),
                      str(res).encode(),
                      timestamp=temp.time_stamp)

    return res
```

### 5.6.2 在推荐中心加入缓存逻辑

* from server import redis_cache

```python
# 1、获取缓存
res = redis_cache.get_reco_from_cache(temp, self.hbu)

# 如果没有，然后走一遍算法推荐 召回+排序，同时写入到hbase待推荐结果列表
 if not res:
     logger.info("{} INFO get user_id:{} channel:{} recall/sort data".
                 format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))

     res = self.user_reco_list(temp)
```

