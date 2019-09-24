# 2.8 文章相似度增量更新

## 学习目标

- 目标
  - 知道文章向量计算方式
  - 了解Word2Vec模型原理
  - 知道文章相似度计算方式
- 应用
  - 应用Spark完成文章相似度计算

### 2.8.1 增量更新需求

每天、每小时都会有大量的新文章过来，当后端审核通过一篇文章之后，我们推荐给用户后，一旦发生点击行为了，就会有相应相似文章计算需求。没有选择去在线计算中处理，新文章来了之后立即推荐召回给用用户，这样增加新文章推荐曝光的比例，达到一定速度的曝光。在线需要用用到离线仓库中部分数据，所以速度会受影响。

* 设计：
  * 按照之前增量更新文章画像那样的频率，按小时更新增量文章的相似文章
* 如何更新：
  * **批量新文章，需要与历史数据进行相似度计算**

### 2.8.2 增量更新文章向量与相似度

* 目标：对每次新增的文章，计算完画像后，计算向量，在进行与历史文章相似度计算
* 步骤:
  * 1、新文章数据，按照频道去计算文章所在频道的相似度
  * 2、求出新文章向量，保存
  * 3、BucketedRandomProjectionLSH计算相似度

完整代码：

```python
    def compute_article_similar(self, articleProfile):
        """
        计算增量文章与历史文章的相似度 word2vec
        :return:
        """
        # 得到要更新的新文章通道类别(不采用)
        # all_channel = set(articleProfile.rdd.map(lambda x: x.channel_id).collect())
        def avg(row):
            x = 0
            for v in row.vectors:
                x += v
            #  将平均向量作为article的向量
            return row.article_id, row.channel_id, x / len(row.vectors)

        for channel_id, channel_name in CHANNEL_INFO.items():

            profile = articleProfile.filter('channel_id = {}'.format(channel_id))
            wv_model = Word2VecModel.load(
                "hdfs://hadoop-master:9000/headlines/models/channel_%d_%s.word2vec" % (channel_id, channel_name))
            vectors = wv_model.getVectors()

            # 计算向量
            profile.registerTempTable("incremental")
            articleKeywordsWeights = ua.spark.sql(
                "select article_id, channel_id, keyword, weight from incremental LATERAL VIEW explode(keywords) AS keyword, weight where channel_id=%d" % channel_id)

            articleKeywordsWeightsAndVectors = articleKeywordsWeights.join(vectors,
                                                            vectors.word == articleKeywordsWeights.keyword, "inner")
            articleKeywordVectors = articleKeywordsWeightsAndVectors.rdd.map(
                lambda r: (r.article_id, r.channel_id, r.keyword, r.weight * r.vector)).toDF(
                ["article_id", "channel_id", "keyword", "weightingVector"])

            articleKeywordVectors.registerTempTable("tempTable")
            articleVector = self.spark.sql(
                "select article_id, min(channel_id) channel_id, collect_set(weightingVector) vectors from tempTable group by article_id").rdd.map(
                avg).toDF(["article_id", "channel_id", "articleVector"])

            # 写入数据库
            def toArray(row):
                return row.article_id, row.channel_id, [float(i) for i in row.articleVector.toArray()]
            articleVector = articleVector.rdd.map(toArray).toDF(['article_id', 'channel_id', 'articleVector'])
            articleVector.write.insertInto("article_vector")

            import gc
            del wv_model
            del vectors
            del articleKeywordsWeights
            del articleKeywordsWeightsAndVectors
            del articleKeywordVectors
            gc.collect()

            # 得到历史数据, 转换成固定格式使用LSH进行求相似
            train = self.spark.sql("select * from article_vector where channel_id=%d" % channel_id)

            def _array_to_vector(row):
                return row.article_id, Vectors.dense(row.articleVector)
            train = train.rdd.map(_array_to_vector).toDF(['article_id', 'articleVector'])
            test = articleVector.rdd.map(_array_to_vector).toDF(['article_id', 'articleVector'])

            brp = BucketedRandomProjectionLSH(inputCol='articleVector', outputCol='hashes', seed=12345,
                                              bucketLength=1.0)
            model = brp.fit(train)
            similar = model.approxSimilarityJoin(test, train, 2.0, distCol='EuclideanDistance')

            def save_hbase(partition):
                import happybase
                for row in partition:
                    pool = happybase.ConnectionPool(size=3, host='hadoop-master')
                    # article_similar article_id similar:article_id sim
                    with pool.connection() as conn:
                        table = connection.table("article_similar")
                        for row in partition:
                            if row.datasetA.article_id == row.datasetB.article_id:
                                pass
                            else:
                                table.put(str(row.datasetA.article_id).encode(),
                                          {b"similar:%d" % row.datasetB.article_id: b"%0.4f" % row.EuclideanDistance})
                        conn.close()
            similar.foreachPartition(save_hbase)
```

添加函数到update_article.py文件中，修改update更新代码

```python
ua = UpdateArticle()
sentence_df = ua.merge_article_data()
if sentence_df.rdd.collect():
    rank, idf = ua.generate_article_label(sentence_df)
    articleProfile = ua.get_article_profile(rank, idf)
    ua.compute_article_similar(articleProfile)
```

