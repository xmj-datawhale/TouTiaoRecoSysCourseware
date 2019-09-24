# 2.4 案例：实现线性回归

## 学习目标

- 目标
  - 应用op的name参数实现op的名字修改
  - 应用variable_scope实现图程序作用域的添加
  - 应用scalar或histogram实现张量值的跟踪显示
  - 应用merge_all实现张量值的合并
  - 应用add_summary实现张量值写入文件
  - 应用tf.train.saver实现TensorFlow的模型保存以及加载
  - 应用tf.app.flags实现命令行参数添加和使用
  - 应用reduce_mean、square实现均方误差计算
  - 应用tf.train.GradientDescentOptimizer实现有梯度下降优化器创建
  - 应用minimize函数优化损失
  - 知道梯度爆炸以及常见解决技巧
- 应用
  - 实现线性回归模型
- 内容预览
  - 2.6.1 线性回归原理复习
  - 2.6.2 案例：实现线性回归的训练
  - 2.6.3 增加其他功能
    - 1 增加变量显示
    - 2 增加命名空间
    - 3 模型的保存与加载
    - 4 命令行参数使用 

## 2.4.1 线性回归原理复习

根据数据建立回归模型，w1x1+w2x2+…..+b = y，通过真实值与预测值之间建立误差，使用梯度下降优化得到损失最小对应的权重和偏置。最终确定模型的权重和偏置参数。最后可以用这些参数进行预测。

## 2.4.2 案例：实现线性回归的训练

### 1 案例确定

- 假设随机指定100个点，只有一个特征
- 数据本身的分布为 y = 0.8 * x + 0.7

> 这里将数据分布的规律确定，是为了使我们训练出的参数跟真实的参数（即0.8和0.7）比较是否训练准确

### 2 API

**运算**

- 矩阵运算
  - tf.matmul(x, w)
- 平方
  - tf.square(error)
- 均值
  - tf.reduce_mean(error)

**梯度下降优化**

- tf.train.GradientDescentOptimizer(learning_rate)
  - 梯度下降优化
  - learning_rate:学习率，一般为0~1之间比较小的值
  - method:
    - minimize(loss)
  - return:梯度下降op

### 3 步骤分析

- 1 准备好数据集：y = 0.8x + 0.7 100个样本
- 2 建立线性模型
  - 随机初始化W1和b1
  - y = W·X + b，目标：求出权重W和偏置b
- 3 确定损失函数（预测值与真实值之间的误差）-均方误差
- 4 梯度下降优化损失：需要指定学习率（超参数）

### 4 实现完整功能

```python
import tensorflow as tf
import os

def linear_regression():
    """
    自实现线性回归
    :return: None
    """
    # 1）准备好数据集：y = 0.8x + 0.7 100个样本
    # 特征值X, 目标值y_true
    X = tf.random_normal(shape=(100, 1), mean=2, stddev=2)
    # y_true [100, 1]
    # 矩阵运算 X（100， 1）* （1, 1）= y_true(100, 1)
    y_true = tf.matmul(X, [[0.8]]) + 0.7
    # 2）建立线性模型：
    # y = W·X + b，目标：求出权重W和偏置b
    # 3）随机初始化W1和b1
    weights = tf.Variable(initial_value=tf.random_normal(shape=(1, 1)))
    bias = tf.Variable(initial_value=tf.random_normal(shape=(1, 1)))
    y_predict = tf.matmul(X, weights) + bias
    # 4）确定损失函数（预测值与真实值之间的误差）-均方误差
    error = tf.reduce_mean(tf.square(y_predict - y_true))
    # 5）梯度下降优化损失：需要指定学习率（超参数）
    # W2 = W1 - 学习率*(方向)
    # b2 = b1 - 学习率*(方向)
    optimizer = tf.train.GradientDescentOptimizer(learning_rate=0.01).minimize(error)

    # 初始化变量
    init = tf.global_variables_initializer()
    # 开启会话进行训练
    with tf.Session() as sess:
        # 运行初始化变量Op
        sess.run(init)
        print("随机初始化的权重为%f， 偏置为%f" % (weights.eval(), bias.eval()))
        # 训练模型
        for i in range(100):
            sess.run(optimizer)
            print("第%d步的误差为%f，权重为%f， 偏置为%f" % (i, error.eval(), weights.eval(), bias.eval()))

    return None
```

