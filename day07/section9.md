# 8.9 WDL模型导出

## 学习目标

- 目标
  - 无
- 应用
  - 无

### 8.9.1 线上预估

线上流量是模型效果的试金石。离线训练好的模型只有参与到线上真实流量预估，才能发挥其价值。在演化的过程中，我们开发了一套稳定可靠的线上预估体系，提高了模型迭代的效率。

**基于TF Serving的模型服务**

TF Serving是TensorFlow官方提供的一套用于在线实时预估的框架。它的突出优点是：和TensorFlow无缝链接，具有很好的扩展性。使用TF serving可以快速支持RNN、LSTM、GAN等多种网络结构，而不需要额外开发代码。这非常有利于我们模型快速实验和迭代。

### 8.9.2 SavedModel

* 目的：导出savedmodel的模型格式

TensorFlow的模型格式有很多种，针对不同场景可以使用不同的格式，只要符合规范的模型都可以轻易部署到在线服务或移动设备上，这里简单列举一下。

- Checkpoint： 用于保存模型的权重，主要用于模型训练过程中参数的备份和模型训练热启动。
- SavedModel：使用saved_model接口导出的模型文件，包含模型Graph和权限可直接用于上线，TensorFlowestimator和Keras模型推荐使用这种模型格式。

* 实现：
  * 1、指定指定serving模型的输入特征列类型，用于预测时候输入的列的类型指定
  * 2、指定模型的特征列输入函数以及example，用于预测的时候输入的整体数据格式
    * tf.feature_column.make_parse_example_spec(columns)
    * tf.estimator.export.build_parsing_serving_input_receiver_fn(feature_spec)

导出代码：

```python
# 定义导出模型的输入特征列
wide_columns = [tf.feature_column.categorical_column_with_identity('channel_id', num_buckets=25)]
deep_columns = [tf.feature_column.embedding_column(tf.feature_column.categorical_column_with_identity('channel_id', num_buckets=25), dimension=25),
                tf.feature_column.numeric_column('vector'),
                tf.feature_column.numeric_column('user_weigths'),
                tf.feature_column.numeric_column('article_weights')
                ]

columns = wide_columns + deep_columns
# 模型的特征列输入函数指定，按照example构造
feature_spec = tf.feature_column.make_parse_example_spec(columns)
serving_input_receiver_fn = tf.estimator.export.build_parsing_serving_input_receiver_fn(feature_spec)
estimator.export_savedmodel("./serving_model/wdl/", serving_input_receiver_fn)
```