# 基于 HDFS-HA-QJM 部署 Spark

HDFS-HA-QJM 的部署安装请参考 hdfs-ha-qjm 分支

[参考文档](https://spark.apache.org/docs/latest/quick-start.html)

## 环境

[spark-3.3.0-bin-hadoop3.tgz](https://mirrors.tuna.tsinghua.edu.cn/apache/spark/spark-3.3.0/spark-3.3.0-bin-hadoop3.tgz)

## 服务器规划

192.168.1.101 用于本地模式测试

192.168.1.102 用于独立模式部署

192.168.1.103 基于 yarn 的模式部署

| 192.168.1.101   | 192.168.1.102    | 192.168.1.103 |
| --------------- | ---------------- | ------------- |
| NameNode        | NameNode         |               |
| JournalNode     | JournalNode      | JournalNode   |
| DataNode        | DataNode         | DataNode      |
| zookeeper       | zookeeper        | zookeeper     |
| ZKFC            | ZKFC             |               |
| ResourceManager | ResourceManager  |               |
| NodeManager     | NodeManager      | NodeManager   |
| spark-master    | spark-master     |               |
| spark-work      | spark-work       | spark-work    |
| spark-local     | spark-standAlone | spark-yarn    |

## 获取 Spark 安装包

```bash
su - hadoop

wget https://mirrors.tuna.tsinghua.edu.cn/apache/spark/spark-3.3.0/spark-3.3.0-bin-hadoop3.tgz

tar xaf spark-3.3.0-bin-hadoop3.tgz
```

## Local 模式

### 安装 Python

```bash
yum install -y python36
```

### 测试

`./bin/pyspark`

```python
textFile = spark.read.text("README.md")
textFile.count()
textFile.first()

linesWithSpark = textFile.filter(textFile.value.contains("Spark"))
textFile.filter(textFile.value.contains("Spark")).count()

from pyspark.sql.functions import *
textFile.select(size(split(textFile.value, "\s+")).name("numWords")).agg(max(col("numWords"))).collect()

wordCounts = textFile.select(explode(split(textFile.value, "\s+")).alias("word")).groupBy("word").count()
wordCounts.collect()

linesWithSpark.cache()
linesWithSpark.count()
```

## StandAlone-HA 模式

不同运行环境需要安装不同的语言包并配置相应的环境变量

### 各服务器安装 Python

```bash
yum install -y python36
```

### 配置环境变量

#### /home/hadoop/.bashrc

```bash
export SPARK_HOME="/home/hadoop/spark-3.3.0-bin-hadoop3"
export PATH=$PATH:$SPARK_HOME/bin
```

`source ~/.bashrc`

#### $SPARK_HOME/conf/spark-env.sh

```bash
#!/usr/bin/env bash

HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop/
YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop/

# 单节点配置
# SPARK_MASTER_HOST=hadoop001

# 高可用配置
SPARK_DAEMON_JAVA_OPTS='-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=hadoop001:2181,hadoop002:2181,hadoop003:2181 -Dspark.deploy.zookeeper.dir=/spark'
SPARK_MASTER_PORT=7077
SPARK_MASTER_WEBUI_PORT=8081

SPARK_WORKER_CORES=2
SPARK_WORKER_MEMORY=2g
SPARK_WORKER_PORT=7071
SPARK_WORKER_WEBUI_PORT=8082

SPARK_HISTORY_OPTS='-Dspark.history.ui.port=18080 -Dspark.history.retainedApplications=3 -Dspark.history.fs.logDirectory=hdfs://hadoop001:8020/spark/logs -Dspark.history.fs.cleaner.enabled=true'
```

#### $SPARK_HOME/conf/spark-defaults.conf

```conf
spark.eventLog.enabled           true
spark.eventLog.compress          true
spark.eventLog.dir               hdfs://hadoop001:8020/spark/logs
```

#### $SPARK_HOME/conf/log4j2.properties

使用默认配置即可

`cp $SPARK_HOME/conf/log4j2.properties.template $SPARK_HOME/conf/log4j2.properties`

#### $SPARK_HOME/conf/workers

```
hadoop001
hadoop002
hadoop003
```

### 分发程序包至各服务器

```bash
cd /home/hadoop/
scp -r spark-3.3.0-bin-hadoop3 hadoop001:/home/hadoop
scp -r spark-3.3.0-bin-hadoop3 hadoop003:/home/hadoop
```

### 创建历史文件存储目录

```bash
hadoop fs -mkdir -p /spark/logs
```

### 启动集群

```bash
cd /home/hadoop/spark-3.3.0-bin-hadoop3

sbin/start-history-server.sh 
sbin/start-all.sh

# 在另外两台服务器中找一台执行 master 的启动命令
sbin/start-master.sh 
```

### 测试

```bash
bin/pyspark --master spark://hadoop001:7077
```

```python
textFile = spark.read.text("input/*.xml")
textFile.count()
textFile.first()
```

提交方式执行

`bin/spark-submit --master spark://hadoop001:7077 examples/src/main/python/pi.py`

---

## Spark On Yarn

### 配置环境变量

#### /home/hadoop/.bashrc

```bash
export SPARK_HOME="/home/hadoop/spark-3.3.0-bin-hadoop3"
export PATH=$PATH:$SPARK_HOME/bin
```

`source ~/.bashrc`

#### $SPARK_HOME/conf/spark-env.sh

```bash
#!/usr/bin/env bash

HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop/
YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop/
```

### 启动 hdfs

```bash
# 分别在三台主机执行
~/apache-zookeeper-3.7.1-bin/bin/zkServer.sh start
# 在 namenode 节点执行；
start-all.sh
```

### 测试 spark

```bash
cd spark-3.3.0-bin-hadoop3
spark-submit --class org.apache.spark.examples.SparkPi \
    --master yarn \
    --deploy-mode cluster \
    --driver-memory 2g \
    --executor-memory 2g \
    --executor-cores 1 \
    --queue default \
    examples/jars/spark-examples*.jar \
    10
```

如果 `--queue` 设置为 `thequeue` 则报如下错误

```
INFO Client: Deleted staging directory hdfs://mycluster/user/hadoop/.sparkStaging/application_1663772501430_0002
Exception in thread "main" org.apache.hadoop.yarn.exceptions.YarnException: Failed to submit application_1663772501430_0002 to YARN : Application application_1663772501430_0002 submitted by user hadoop to unknown queue: thequeue
```