### 6 变量的trainable设置观察

trainable的参数作用，指定是否训练

```python
weight = tf.Variable(tf.random_normal([1, 1], mean=0.0, stddev=1.0), name="weights", trainable=False)
```

## 2.4.3 增加其他功能

- **增加命名空间**
- **命令行参数设置**

### 2 增加命名空间 

是代码结构更加清晰，Tensorboard图结构清楚

```python
with tf.variable_scope("lr_model"):
```

```python
def linear_regression():
    # 1）准备好数据集：y = 0.8x + 0.7 100个样本
    # 特征值X, 目标值y_true
    with tf.variable_scope("original_data"):
        X = tf.random_normal(shape=(100, 1), mean=2, stddev=2, name="original_data_x")
        # y_true [100, 1]
        # 矩阵运算 X（100， 1）* （1, 1）= y_true(100, 1)
        y_true = tf.matmul(X, [[0.8]], name="original_matmul") + 0.7
    # 2）建立线性模型：
    # y = W·X + b，目标：求出权重W和偏置b
    # 3）随机初始化W1和b1
    with tf.variable_scope("linear_model"):
        weights = tf.Variable(initial_value=tf.random_normal(shape=(1, 1)), name="weights")
        bias = tf.Variable(initial_value=tf.random_normal(shape=(1, 1)), name="bias")
        y_predict = tf.matmul(X, weights, name="model_matmul") + bias
    # 4）确定损失函数（预测值与真实值之间的误差）-均方误差
    with tf.variable_scope("loss"):
        error = tf.reduce_mean(tf.square(y_predict - y_true), name="error_op")
    # 5）梯度下降优化损失：需要指定学习率（超参数）
    # W2 = W1 - 学习率*(方向)
    # b2 = b1 - 学习率*(方向)
    with tf.variable_scope("gd_optimizer"):
        optimizer = tf.train.GradientDescentOptimizer(learning_rate=0.01, name="optimizer").minimize(error)

    # 2）收集变量
    tf.summary.scalar("error", error)
    tf.summary.histogram("weights", weights)
    tf.summary.histogram("bias", bias)

    # 3）合并变量
    merge = tf.summary.merge_all()

    # 初始化变量
    init = tf.global_variables_initializer()
    # 开启会话进行训练
    with tf.Session() as sess:
        # 运行初始化变量Op
        sess.run(init)
        print("随机初始化的权重为%f， 偏置为%f" % (weights.eval(), bias.eval()))
        # 1）创建事件文件
        file_writer = tf.summary.FileWriter(logdir="./summary", graph=sess.graph)
        # 训练模型
        for i in range(100):
            sess.run(optimizer)
            print("第%d步的误差为%f，权重为%f， 偏置为%f" % (i, error.eval(), weights.eval(), bias.eval()))
            # 4）运行合并变量op
            summary = sess.run(merge)
            file_writer.add_summary(summary, i)

    return None
```

### 3 模型的保存与加载 

- **tf.train.Saver(var_list=None,max_to_keep=5)**
  - 保存和加载模型（保存文件格式：checkpoint文件）
  - var_list:指定将要保存和还原的变量。它可以作为一个dict或一个列表传递.
  - max_to_keep：指示要保留的最近检查点文件的最大数量。创建新文件时，会删除较旧的文件。如果无或0，则保留所有检查点文件。默认为5（即保留最新的5个检查点文件。）

使用

```python
例如：
指定目录+模型名字
saver.save(sess, '/tmp/ckpt/test/myregression.ckpt')
saver.restore(sess, '/tmp/ckpt/test/myregression.ckpt')
```

如要判断模型是否存在，直接指定目录

