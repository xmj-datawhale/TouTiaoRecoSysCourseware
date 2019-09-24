# 2.5 离线增量文章画像计算

## 学习目标

- 目标
  - 了解增量更新代码过程
- 应用
  - 无

### 2.5.1 离线文章画像更新需求

文章画像，就是给每篇文章定义一些词。

* 关键词：TEXTRANK + IDF共同的词
* 主题词：TEXTRANK + ITFDF共同的词

- 更新文章时间：

1、toutiao 数据库中，news_article_content 与news_article_basic—>更新到article数据库中article_data表，方便操作

第一次：所有更新，后面增量每天的数据更新26日：1：00~2：00，2：00~3：00，左闭右开,一个小时更新一次

2、刚才新更新的文章，通过已有的idf计算出tfidf值以及hive 的textrank_keywords_values

3、更新hive的**article_profile**

### 2.5.2 定时更新文章设置

* 目的：通过Supervisor管理Apscheduler定时运行更新程序
* 步骤：
  * 1、更新程序代码整理，并测试运行
  * 2、Apscheduler设置定时运行时间，并启动日志添加
  * 3、Supervisor进程管理

#### 2.5.2.1 更新程序代码整理，并测试运行

注意在Pycharm中运行要设置环境：

```python
PYTHONUNBUFFERED=1
JAVA_HOME=/root/bigdata/jdk
SPARK_HOME=/root/bigdata/spark
HADOOP_HOME=/root/bigdata/hadoop
PYSPARK_PYTHON=/root/anaconda3/envs/reco_sys/bin/python
PYSPARK_DRIVER_PYTHON=/root/anaconda3/envs/reco_sys/bin/python
```

![](../images/更新环境设置.png)

