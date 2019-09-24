# 7. 8 排序模型在线测试

### 学习目标

- 目标
  - 无
- 应用
  - 应用TensorFlow Serving apis完成在线模型的获取排序测试

### 7.8.1 排序模型在线预测添加

* 目的：编写tf serving客户端程序调用serving模型服务，进行在线预测测试
* 步骤：
  * 1、用户特征与文章特征合并
  * 2、serving服务端的example样本结构构造
  * 3、模型服务调用对接

**1、用户特征与文章特征合并**

```python
import tensorflow as tf
from grpc.beta import implementations
from tensorflow_serving.apis import prediction_service_pb2_grpc
from tensorflow_serving.apis import predict_pb2
from tensorflow_serving.apis import classification_pb2
import os
import sys
import grpc
from server.utils import HBaseUtils
from server import pool
import numpy
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
print(BASE_DIR)
sys.path.insert(0, os.path.join(BASE_DIR))


def wdl_sort_service():
    """
    wide&deep进行排序预测
    :param reco_set:
    :param temp:
    :param hbu:
    :return:
    """
    hbu = HBaseUtils(pool)
    # 排序
    # 1、读取用户特征中心特征
    try:
        user_feature = eval(hbu.get_table_row('ctr_feature_user',
                                              '{}'.format(1115629498121846784).encode(),
                                              'channel:{}'.format(18).encode()))
        # logger.info("{} INFO get user user_id:{} channel:{} profile data".format(
        #     datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))
    except Exception as e:
        user_feature = []
    if user_feature:
        # 2、读取文章特征中心特征
        result = []

        # examples
        examples = []
        for article_id in [17749, 17748, 44371, 44368]:
            try:
                article_feature = eval(hbu.get_table_row('ctr_feature_article',
                                                         '{}'.format(article_id).encode(),
                                                         'article:{}'.format(article_id).encode()))
            except Exception as e:

                article_feature = [0.0] * 111
```

**2、serving服务端的example样本结构构造**

```python
channel_id = int(article_feature[0])
# 求出后面若干向量的平均值
vector = np.mean(article_feature[11:])
# 第三个用户权重特征
user_feature = np.mean(user_feature)
# 第四个文章权重特征
article_feature = np.mean(article_feature[1:11])

# 组建example
example = tf.train.Example(features=tf.train.Features(feature={
    "channel_id": tf.train.Feature(int64_list=tf.train.Int64List(value=[channel_id])),
    "vector": tf.train.Feature(float_list=tf.train.FloatList(value=[vector])),
    'user_weigths': tf.train.Feature(float_list=tf.train.FloatList(value=[user_feature])),
    'article_weights': tf.train.Feature(float_list=tf.train.FloatList(value=[article_feature])),
}))

examples.append(example)
```

**3、模型服务调用对接**

* 导入包：
  * from tensorflow_serving.apis import prediction_service_pb2_grpc：与grpc建立stub
    * PredictionServiceStub：建立stub，负责将调用的接口、方法和参数约定
      * Classify:发送请求, 设置延时时间 secs timeout
  * from tensorflow_serving.apis import classification_pb2：调用服务，提供请求参数

```python
with grpc.insecure_channel('127.0.0.1:8500') as channel:
    stub = prediction_service_pb2_grpc.PredictionServiceStub(channel)

    # 获取测试数据集，并转换成 Example 实例
    # 准备 RPC 请求，指定模型名称。
    request = classification_pb2.ClassificationRequest()
    request.model_spec.name = 'wdl'
    request.input.example_list.examples.extend(examples)

    # 获取结果
    response = stub.Classify(request, 10.0)
    print(response)
```

完整代码：

```python
import tensorflow as tf
from grpc.beta import implementations
from tensorflow_serving.apis import prediction_service_pb2_grpc
from tensorflow_serving.apis import predict_pb2
from tensorflow_serving.apis import classification_pb2
import os
import sys
import grpc
from server.utils import HBaseUtils
from server import pool
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
print(BASE_DIR)
sys.path.insert(0, os.path.join(BASE_DIR))


def wdl_sort_service():
    """
    wide&deep进行排序预测
    :param reco_set:
    :param temp:
    :param hbu:
    :return:
    """
    hbu = HBaseUtils(pool)
    # 排序
    # 1、读取用户特征中心特征
    try:
        user_feature = eval(hbu.get_table_row('ctr_feature_user',
                                              '{}'.format(1115629498121846784).encode(),
                                              'channel:{}'.format(18).encode()))
        # logger.info("{} INFO get user user_id:{} channel:{} profile data".format(
        #     datetime.now().strftime('%Y-%m-%d %H:%M:%S'), temp.user_id, temp.channel_id))
    except Exception as e:
        user_feature = []
    if user_feature:
        # 2、读取文章特征中心特征
        result = []

        # examples
        examples = []
        for article_id in [17749, 17748, 44371, 44368]:
            try:
                article_feature = eval(hbu.get_table_row('ctr_feature_article',
                                                         '{}'.format(article_id).encode(),
                                                         'article:{}'.format(article_id).encode()))
            except Exception as e:

                article_feature = [0.0] * 111

            channel_id = int(article_feature[0])
            # 求出后面若干向量的平均值
            vector = np.mean(article_feature[11:])
            # 第三个用户权重特征
            user_feature = np.mean(user_feature)
            # 第四个文章权重特征
            article_feature = np.mean(article_feature[1:11])

            # 组建example
            example = tf.train.Example(features=tf.train.Features(feature={
                "channel_id": tf.train.Feature(int64_list=tf.train.Int64List(value=[channel_id])),
                "vector": tf.train.Feature(float_list=tf.train.FloatList(value=[vector])),
                'user_weigths': tf.train.Feature(float_list=tf.train.FloatList(value=[user_feature])),
                'article_weights': tf.train.Feature(float_list=tf.train.FloatList(value=[article_feature])),
            }))

            examples.append(example)

        with grpc.insecure_channel('127.0.0.1:8500') as channel:
            stub = prediction_service_pb2_grpc.PredictionServiceStub(channel)

            # 获取测试数据集，并转换成 Example 实例
            # 准备 RPC 请求，指定模型名称。
            request = classification_pb2.ClassificationRequest()
            request.model_spec.name = 'wdl'
            request.input.example_list.examples.extend(examples)

            # 获取结果
            response = stub.Classify(request, 10.0)
            print(response)

    return None


if __name__ == '__main__':
    wdl_sort_service()
```