```python
checkpoint = tf.train.latest_checkpoint("./tmp/model/")

saver.restore(sess, checkpoint)
```

### 4 命令行参数使用 

- 1、![](/Users/huxinghui/Desktop/%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E4%B8%8E%E6%A3%80%E6%B5%8B%E9%A1%B9%E7%9B%AE%E8%AF%BE%E4%BB%B6/images/%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%8F%82%E6%95%B0.png)

- 2、 tf.app.flags.,在flags有一个FLAGS标志，它在程序中可以调用到我们

前面具体定义的flag_name

- 3、通过tf.app.run()启动main(argv)函数

```python
# 定义一些常用的命令行参数
# 训练步数
tf.app.flags.DEFINE_integer("max_step", 0, "训练模型的步数")
# 定义模型的路径
tf.app.flags.DEFINE_string("model_dir", " ", "模型保存的路径+模型名字")

# 定义获取命令行参数
FLAGS = tf.app.flags.FLAGS

# 开启训练
# 训练的步数（依据模型大小而定）
for i in range(FLAGS.max_step):
     sess.run(train_op)
```

### 完整代码

```python
import tensorflow as tf
import os

tf.app.flags.DEFINE_string("model_path", "./linear_regression/", "模型保存的路径和文件名")
FLAGS = tf.app.flags.FLAGS


def linear_regression():
    # 1）准备好数据集：y = 0.8x + 0.7 100个样本
    # 特征值X, 目标值y_true
    with tf.variable_scope("original_data"):
        X = tf.random_normal(shape=(100, 1), mean=2, stddev=2, name="original_data_x")
        # y_true [100, 1]
        # 矩阵运算 X（100， 1）* （1, 1）= y_true(100, 1)
        y_true = tf.matmul(X, [[0.8]], name="original_matmul") + 0.7
    # 2）建立线性模型：
    # y = W·X + b，目标：求出权重W和偏置b
    # 3）随机初始化W1和b1
    with tf.variable_scope("linear_model"):
        weights = tf.Variable(initial_value=tf.random_normal(shape=(1, 1)), name="weights")
        bias = tf.Variable(initial_value=tf.random_normal(shape=(1, 1)), name="bias")
        y_predict = tf.matmul(X, weights, name="model_matmul") + bias
    # 4）确定损失函数（预测值与真实值之间的误差）-均方误差
    with tf.variable_scope("loss"):
        error = tf.reduce_mean(tf.square(y_predict - y_true), name="error_op")
    # 5）梯度下降优化损失：需要指定学习率（超参数）
    # W2 = W1 - 学习率*(方向)
    # b2 = b1 - 学习率*(方向)
    with tf.variable_scope("gd_optimizer"):
        optimizer = tf.train.GradientDescentOptimizer(learning_rate=0.01, name="optimizer").minimize(error)

    # 2）收集变量
    tf.summary.scalar("error", error)
    tf.summary.histogram("weights", weights)
    tf.summary.histogram("bias", bias)

    # 3）合并变量
    merge = tf.summary.merge_all()

    # 初始化变量
    init = tf.global_variables_initializer()

    # 开启会话进行训练
    with tf.Session() as sess:
        # 运行初始化变量Op
        sess.run(init)
        # 未经训练的权重和偏置
        print("随机初始化的权重为%f， 偏置为%f" % (weights.eval(), bias.eval()))
        # 当存在checkpoint文件，就加载模型

        # 1）创建事件文件
        file_writer = tf.summary.FileWriter(logdir="./summary", graph=sess.graph)
        # 训练模型
        for i in range(100):
            sess.run(optimizer)
            print("第%d步的误差为%f，权重为%f， 偏置为%f" % (i, error.eval(), weights.eval(), bias.eval()))
            # 4）运行合并变量op
            summary = sess.run(merge)
            file_writer.add_summary(summary, i)
          
    return None


def main(argv):
    print("这是main函数")
    print(argv)
    print(FLAGS.model_path)
    linear_regression()

if __name__ == "__main__":
    tf.app.run()
```

