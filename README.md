# 基于 HDFS-HA-QJM 部署 HBase

HDFS-HA-QJM 的部署安装请参考 hdfs-ha-qjm 分支

[参考文档](https://hbase.apache.org/book.html#fully_dist)

## 环境

[hbase-2.5.0-bin.tar.gz](https://dlcdn.apache.org/hbase/2.5.0/hbase-2.5.0-bin.tar.gz)

## 服务器规划

| 192.168.1.101   | 192.168.1.102     | 192.168.1.103 |
| --------------- | ----------------- | ------------- |
| NameNode        | NameNode          |               |
| JournalNode     | JournalNode       | JournalNode   |
| DataNode        | DataNode          | DataNode      |
| zookeeper       | zookeeper         | zookeeper     |
| ZKFC            | ZKFC              |               |
| ResourceManager | ResourceManager   |               |
| NodeManager     | NodeManager       | NodeManager   |
| HbaseMaster     | HbaseMasterBackup |               |
| RegionServer    | RegionServer      | RegionServer  |

## 获取 Hbase 安装包

```bash
su - hadoop

wget https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/2.5.0/hbase-2.5.0-bin.tar.gz

tar xaf hbase-2.5.0-bin.tar.gz
```

## 配置环境变量

#### /home/hadoop/.bashrc

```bash
export HBASE_HOME="/home/hadoop/hbase-2.5.0"
export PATH=$PATH:$HBASE_HOME/bin
```

`source ~/.bashrc`

#### $HBASE_HOME/conf/hbase-env.sh

```bash
#!/usr/bin/env bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0
export HBASE_MANAGES_ZK=false
```

#### $HBASE_HOME/conf/hbase-site.xml

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://hadoop001:8020/hbase</value>
    </property>
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <property>
        <name>hbase.zookeeper.quorum</name>
        <value>hadoop001,hadoop002,hadoop003</value>
    </property>
    <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/var/lib/zookeeper</value>
    </property>
    <property>
        <name>hbase.wal.provider</name>
        <value>filesystem</value>
    </property>
</configuration>
```

#### $HBASE_HOME/conf/regionservers

regionservers

```
hadoop001
hadoop002
hadoop003
```

#### $HBASE_HOME/conf/backup-masters

备节点

```
hadoop002
```

#### $HADOOP_HOME/etc/hadoop/hdfs-site.xml

新增如下配置

```xml
<property>
  <name>dfs.datanode.max.transfer.threads</name>
  <value>4096</value>
</property>
```

## 分发程序包

```bash
scp -r ~/hbase-2.5.0 hadoop002:~
```

## 启动 hdfs

```bash
# 分别在三台主机执行
~/apache-zookeeper-3.7.1-bin/bin/zkServer.sh start
# 在 namenode 节点执行；
start-all.sh
```

## 启动 hbase

在 hadoop001 执行

```bash
start-hbase.sh
```

单个组件启动

```bash
hbase-daemon.sh start master
```

## 操作测试

```bash
hbase shell

version

status

create 'test', 'cf'

list 'test'

describe 'test'

put 'test', 'row1', 'cf:a', 'value1'
put 'test', 'row2', 'cf:b', 'value2'
put 'test', 'row3', 'cf:c', 'value3'

scan 'test'

get 'test', 'row1'

disable 'test'
drop 'test'

quit
```

## 网页访问

http://hadoop001:16010