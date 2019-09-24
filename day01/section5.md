# 2.6 Apscheduler定时更新文章画像

## 学习目标

- 目标
  - 知道Apscheduler定时更新工具使用
- 应用
  - 应用Super进程管理与Apscheduler完成定时文章画像更新

### 2.6.1 Apscheduler使用

**APScheduler**：强大的**任务调度工具**，可以完成**定时任务**，**周期任务**等，它是跨平台的，用于取代Linux下的cron daemon或者Windows下的task scheduler。

**下载**

```python
pip install APScheduler==3.5.3
```

**基本概念：4个组件**
 `triggers`: 描述一个任务何时被触发，有按日期、按时间间隔、按cronjob描述式三种触发方式
 `job stores`: 任务持久化仓库，默认保存任务在内存中，也可将任务保存都各种数据库中，任务中的数据序列化后保存到持久化数据库，从数据库加载后又反序列化。
 `executors`: 执行任务模块，当任务完成时executors通知schedulers，schedulers收到后会发出一个适当的事件
 `schedulers`: 任务调度器，控制器角色，通过它配置job stores和executors，添加、修改和删除任务。

**简单使用**

```python
from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.executors.pool import ProcessPoolExecutor
 
def test_job():
    print("python")
 
# 创建scheduler，多进程执行
executors = {
    'default': ProcessPoolExecutor(3)
}

scheduler = BlockingScheduler(executors=executors)
'''
 #该示例代码生成了一个BlockingScheduler调度器，使用了默认的默认的任务存储MemoryJobStore，以及默认的执行器ThreadPoolExecutor，并且最大线程数为10。
'''
scheduler.add_job(test_job, trigger='interval', seconds=5)
'''
 #该示例中的定时任务采用固定时间间隔（interval）的方式，每隔5秒钟执行一次。
 #并且还为该任务设置了一个任务id
'''
scheduler.start()
```

##### 在目录中新建scheduler目录用于启动各种定时任务

新建main.py文件编写Apscheduler代码，和update.py文件用于调用各种更新函数

```python
from offline.update_article import UpdateArticle

def update_article_profile():
    """
    更新文章画像
    :return:
    """
    ua = UpdateArticle()
    sentence_df = ua.merge_article_data()
    if sentence_df.rdd.collect():
      text_rank_res = ua.generate_article_label(sentence_df)
      article_profile = ua.get_article_profile(text_rank_res)
```

编写定时运行代码

```python
import sys
import os
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
sys.path.insert(0, os.path.join(BASE_DIR))
sys.path.insert(0, os.path.join(BASE_DIR, 'reco_sys'))
from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.executors.pool import ProcessPoolExecutor
from scheduler.update import update_article_profile


# 创建scheduler，多进程执行
executors = {
    'default': ProcessPoolExecutor(3)
}

scheduler = BlockingScheduler(executors=executors)

# 添加定时更新任务更新文章画像,每隔一小时更新
scheduler.add_job(update_article_profile, trigger='interval', hours=1)


scheduler.start()
```

**为了观察定期运行结果和异常，添加日志**

创建setting目录，添加自定义日志文件logging.py创建

```python
import logging
import logging.handlers
import os

logging_file_dir = '/root/logs/'


def create_logger():
    """
    设置日志
    :param app:
    :return:
    """

    # 离线处理更新打印日志
    trace_file_handler = logging.FileHandler(
        os.path.join(logging_file_dir, 'offline.log')
    )
    trace_file_handler.setFormatter(logging.Formatter('%(message)s'))
    log_trace = logging.getLogger('offline')
    log_trace.addHandler(trace_file_handler)
    log_trace.setLevel(logging.INFO)
```

在Apscheduler的main文件中加入，运行时初始化一次

```python
from settings import logging as lg
lg.create_logger()
```

并在需要打日志的文件中如update_article.py中加入：

```python
import logging

logger = logging.getLogger('offline')

logger.info
logger.warn
```

#### 2.6.2 Supervisor进程管理

在reco.conf中添加如下：

```shell
[program:offline]
environment=JAVA_HOME=/root/bigdata/jdk,SPARK_HOME=/root/bigdata/spark,HADOOP_HOME=/root/bigdata/hadoop,PYSPARK_PYTHON=/miniconda2/envs/reco_sys/bin/python,PYSPARK_DRIVER_PYTHON=/miniconda2/envs/reco_sys/bin/python
command=/miniconda2/envs/reco_sys/bin/python /root/toutiao_project/scheduler/main.py
directory=/root/toutiao_project/scheduler
user=root
autorestart=true
redirect_stderr=true
stdout_logfile=/root/logs/offlinesuper.log
loglevel=info
stopsignal=KILL
stopasgroup=true
killasgroup=true
```

