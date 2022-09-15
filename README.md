# 伪分布式

启动服务器

`cd vagrant`

`vagrant up`

## 环境

操作系统：CentOS 7.9 (Linux hostname 3.10.0-1160.25.1.el7.x86_64)

Java™：java-1.8.0-openjdk （ yum install -y java-1.8.0-openjdk ）

远程连接：openssh （ yum install -y openssh ）

[hadoop-3.3.4.tar.gz](https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.3.4/hadoop-3.3.4.tar.gz)

## 配置 Hadoop

```bash
# 添加普通用户
useradd hadoop
su - hadoop

# 解压下载好的安装包
tar xaf /vagrant/hadoop-3.3.4.tar.gz -C .
cd hadoop-3.3.4

# 编辑 Java 环境变量 （ yum install -y java-1.8.0-openjdk ）
vi etc/hadoop/hadoop-env.sh
export JAVA_HOME=/usr/lib/jvm/java-1.8.0

# 执行命令测试是否正常输出
bin/hadoop

```

#### etc/hadoop/core-site.xml

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

#### etc/hadoop/hdfs-site.xml

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

## 设置 ssh 免密登录

```bash
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
```

## 格式化文件系统

```bash
bin/hdfs namenode -format
```

成功提示信息

> INFO common.Storage: Storage directory /tmp/hadoop-hadoop/dfs/name has been successfully formatted.

## 启动 dfs

```bash
sbin/start-dfs.sh

# 临时开启防火墙的端口访问
firewall-cmd --add-port=9870/tcp --permanent
firewall-cmd --reload
```

## 访问测试

192.168.1.101:9870

功能性测试

```bash
bin/hdfs dfs -mkdir /user
bin/hdfs dfs -mkdir /user/hadoop

bin/hdfs dfs -mkdir input
bin/hdfs dfs -put etc/hadoop/*.xml input

bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.4.jar grep input output 'dfs[a-z.]+'

bin/hdfs dfs -cat output/*
```

---

## YARN on a Single Node

#### etc/hadoop/mapred-site.xml

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

#### etc/hadoop/yarn-site.xml

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

### 启动 yarn

```bash
sbin/start-yarn.sh

firewall-cmd --add-port=8088/tcp
```

访问测试

http://192.168.1.101:8088/