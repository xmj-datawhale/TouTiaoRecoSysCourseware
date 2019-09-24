# 3.5 离线用户基于内容召回集

## 学习目标

- 目标
  - 知道离线内容召回的概念
  - 知道如何进行内容召回计算存储规则
- 应用
  - 应用spark完成离线用户基于内容的协同过滤推荐

### 3.5.1 基于内容召回实现

**基于Item协同过滤与基于内容协同过滤区别:**

* 基于物品的协同过滤: 用户喜欢的东西，然后从剩下的物品中找到和他历史兴趣近似的物品推荐给他，核心是要通过两个物品被同时喜欢的用户数等决定。

```python
假设有A、B、C、D4个用户:
用户A买了a、b、d三个商品；
用户B买了a、c；
用户C买了b、e；
用户D买了c、d、e

推荐给A:我们就需要比较c和e哪个商品与a和b和d的相似度总和最大？

例如我们现在先比较c和a的相似度（对c和b、c和d进行相同处理），就是（同时拥有c和a的用户数/喜欢a的用户数），同理可以用来考察e和a
```

* 基于内容的过滤：给用户推荐和他们之前喜欢的物品在内容上相似的其他物品。核心任务就是计算物品的内容相似度。
  * 给物品内容建模，如Word2vec模型得到词向量或者文章向量

* 目的：实现定时离线更新用户的内容召回集合
* 步骤：
  * 1、过滤用户点击的文章
  * 2、用户每次操作文章进行相似获取并进行推荐

过滤用户点击的文章


```mysql
ur.spark.sql("use profile")
user_article_basic = ur.spark.sql("select * from user_article_basic")
user_article_basic = user_article_basic.filter('clicked=True')
```

用户每次操作文章进行相似获取并进行推荐

```python
# 基于内容相似召回（画像召回）
ur.spark.sql("use profile")
user_article_basic = self.spark.sql("select * from user_article_basic")
user_article_basic = user_article_basic.filter("clicked=True")

def save_content_filter_history_to__recall(partition):
    """计算每个用户的每个操作文章的相似文章，过滤之后，写入content召回表当中（支持不同时间戳版本）
    """
    import happybase
    pool = happybase.ConnectionPool(size=10, host='hadoop-master')

    # 进行为相似文章获取
    with pool.connection() as conn:

        # key:   article_id,    column:  similar:article_id
        similar_table = conn.table('article_similar')
        # 循环partition
        for row in partition:
            # 获取相似文章结果表
            similar_article = similar_table.row(str(row.article_id).encode(),
                                                columns=[b'similar'])
            # 相似文章相似度排序过滤，召回不需要太大的数据， 百个，千
            _srt = sorted(similar_article.items(), key=lambda item: item[1], reverse=True)
            if _srt:
                # 每次行为推荐10篇文章
                reco_article = [int(i[0].split(b':')[1]) for i in _srt][:10]

                # 获取历史看过的该频道文章
                history_table = conn.table('history_recall')
                # 多个版本
                data = history_table.cells('reco:his:{}'.format(row.user_id).encode(),
                                           'channel:{}'.format(row.channel_id).encode())

                history = []
                if len(data) >= 2:
                    for l in data[:-1]:
                        history.extend(eval(l))
                else:
                    history = []

                # 过滤reco_article与history
                reco_res = list(set(reco_article) - set(history))

                # 进行推荐，放入基于内容的召回表当中以及历史看过的文章表当中
                if reco_res:
                    # content_table = conn.table('cb_content_recall')
                    content_table = conn.table('cb_recall')
                    content_table.put("recall:user:{}".format(row.user_id).encode(),
                                      {'content:{}'.format(row.channel_id).encode(): str(reco_res).encode()})

                    # 放入历史推荐过文章
                    history_table.put("reco:his:{}".format(row.user_id).encode(),
                                      {'channel:{}'.format(row.channel_id).encode(): str(reco_res).encode()})

        conn.close()

user_article_basic.foreachPartition(save_content_filter_history_to__recall)
```

* 1、获取用户点击的某文章相似文章结果并排序过滤
  * 相似结果取出TOPK：根据实际场景选择大小，10或20

```python
# 循环partition
for row in partition:
    # 获取相似文章结果表
    similar_article = similar_table.row(str(row.article_id).encode(),
                                        columns=[b'similar'])
    # 相似文章相似度排序过滤，召回不需要太大的数据， 百个，千
    _srt = sorted(similar_article.items(), key=lambda item: item[1], reverse=True)
    if _srt:
        # 每次行为推荐若干篇文章
        reco_article = [int(i[0].split(b':')[1]) for i in _srt][:10]
```

* 2、过滤历史召回的所有文章(所有的召回类型)

```python
# 获取历史看过的该频道文章
history_table = conn.table('history_recall')
# 多个版本
data = history_table.cells('reco:his:{}'.format(row.user_id).encode(),
                           'channel:{}'.format(row.channel_id).encode())

history = []
if len(data) >= 2:
    for l in data[:-1]:
        history.extend(eval(l))
else:
    history = []

# 过滤reco_article与history
reco_res = list(set(reco_article) - set(history))
```

* 3、对结果进行存储，历史推荐存储

```python
# 进行推荐，放入基于内容的召回表当中以及历史看过的文章表当中
if reco_res:
    # content_table = conn.table('cb_content_recall')
    content_table = conn.table('cb_recall')
    content_table.put("recall:user:{}".format(row.user_id).encode(),
                      {'content:{}'.format(row.channel_id).encode(): str(reco_res).encode()})

    # 放入历史推荐过文章
    history_table.put("reco:his:{}".format(row.user_id).encode(),
                      {'channel:{}'.format(row.channel_id).encode(): str(reco_res).encode()})
```