```python
import os
import sys
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
sys.path.insert(0, os.path.join(BASE_DIR))
from offline import SparkSessionBase
from datetime import datetime
from datetime import timedelta
import pyspark.sql.functions as F
import pyspark
import gc

class UpdateArticle(SparkSessionBase):
    """
    更新文章画像
    """
    SPARK_APP_NAME = "updateArticle"
    ENABLE_HIVE_SUPPORT = True

    SPARK_EXECUTOR_MEMORY = "7g"

    def __init__(self):
        self.spark = self._create_spark_session()

        self.cv_path = "hdfs://hadoop-master:9000/headlines/models/countVectorizerOfArticleWords.model"
        self.idf_path = "hdfs://hadoop-master:9000/headlines/models/IDFOfArticleWords.model"

    def get_cv_model(self):
        # 词语与词频统计
        from pyspark.ml.feature import CountVectorizerModel
        cv_model = CountVectorizerModel.load(self.cv_path)
        return cv_model

    def get_idf_model(self):
        from pyspark.ml.feature import IDFModel
        idf_model = IDFModel.load(self.idf_path)
        return idf_model

    @staticmethod
    def compute_keywords_tfidf_topk(words_df, cv_model, idf_model):
        """保存tfidf值高的20个关键词
        :param spark:
        :param words_df:
        :return:
        """
        cv_result = cv_model.transform(words_df)
        tfidf_result = idf_model.transform(cv_result)
        # print("transform compelete")

        # 取TOP-N的TFIDF值高的结果
        def func(partition):
            TOPK = 20
            for row in partition:
                _ = list(zip(row.idfFeatures.indices, row.idfFeatures.values))
                _ = sorted(_, key=lambda x: x[1], reverse=True)
                result = _[:TOPK]
                #         words_index = [int(i[0]) for i in result]
                #         yield row.article_id, row.channel_id, words_index

                for word_index, tfidf in result:
                    yield row.article_id, row.channel_id, int(word_index), round(float(tfidf), 4)

        _keywordsByTFIDF = tfidf_result.rdd.mapPartitions(func).toDF(["article_id", "channel_id", "index", "tfidf"])

        return _keywordsByTFIDF

    def merge_article_data(self):
        """
        合并业务中增量更新的文章数据
        :return:
        """
        # 获取文章相关数据, 指定过去一个小时整点到整点的更新数据
        # 如：26日：1：00~2：00，2：00~3：00，左闭右开
        self.spark.sql("use toutiao")
        _yester = datetime.today().replace(minute=0, second=0, microsecond=0)
        start = datetime.strftime(_yester + timedelta(days=0, hours=-1, minutes=0), "%Y-%m-%d %H:%M:%S")
        end = datetime.strftime(_yester, "%Y-%m-%d %H:%M:%S")

        # 合并后保留：article_id、channel_id、channel_name、title、content
        # +----------+----------+--------------------+--------------------+
        # | article_id | channel_id | title | content |
        # +----------+----------+--------------------+--------------------+
        # | 141462 | 3 | test - 20190316 - 115123 | 今天天气不错，心情很美丽！！！ |
        basic_content = self.spark.sql(
            "select a.article_id, a.channel_id, a.title, b.content from news_article_basic a "
            "inner join news_article_content b on a.article_id=b.article_id where a.review_time >= '{}' "
            "and a.review_time < '{}' and a.status = 2".format(start, end))
        # 增加channel的名字，后面会使用
        basic_content.registerTempTable("temparticle")
        channel_basic_content = self.spark.sql(
            "select t.*, n.channel_name from temparticle t left join news_channel n on t.channel_id=n.channel_id")

        # 利用concat_ws方法，将多列数据合并为一个长文本内容（频道，标题以及内容合并）
        self.spark.sql("use article")
        sentence_df = channel_basic_content.select("article_id", "channel_id", "channel_name", "title", "content", \
                                                   F.concat_ws(
                                                       ",",
                                                       channel_basic_content.channel_name,
                                                       channel_basic_content.title,
                                                       channel_basic_content.content
                                                   ).alias("sentence")
                                                   )
        del basic_content
        del channel_basic_content
        gc.collect()

        sentence_df.write.insertInto("article_data")
        return sentence_df

    def generate_article_label(self, sentence_df):
        """
        生成文章标签  tfidf, textrank
        :param sentence_df: 增量的文章内容
        :return:
        """
        # 进行分词
        words_df = sentence_df.rdd.mapPartitions(segmentation).toDF(["article_id", "channel_id", "words"])
        cv_model = self.get_cv_model()
        idf_model = self.get_idf_model()

        # 1、保存所有的词的idf的值，利用idf中的词的标签索引
        # 工具与业务隔离
        _keywordsByTFIDF = UpdateArticle.compute_keywords_tfidf_topk(words_df, cv_model, idf_model)

        keywordsIndex = self.spark.sql("select keyword, index idx from idf_keywords_values")

        keywordsByTFIDF = _keywordsByTFIDF.join(keywordsIndex, keywordsIndex.idx == _keywordsByTFIDF.index).select(
            ["article_id", "channel_id", "keyword", "tfidf"])

        keywordsByTFIDF.write.insertInto("tfidf_keywords_values")

        del cv_model
        del idf_model
        del words_df
        del _keywordsByTFIDF
        gc.collect()

        # 计算textrank
        textrank_keywords_df = sentence_df.rdd.mapPartitions(textrank).toDF(
            ["article_id", "channel_id", "keyword", "textrank"])
        textrank_keywords_df.write.insertInto("textrank_keywords_values")

        return textrank_keywords_df, keywordsIndex

    def get_article_profile(self, textrank, keywordsIndex):
        """
        文章画像主题词建立
        :param idf: 所有词的idf值
        :param textrank: 每个文章的textrank值
        :return: 返回建立号增量文章画像
        """
        keywordsIndex = keywordsIndex.withColumnRenamed("keyword", "keyword1")
        result = textrank.join(keywordsIndex, textrank.keyword == keywordsIndex.keyword1)

        # 1、关键词（词，权重）
        # 计算关键词权重
        _articleKeywordsWeights = result.withColumn("weights", result.textrank * result.idf).select(
            ["article_id", "channel_id", "keyword", "weights"])

        # 合并关键词权重到字典
        _articleKeywordsWeights.registerTempTable("temptable")
        articleKeywordsWeights = self.spark.sql(
            "select article_id, min(channel_id) channel_id, collect_list(keyword) keyword_list, collect_list(weights) weights_list from temptable group by article_id")
        def _func(row):
            return row.article_id, row.channel_id, dict(zip(row.keyword_list, row.weights_list))
        articleKeywords = articleKeywordsWeights.rdd.map(_func).toDF(["article_id", "channel_id", "keywords"])

        # 2、主题词
        # 将tfidf和textrank共现的词作为主题词
        topic_sql = """
                select t.article_id article_id2, collect_set(t.keyword) topics from tfidf_keywords_values t
                inner join 
                textrank_keywords_values r
                where t.keyword=r.keyword
                group by article_id2
                """
        articleTopics = self.spark.sql(topic_sql)

        # 3、将主题词表和关键词表进行合并，插入表
        articleProfile = articleKeywords.join(articleTopics,
                                              articleKeywords.article_id == articleTopics.article_id2).select(
            ["article_id", "channel_id", "keywords", "topics"])
        articleProfile.write.insertInto("article_profile")

        del keywordsIndex
        del _articleKeywordsWeights
        del articleKeywords
        del articleTopics
        gc.collect()

        return articleProfile


if __name__ == '__main__':
    ua = UpdateArticle()
    sentence_df = ua.merge_article_data()
    if sentence_df.rdd.collect():
        rank, idf = ua.generate_article_label(sentence_df)
        articleProfile = ua.get_article_profile(rank, idf)
```