### 作业：将面向过程改为面向对象

参考代码

```python
# 用tensorflow自实现一个线性回归案例

# 定义一些常用的命令行参数
# 训练步数
tf.app.flags.DEFINE_integer("max_step", 0, "训练模型的步数")
# 定义模型的路径
tf.app.flags.DEFINE_string("model_dir", " ", "模型保存的路径+模型名字")

FLAGS = tf.app.flags.FLAGS

class MyLinearRegression(object):
    """
    自实现线性回归
    """
    def __init__(self):
        pass

    def inputs(self):
        """
        获取特征值目标值数据数据
        :return:
        """
        x_data = tf.random_normal([100, 1], mean=1.0, stddev=1.0, name="x_data")
        y_true = tf.matmul(x_data, [[0.7]]) + 0.8

        return x_data, y_true

    def inference(self, feature):
        """
        根据输入数据建立模型
        :param feature:
        :param label:
        :return:
        """
        with tf.variable_scope("linea_model"):
            # 2、建立回归模型，分析别人的数据的特征数量--->权重数量， 偏置b
            # 由于有梯度下降算法优化，所以一开始给随机的参数，权重和偏置
            # 被优化的参数，必须得使用变量op去定义
            # 变量初始化权重和偏置
            # weight 2维[1, 1]    bias [1]
            # 变量op当中会有trainable参数决定是否训练
            self.weight = tf.Variable(tf.random_normal([1, 1], mean=0.0, stddev=1.0),
                                 name="weights")

            self.bias = tf.Variable(0.0, name='biases')

            # 建立回归公式去得出预测结果
            y_predict = tf.matmul(feature, self.weight) + self.bias

        return y_predict

    def loss(self, y_true, y_predict):
        """
        目标值和真实值计算损失
        :return: loss
        """
        # 3、求出我们模型跟真实数据之间的损失
        # 均方误差公式
        loss = tf.reduce_mean(tf.square(y_true - y_predict))

        return loss

    def merge_summary(self, loss):

        # 1、收集张量的值
        tf.summary.scalar("losses", loss)

        tf.summary.histogram("w", self.weight)
        tf.summary.histogram('b', self.bias)

        # 2、合并变量
        merged = tf.summary.merge_all()

        return merged

    def sgd_op(self, loss):
        """
        获取训练OP
        :return:
        """
        # 4、使用梯度下降优化器优化
        # 填充学习率：0 ~ 1    学习率是非常小，
        # 学习率大小决定你到达损失一个步数多少
        # 最小化损失
        train_op = tf.train.GradientDescentOptimizer(0.1).minimize(loss)

        return train_op

    def train(self):
        """
        训练模型
        :param loss:
        :return:
        """

        g = tf.get_default_graph()

        with g.as_default():

            x_data, y_true = self.inputs()

            y_predict = self.inference(x_data)

            loss = self.loss(y_true, y_predict)

            train_op = self.sgd_op(loss)

            # 收集观察的结果值
            merged = self.merge_summary(loss)

            saver = tf.train.Saver()

            with tf.Session() as sess:

                sess.run(tf.global_variables_initializer())
                
                # 在没训练，模型的参数值
        		print("初始化的权重：%f, 偏置：%f" % (self.weight.eval(), self.bias.eval()))

                # 开启训练
                # 训练的步数（依据模型大小而定）
                for i in range(FLAGS.max_step):

                    sess.run(train_op)

                    # 生成事件文件，观察图结构
                    file_writer = tf.summary.FileWriter("./tmp/summary/", graph=sess.graph)

                    print("训练第%d步之后的损失:%f, 权重：%f, 偏置：%f" % (
                        i,
                        loss.eval(),
                        self.weight.eval(),
                        self.bias.eval()))

                    # 运行收集变量的结果
                    summary = sess.run(merged)

                    # 添加到文件
                    file_writer.add_summary(summary, i)


if __name__ == '__main__':
    lr = MyLinearRegression()
    lr.train()
```



