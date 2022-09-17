# 基于 HDFS-HA-QJM 部署 hive

HDFS-HA-QJM 的部署安装请参考 hdfs-ha-qjm 分支

## 环境

[apache-hive-3.1.3-bin.tar.gz](https://dlcdn.apache.org/hive/hive-3.1.3/apache-hive-3.1.3-bin.tar.gz)

## 服务器规划

| 172.16.16.100   | 172.16.16.101   | 172.16.16.102 |
| --------------- | --------------- | ------------- |
| NameNode        | NameNode        |               |
| JournalNode     | JournalNode     | JournalNode   |
| DataNode        | DataNode        | DataNode      |
| zookeeper       | zookeeper       | zookeeper     |
| ZKFC            | ZKFC            |               |
| ResourceManager | ResourceManager |               |
| NodeManager     | NodeManager     | NodeManager   |
|                 |                 | HiveServer2   |

## 获取 Hive 安装包

```bash
su - hadoop

wget https://mirrors.tuna.tsinghua.edu.cn/apache/hive/hive-3.1.3/apache-hive-3.1.3-bin.tar.gz

tar xaf apache-hive-3.1.3-bin.tar.gz

cd apache-hive-3.1.3-bin
```

## 配置环境变量

#### /home/hadoop/.bashrc

```bash
export HIVE_HOME="/home/hadoop/apache-hive-3.1.3-bin"
export PATH=$PATH:$HIVE_HOME/bin
```

`source ~/.bashrc`

#### $HADOOP_HOME/etc/hadoop/core-site.xml

新增如下配置，参考文档

> https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/Superusers.html

```xml
    <property>
        <name>hadoop.proxyuser.hadoop.hosts</name>
        <value>192.168.1.0/24</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hadoop.groups</name>
        <value>*</value>
    </property>
```

## 启动 hdfs

```bash
# 分别在三台主机执行
~/apache-zookeeper-3.7.1-bin/bin/zkServer.sh start
# 在 namenode 节点执行；执行后检查一下，发现有未启动的情况。
start-all.sh
```

## 创建 hive 所需目录

```bash
hadoop fs -mkdir /tmp
hadoop fs -chmod g+w /tmp
hadoop fs -mkdir -p /user/hive/warehouse
hadoop fs -chmod g+w /user/hive/warehouse
```

## 内嵌模式

> https://cwiki.apache.org/confluence/display/Hive/AdminManual+Metastore+3.0+Administration

在此配置中，只有一个客户端可以使用 Metastore，并且任何更改在客户端生命周期后都不会持久

#### $HIVE_HOME/conf/hive-site.xml

新建此文件

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>hive.metastore.db.type</name>
    <value>DERBY</value>
    <description>
      Expects one of [derby, oracle, mysql, mssql, postgres].
      Type of database used by the metastore. Information schema & JDBCStorageHandler depend on it.
    </description>
  </property>
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
    <description>location of default database for the warehouse</description>
  </property>
  <property>
    <name>hive.metastore.uris</name>
    <value/>
    <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:derby:;databaseName=metastore_db;create=true</value>
    <description>
      JDBC connect string for a JDBC metastore.
      To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL. 
      For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
    </description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.apache.derby.jdbc.EmbeddedDriver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>
  <property>
    <name>hive.metastore.schema.verification</name>
    <value>false</value>
    <description>
      Enforce metastore schema version consistency.
      True: Verify that version information stored in is compatible with one from Hive jars.  Also disable automatic
            schema migration attempt. Users are required to manually migrate schema after Hive upgrade which ensures
            proper metastore schema migration. (Default)
      False: Warn if the version information stored in metastore doesn't match with one from in Hive jars.
    </description>
  </property>
  <property>
    <name>hive.server2.enable.doAs</name>
    <value>false</value>
  </property>
</configuration>
```

### 初始化 Derby 数据库

```bash
schematool -initSchema -dbType derby
```

执行成功提示信息

> Initialization script completed
> schemaTool completed

生成的数据文件在 `$HIVE_HOME/metastore_db` 

### 连接 hive 数据库

#### 通过 hive 命令

```bash
hive
```

#### 通过 hiveserver 服务

启动 hiveserver2

```bash
hiveserver2
```

测试连接

```bash
beeline -n hadoop -u jdbc:hive2://localhost:10000

# 退出 beeline
jdbc:hive2://localhost:10000> !q
```

### 测试 SQL 语句

```sql
create table test(id string);

insert into test values('1');

select * from test;
```

---