### 2.5.3 增量更新文章TFIDF与TextRank(作为测试代码，不往HIVE中存储)

在jupyter notebook中实现计算过程

- 目的：能够定时增量的更新新发表的文章
- 步骤：
  - 合并新文章数据
  - 利用现有CV和IDF模型计算新文章TFIDF存储，以及TextRank保存
  - 利用新文章数据的
- 导入包

```python
import os
# 配置spark driver和pyspark运行时，所使用的python解释器路径
PYSPARK_PYTHON = "/miniconda2/envs/reco_sys/bin/python"
# 当存在多个版本时，不指定很可能会导致出错
os.environ["PYSPARK_PYTHON"] = PYSPARK_PYTHON
os.environ["PYSPARK_DRIVER_PYTHON"] = PYSPARK_PYTHON
import sys
BASE_DIR = os.path.dirname(os.getcwd())
sys.path.insert(0, os.path.join(BASE_DIR))
from datetime import datetime
from datetime import timedelta
import pyspark.sql.functions as F
from offline import SparkSessionBase
import pyspark
import gc
```

#### 2.5.3.1 合并新文章数据

```python
class UpdateArticle(SparkSessionBase):
    """
    更新文章画像
    """
    SPARK_APP_NAME = "updateArticle"
    ENABLE_HIVE_SUPPORT = True

    SPARK_EXECUTOR_MEMORY = "7g"

    def __init__(self):
        self.spark = self._create_spark_session()
```

- 增量合并文章

可以根据自己的业务制定符合现有阶段的更新计划，比如按照天，小时更新, 

```python 
ua.spark.sql("use toutiao")
_yester = datetime.today().replace(minute=0, second=0, microsecond=0)
start = datetime.strftime(_yester + timedelta(days=0, hours=-1, minutes=0), "%Y-%m-%d %H:%M:%S")
end = datetime.strftime(_yester, "%Y-%m-%d %H:%M:%S")
```

选取指定时间段的新文章(测试时候，为了有数据出现，可以将偏移多一些天数，如days=-50)

注：确保news_article_basic与news_article_content是一致的。

```python
# 合并后保留：article_id、channel_id、channel_name、title、content
# select * from news_article_basic where review_time > "2019-03-05";
# +----------+----------+--------------------+--------------------+
# | article_id | channel_id | title | content |
# +----------+----------+--------------------+--------------------+
# | 141462 | 3 | test - 20190316 - 115123 | 今天天气不错，心情很美丽！！！ |
basic_content = ua.spark.sql(
  "select a.article_id, a.channel_id, a.title, b.content from news_article_basic a "
  "inner join news_article_content b on a.article_id=b.article_id where a.review_time >= '{}' "
  "and a.review_time < '{}' and a.status = 2".format(start, end))
# 增加channel的名字，后面会使用
basic_content.registerTempTable("temparticle")
channel_basic_content = ua.spark.sql(
  "select t.*, n.channel_name from temparticle t left join news_channel n on t.channel_id=n.channel_id")

# 利用concat_ws方法，将多列数据合并为一个长文本内容（频道，标题以及内容合并）
ua.spark.sql("use article")
sentence_df = channel_basic_content.select("article_id", "channel_id", "channel_name", "title", "content", \
                                           F.concat_ws(
                                             ",",
                                             channel_basic_content.channel_name,
                                             channel_basic_content.title,
                                             channel_basic_content.content
                                           ).alias("sentence")
                                          )
del basic_content
del channel_basic_content
gc.collect()

# sentence_df.write.insertInto("article_data")
```

#### 2.5.3.2 更新TFIDF

- 问题：计算出TFIDF，TF文档词频，IDF 逆文档频率（文档数量、某词出现的文档数量）已有N个文章中词的IDF会随着新增文章而动态变化，就会涉及TFIDF的增量计算。
  - 解决办法可以在**固定时间定时对所有文章数据进行全部计算CV和IDF的模型结果，替换模型即可**

