
# OpenMLDB 快速上手指南（集群模式）
本教程针对 OpenMLDB 集群模式，会涵盖使用 OpenMLDB 构建机器学习应用的整个流程，包括离线特征抽取、模型训练，以及在线的数据导入、在线特征抽取、模型预测等步骤。希望通过该教程，读者可以了解通过 OpenMLDB，如何快速完成从原始数据处理到模型上线的整个生命周期。


OpenMLDB 提供了 Java 和 Python SDKs，该示例中，我们使用 Python SDK。为了方便理解，我们会基于Kaggle比赛 [Predict Taxi Tour Duration 数据集](https://github.com/4paradigm/OpenMLDB/tree/main/demo/predict-taxi-trip-duration-nb/script/data)，来介绍整个流程。数据集和相关代码可以在[这里](https://github.com/4paradigm/OpenMLDB/tree/main/demo/predict-taxi-trip-duration-nb/script)查看，并且可以根据步骤尝试运行。

## 1. 离线流程
### 1.1. 特征抽取
对于特征抽取，用户首先需要通过对数据集的了解，构建特征抽取的 SQL 脚本，例如，在 Taxi Tour Duration 数据集中，我们可以构建如下特征抽取的 [SQL](https://github.com/4paradigm/OpenMLDB/blob/main/demo/predict-taxi-trip-duration-nb/script/fe.sql):
```sql
SELECT trip_duration, passenger_count,
sum(pickup_latitude) OVER w AS vendor_sum_pl,
max(pickup_latitude) OVER w AS vendor_max_pl,
min(pickup_latitude) OVER w AS vendor_min_pl,
avg(pickup_latitude) OVER w AS vendor_avg_pl,
sum(pickup_latitude) OVER w2 AS pc_sum_pl,
max(pickup_latitude) OVER w2 AS pc_max_pl,
min(pickup_latitude) OVER w2 AS pc_min_pl,
avg(pickup_latitude) OVER w2 AS pc_avg_pl,
count(vendor_id) OVER w2 AS pc_cnt,
count(vendor_id) OVER w AS vendor_cnt
FROM t1
WINDOW w AS (PARTITION BY vendor_id ORDER BY pickup_datetime ROWS_RANGE BETWEEN 1d PRECEDING AND CURRENT ROW),
w2 AS (PARTITION BY passenger_count ORDER BY pickup_datetime ROWS_RANGE BETWEEN 1d PRECEDING AND CURRENT ROW);
```


通过执行特征抽取的 SQL，可以从原数据里提取出我们需要的特征数据集，用来进行训练。对于离线特征抽取的输入数据，可以直接存放在本地文件或者 HDFS 上面，然后通过 Spark 执行特征抽取 SQL。Python 示例代码如下（完整代码参考[这里](https://github.com/4paradigm/OpenMLDB/blob/main/demo/predict-taxi-trip-duration-nb/script/train.py)）：

```python
from pyspark.sql import SparkSession
import numpy as np
import pandas as pd

spark = SparkSession.builder.appName("OpenMLDB Demo").getOrCreate()
parquet_train = "./data/taxi_tour_table_train_simple.snappy.parquet"
train = spark.read.parquet(parquet_train)
train.createOrReplaceTempView("t1")
train_df = spark.sql(sql)
df = train_df.toPandas()
```

***注意***：本示例中使用了 [OpenMLDB Spark Distribution](https://github.com/4paradigm/OpenMLDB/blob/main/docs/en/compile.md#optimized-spark-distribution-for-openmldb-optional)


### 1.2. 模型训练 
通过特征抽取，获得训练和预测数据集，就可以使用标准的模型训练方式，来进行模型训练。下面示例使用 Gradient Boosting Machine (GBM) 进行模型训练，并将训练好的模型保存在`model_path`路径下（完整代码参考[这里](https://github.com/4paradigm/OpenMLDB/blob/main/demo/predict-taxi-trip-duration-nb/script/train.py)）：

```python
import lightgbm as lgb
from sklearn.model_selection import train_test_split

train_set, predict_set = train_test_split(df, test_size=0.2)
y_train = train_set['trip_duration']
x_train = train_set.drop(columns=['trip_duration'])
y_predict = predict_set['trip_duration']
x_predict = predict_set.drop(columns=['trip_duration'])
lgb_train = lgb.Dataset(x_train, y_train)
lgb_eval = lgb.Dataset(x_predict, y_predict, reference=lgb_train)

gbm = lgb.train(params,
                lgb_train,
                num_boost_round=20,
                valid_sets=lgb_eval,
                early_stopping_rounds=5)
gbm.save_model(model_path)
```

## 2. 在线流程
在线服务主要需要两个输入：
- 离线过程中训练好的模型
- 在线数据集

对于在线模型预测，一般我们需要最新的历史数据，来完成特征抽取。这主要是因为特征抽取 SQL 一般是基于一定的时间窗口，因此需要一段时间的数据做特征统计。我们称在线预测需要依赖的这部分最新的历史数据为在线数据集，这部分数据集相对离线训练的数据集会较小，主要为近期的一些历史数据。

### 2.1. 在线数据导入
在线数据集可以通过类似数据库的导入方式，导入OpenMLDB。下面示例展示如何从csv文件导入数据到OpenMLDB，和标准数据库导入数据类似（完整代码参考[这里](https://github.com/4paradigm/OpenMLDB/blob/main/demo/predict-taxi-trip-duration-nb/script/import.py)）

```python
import sqlalchemy as db

ddl="""
create table t1(
id string,
vendor_id int,
pickup_datetime timestamp,
dropoff_datetime timestamp,
passenger_count int,
pickup_longitude double,
pickup_latitude double,
dropoff_longitude double,
dropoff_latitude double,
store_and_fwd_flag string,
trip_duration int,
index(key=vendor_id, ts=pickup_datetime),
index(key=passenger_count, ts=pickup_datetime)
);
"""

engine = db.create_engine('openmldb:///db_test?zk=127.0.0.1:2181&zkPath=/openmldb')
connection = engine.connect()
# 创建数据库
connection.execute("create database db_test;")
# 建表
connection.execute(ddl)

# 从csv读出数据并插入到表里
with open('data/taxi_tour_table_train_simple.csv', 'r') as fd:
    for line in fd:
        row = line.split(',')
        insert = "insert into t1 values('%s', %s, %s, %s, %s, %s, %s, %s, %s, '%s', %s);"% tuple(row)
        connection.execute(insert)
```

### 2.2. 在线特征抽取
对于在线特征抽取，需要根据输入的数据，以及在线数据集，通过和离线特征相同的特征抽取[SQL](https://github.com/4paradigm/OpenMLDB/blob/main/demo/predict-taxi-trip-duration-nb/script/fe.sql)，获得特征。示例代码如下（[完整代码](https://github.com/4paradigm/OpenMLDB/blob/main/demo/predict-taxi-trip-duration-nb/script/predict_server.py)）：

```python
import sqlalchemy as db

engine = db.create_engine('openmldb:///db_test?zk=127.0.0.1:2181&zkPath=/openmldb')
connection = engine.connect()
features = connection.execute(sql, request_data)
```

### 2.3. 模型预测
根据在线特征抽取结果，以及离线训练好的模型，可以获得预测值，就完成了在线预估的过程。


示例代码如下：
```python
import lightgbm as lgb

bst = lgb.Booster(model_path)
duration = bst.predict(feature)
```

***注***：实际中，会启动预估服务，通过线上服务的方式，接收在线输入的数据，进行模型预测，然后再把预测结果返回给用户。这里省略了服务上线的过程，不过在[demo](https://github.com/4paradigm/OpenMLDB/blob/main/demo/predict-taxi-trip-duration-nb/script)的上手示例中，包含了模型上线的过程。
