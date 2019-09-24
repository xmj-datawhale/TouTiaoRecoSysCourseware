# 8.6 分桶与特征交叉

## 学习目标

- 目标
  - 了解分桶方式和作用
- 应用
  - 无

### 8.6.1 通过分桶将连续特征变成类别特征

有时，连续特征与标签不是线性关系。例如，年龄和收入 - 一个人的收入在其职业生涯早期阶段会增长，然后在某一阶段，增长速度减慢，最后，在退休后减少。在这种情况下，使用原始 `age` 作为实值特征列也许并非理想之选，因为模型只能学习以下三种情况之一：

1. 收入始终随着年龄的增长而以某一速率增长（正相关）；
2. 收入始终随着年龄的增长而以某一速率减少（负相关）；或者
3. 无论年龄多大，收入都保持不变（不相关）。

如果我们要分别学习收入与各个年龄段之间的精细关系，则可以采用分桶技巧。分桶是将整个连续特征范围分割为一组连续分桶，然后根据值所在的分桶将原数值特征转换为分桶 ID（作为类别特征）的过程。因此，我们可以针对 `age` 将 `bucketized_column` 定义为：

```python
age_buckets = tf.feature_column.bucketized_column(
    age, boundaries=[18, 25, 30, 35, 40, 45, 50, 55, 60, 65])
```

### 8.6.2 通过特征组合学习复杂关系

单独使用各个基准特征列可能不足以解释数据。例如，对于不同的职业，受教育程度和标签（收入超过 5 万美元）之间的相关性可能各不相同。因此，如果我们仅学习 `education="Bachelors"` 和 `education="Masters"` 的单个模型权重，则无法捕获每个受教育程度-职业组合（例如，区分 `education="Bachelors"` AND `occupation="Exec-managerial"`和 `education="Bachelors" AND occupation="Craft-repair"`）。

要了解各个特征组合之间的差异，我们可以向模型中添加组合特征列：

```python
education_x_occupation = tf.feature_column.crossed_column(['education', 'occupation'], hash_bucket_size=1000)
```

我们还可以针对两个以上的列创建一个 `crossed_column`。每个组成列可以是类别基准特征列 (`SparseColumn`)、分桶实值特征列，也可以是其他 `CrossColumn`。例如：

```python
age_buckets_x_education_x_occupation = tf.feature_column.crossed_column([age_buckets, 'education', 'occupation'], hash_bucket_size=1000)
```

### 8.6.3 添加交叉特征

效果提高

|          | baseline   | Feature intersection |
| -------- | ---------- | -------------------- |
| accuracy | 0.8323813  | 0.8401818            |
| auc      | 0.87850624 | 0.89078486           |

```python
{'accuracy': 0.8323813, 'accuracy_baseline': 0.76377374, 'auc': 0.87850624, 'auc_precision_recall': 0.66792196, 'average_loss': 0.5613808, 'label/mean': 0.23622628, 'loss': 17.956465, 'precision': 0.6553547, 'prediction/mean': 0.24526471, 'recall': 0.61258453, 'global_step': 3053}


{'accuracy': 0.8401818, 'accuracy_baseline': 0.76377374, 'auc': 0.89078486, 'auc_precision_recall': 0.71612483, 'average_loss': 0.3730738, 'label/mean': 0.23622628, 'loss': 11.93323, 'precision': 0.7046053, 'prediction/mean': 0.22067882, 'recall': 0.5569423, 'global_step': 3053}
```

特征列版本

```python
def get_feature_column_v2():
    """特征交叉与分桶
    :return:
    """
    age = tf.feature_column.numeric_column('age')
    education_num = tf.feature_column.numeric_column('education_num')
    capital_gain = tf.feature_column.numeric_column('capital_gain')
    capital_loss = tf.feature_column.numeric_column('capital_loss')
    hours_per_week = tf.feature_column.numeric_column('hours_per_week')

    numeric_columns = [age, education_num, capital_gain, capital_loss, hours_per_week]

    # 类别型特征
    # categorical_column_with_vocabulary_list, 将字符串转换成ID
    relationship = tf.feature_column.categorical_column_with_vocabulary_list(
        'relationship',
        ['Husband', 'Not-in-family', 'Wife', 'Own-child', 'Unmarried', 'Other-relative'])

    marital_status = tf.feature_column.categorical_column_with_vocabulary_list(
        'marital_status', [
            'Married-civ-spouse', 'Divorced', 'Married-spouse-absent',
            'Never-married', 'Separated', 'Married-AF-spouse', 'Widowed'])

    workclass = tf.feature_column.categorical_column_with_vocabulary_list(
        'workclass', [
            'Self-emp-not-inc', 'Private', 'State-gov', 'Federal-gov',
            'Local-gov', '?', 'Self-emp-inc', 'Without-pay', 'Never-worked'])

    # categorical_column_with_hash_bucket--->哈希列
    # 对不确定类别数量以及字符时，哈希列进行分桶
    occupation = tf.feature_column.categorical_column_with_hash_bucket(
        'occupation', hash_bucket_size=1000)
    categorical_columns = [relationship, marital_status, workclass, occupation]

    # 分桶，交叉特征
    age_buckets = tf.feature_column.bucketized_column(
        age, boundaries=[18, 25, 30, 35, 40, 45, 50, 55, 60, 65])
    crossed_columns = [
        tf.feature_column.crossed_column(
            ['education', 'occupation'], hash_bucket_size=1000),
        tf.feature_column.crossed_column(
            [age_buckets, 'education', 'occupation'], hash_bucket_size=1000),
    ]

    return numeric_columns + categorical_columns + crossed_columns
```

模型训练

```python
# 分桶与特征交叉
# # 构造模型
feature_v2 = get_feature_column_v2()
classifiry = tf.estimator.LinearClassifier(feature_columns=feature_v2)
# train输入的input_func，不能调用传入
# 1、input_func，构造的时候不加参数，但是这样不灵活， 里面参数不能固定的时候
# 2、functools.partial
train_func = functools.partial(input_func, train_file, epoches=3, batch_size=32)
test_func = functools.partial(input_func, test_file, epoches=1, batch_size=32)
classifiry.train(train_func)
result = classifiry.evaluate(test_func)
print(result)
```