对新文章分词,读取模型

```python
# 进行分词前面计算出的sentence_df
words_df = sentence_df.rdd.mapPartitions(segmentation).toDF(["article_id", "channel_id", "words"])
cv_model = get_cv_model()
idf_model = get_idf_model()
```

定义两个读取函数

```python
def get_cv_model(self):
		# 词语与词频统计
		from pyspark.ml.feature import CountVectorizerModel
    cv_model = CountVectorizerModel.load(cv_path)
    return cv_model

def get_idf_model(self):
		from pyspark.ml.feature import IDFModel
    idf_model = IDFModel.load(idf_path)
    return idf_model
```

```python
    def compute_keywords_tfidf_topk(words_df, cv_model, idf_model):
        """保存tfidf值高的20个关键词
        :param spark:
        :param words_df:
        :return:
        """
        cv_result = cv_model.transform(words_df)
        tfidf_result = idf_model.transform(cv_result)
        # print("transform compelete")

        # 取TOP-N的TFIDF值高的结果
        def func(partition):
            TOPK = 20
            for row in partition:
                _ = list(zip(row.idfFeatures.indices, row.idfFeatures.values))
                _ = sorted(_, key=lambda x: x[1], reverse=True)
                result = _[:TOPK]
                for word_index, tfidf in result:
                    yield row.article_id, row.channel_id, int(word_index), round(float(tfidf), 4)

        _keywordsByTFIDF = tfidf_result.rdd.mapPartitions(func).toDF(["article_id", "channel_id", "index", "tfidf"])

        return _keywordsByTFIDF
# 1、保存所有的词的idf的值，利用idf中的词的标签索引
# 工具与业务隔离
_keywordsByTFIDF = compute_keywords_tfidf_topk(words_df, cv_model, idf_model)

keywordsIndex = ua.spark.sql("select keyword, index idx from idf_keywords_values")

keywordsByTFIDF = _keywordsByTFIDF.join(keywordsIndex, keywordsIndex.idx == _keywordsByTFIDF.index).select(
  ["article_id", "channel_id", "keyword", "tfidf"])

# keywordsByTFIDF.write.insertInto("tfidf_keywords_values")

del cv_model
del idf_model
del words_df
del _keywordsByTFIDF
gc.collect()

# 计算textrank
textrank_keywords_df = sentence_df.rdd.mapPartitions(textrank).toDF(
  ["article_id", "channel_id", "keyword", "textrank"])
# textrank_keywords_df.write.insertInto("textrank_keywords_values")
```

前面这些得到textrank_keywords_df，接下来往后进行文章的画像更新

### 2.5.3.3 增量更新文章画像结果

对于新文章进行计算画像

- 步骤：
  - 1、加载IDF，保留关键词以及权重计算(TextRank * IDF)
  - 2、合并关键词权重到字典结果
  - 3、将tfidf和textrank共现的词作为主题词
  - 4、将主题词表和关键词表进行合并，插入表

**加载IDF，保留关键词以及权重计算(TextRank * IDF)**

```python
idf = ua.spark.sql("select * from idf_keywords_values")
idf = idf.withColumnRenamed("keyword", "keyword1")
result = textrank_keywords_df.join(idf,textrank_keywords_df.keyword==idf.keyword1)
keywords_res = result.withColumn("weights", result.textrank * result.idf).select(["article_id", "channel_id", "keyword", "weights"])
```

**合并关键词权重到字典结果**

```python
keywords_res.registerTempTable("temptable")
merge_keywords = ua.spark.sql("select article_id, min(channel_id) channel_id, collect_list(keyword) keywords, collect_list(weights) weights from temptable group by article_id")

# 合并关键词权重合并成字典
def _func(row):
    return row.article_id, row.channel_id, dict(zip(row.keywords, row.weights))

keywords_info = merge_keywords.rdd.map(_func).toDF(["article_id", "channel_id", "keywords"])
```

**将tfidf和textrank共现的词作为主题词**

```python
topic_sql = """
                select t.article_id article_id2, collect_set(t.keyword) topics from tfidf_keywords_values t
                inner join 
                textrank_keywords_values r
                where t.keyword=r.keyword
                group by article_id2
                """
articleTopics = ua.spark.sql(topic_sql)
```

**将主题词表和关键词表进行合并。**

```python
article_profile = keywords_info.join(article_topics, keywords_info.article_id==article_topics.article_id2).select(["article_id", "channel_id", "keywords", "topics"])

# articleProfile.write.insertInto("article_profile")
```

