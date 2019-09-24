# 7.4 TFRecords与黑马训练数据存储

## 学习目标

- 目标
  - 说明Example的结构
- 应用
  - 应用TF保存Spark构建的样本到TFRecords文件

## 7.4.1 训练样本准备流程

![](../images/实时排序逻辑.png)

## 2.8.2 什么是TFRecords文件

TFRecords其实是一种二进制文件，虽然它不如其他格式好理解，但是它能更好的利用内存，更方便复制和移动，**并且不需要单独的标签文件**。

TFRecords文件包含了`tf.train.Example` 协议内存块(protocol buffer)(协议内存块包含了字段 `Features`)。可以获取你的数据， 将数据填入到`Example`协议内存块(protocol buffer)，将协议内存块序列化为一个字符串， 并且通过`tf.python_io.TFRecordWriter` 写入到TFRecords文件。

- 文件格式 *.tfrecords 

## 2.8.3 Example结构解析

`tf.train.Example` 协议内存块(protocol buffer)(协议内存块包含了字段 `Features`)，`Features`包含了一个`Feature`字段，`Features`中包含要写入的数据、并指明数据类型。这是一个样本的结构，批数据需要循环存入这样的结构

```python
 example = tf.train.Example(features=tf.train.Features(feature={
                "features": tf.train.Feature(bytes_list=tf.train.BytesList(value=[features])),
                "label": tf.train.Feature(int64_list=tf.train.Int64List(value=[label])),
            }))
```

- tf.train.Example(**features**=None)
  - 写入tfrecords文件
  - features:tf.train.Features类型的特征实例
  - return：example格式协议块
- tf.train.**Features**(**feature**=None)
  - 构建每个样本的信息键值对
  - feature:字典数据,key为要保存的名字
  - value为tf.train.Feature实例
  - return:Features类型
- tf.train.**Feature**(options)
  - options：例如
    - bytes_list=tf.train. BytesList(value=[Bytes])
    - int64_list=tf.train. Int64List(value=[Value])
  - 支持存入的类型如下
  - tf.train.Int64List(value=[Value])
  - tf.train.BytesList(value=[Bytes]) 
  - tf.train.FloatList(value=[value]) 

> 这种结构是不是很好的解决了**数据和标签(训练的类别标签)或者其他属性数据存储在同一个文件中** 

使用步骤：

**第一步，生成TFRecord Writer**

```python3
writer = tf.python_io.TFRecordWriter(path, options=None)
```

**path：**TFRecord文件的存放路径；

**option：**TFRecordOptions对象，定义TFRecord文件保存的压缩格式；

**第二步，tf.train.Feature生成协议信息**

一个协议信息特征（这里翻译可能不准确）是将原始数据编码成特定的格式，内层feature是一个字典值，它是将某个类型列表编码成特定的feature格式，而该字典键用于读取TFRecords文件时索引得到不同的数据，某个类型列表可能包含零个或多个值，**列表类型一般有BytesList, FloatList, Int64List**

```python3
tf.train.BytesList(value=[value]) # value转化为字符串（二进制）列表
tf.train.FloatList(value=[value]) # value转化为浮点型列表
tf.train.Int64List(value=[value]) # value转化为整型列表
```

其中，value是你要保存的数据。内层feature编码方式：

```python3
feature_internal = {
"width":tf.train.Feature(int64_list=tf.train.Int64List(value=[width])),
"weights":tf.train.Feature(float_list=tf.train.FloatList(value=[weights])),
"image_raw":tf.train.Feature(bytes_list=tf.train.BytesList(value=[image_raw]))
}
```

外层features再将内层字典编码：

```text
features_extern = tf.train.Features(feature_internal)
```

* tf.train.Feature这个接口可以编码封装列表类型和字典类型，内层用的是tf.train.Feature
* 外层使用tf.train.Features

**第三步，使用tf.train.Example将features编码数据封装成特定的PB协议格式**

```text
example = tf.train.Example(features_extern)
```

**第四步，将example数据系列化为字符串**

```text
example_str = example.SerializeToString()
```

**第五步，将系列化为字符串的example数据写入协议缓冲区**

```text
writer.write(example_str)
```

## 2.8.4 案例：CIFAR10数据存入TFRecords文件

### 2.8.4.1 分析

![](../images/离线样本构造.png)

- 构造存储实例，tf.python_io.TFRecordWriter(path)
  - 写入tfrecords文件
  - path: TFRecords文件的路径
  - return：写文件
  - method
  - write(record):向文件中写入一个example
  - close():关闭文件写入器

- 循环将数据填入到`Example`协议内存块(protocol buffer)

### 2.8.4.2 代码

对于每一个点击事件样本数据，都需要写入到example当中，所以这里需要取出每一样本进行构造存入

```python
# 保存到TFRecords文件中
df = train_res.select(['user_id', 'article_id', 'clicked', 'features'])
df_array = df.collect()
import pandas as pd
df = pd.DataFrame(df_array)
```

存储

```python
import tensorflow as tf

def write_to_tfrecords(click_batch, feature_batch):
    """将用户与文章的点击日志构造的样本写入TFRecords文件
    """
    
    # 1、构造tfrecords的存储实例
    writer = tf.python_io.TFRecordWriter("./train_ctr_20190605.tfrecords")
    
    # 2、循环将所有样本一个个封装成example，写入这个文件
    for i in range(len(click_batch)):
        # 取出第i个样本的特征值和目标值，格式转换
        click = click_batch[i]
        feature = feature_batch[i].tostring()
        # [18.0, 0.09475817797242475, 0.0543921297305341...
        
        # 构造example，int64, float64, bytes
        example = tf.train.Example(features=tf.train.Features(feature={
            "label": tf.train.Feature(int64_list=tf.train.Int64List(value=[click])),
            "feature": tf.train.Feature(bytes_list=tf.train.BytesList(value=[feature]))
        }))
        
        # 序列化example,写入文件
        writer.write(example.SerializeToString())
    
    writer.close()

# 开启会话打印内容
with tf.Session() as sess:
    # 创建线程协调器
    coord = tf.train.Coordinator()

    # 开启子线程去读取数据
    # 返回子线程实例
    threads = tf.train.start_queue_runners(sess=sess, coord=coord)

    # 存入数据
    write_to_tfrecords(df.iloc[:, 2], df.iloc[:, 3])

    # 关闭子线程，回收
    coord.request_stop()

    coord.join(threads)
```

