# 完全分布式

启动服务器

`cd vagrant`

`vagrant up`

## 环境

操作系统：2C4G , CentOS 7.9 (Linux hostname 3.10.0-1160.25.1.el7.x86_64)

Java™：java-1.8.0-openjdk （ yum install -y java-1.8.0-openjdk-devel ）

远程连接：openssh （ yum install -y openssh ）

[hadoop-3.3.4.tar.gz](https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.3.4/hadoop-3.3.4.tar.gz)

## 服务器规划

| 192.168.1.101   | 192.168.1.102      | 192.168.1.103 |
| --------------- | ------------------ | ------------- |
| NameNode        | Secondary NameNode |               |
| ResourceManager |                    |               |
| NodeManager     | NodeManager        | NodeManager   |
| DataNode        | DataNode           | DataNode      |

## 时间同步

```bash
yum install chrony -y
firewall-cmd --add-service=ntp --permanent --zone=public
firewall-cmd --reload

systemctl enable chronyd --now

timedatectl set-timezone Asia/Shanghai

```

### 时间服务端配置

`/etc/chrony.conf`

```conf
server ntp1.aliyun.com iburst
server ntp2.aliyun.com iburst
server ntp3.aliyun.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
# 允许同步的 ip
allow 192.168.0.0/16
local stratum 10
logdir /var/log/chrony


stratumweight 0
#bindcmdaddress 127.0.0.1
#bindcmdaddress ::1
commandkey 1
generatecommandkey
noclientlog
logchange 0.5
```

### 时间客户端配置

```conf
server 192.168.1.101 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
logdir /var/log/chrony
```

## 防火墙配置

```bash
firewall-cmd --add-source=192.168.1.0/24 --permanent --zone=trusted
firewall-cmd --reload
```

## 配置 hosts

`/etc/hosts`

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.1.101 hadoop001 hadoop001.domain.com
192.168.1.102 hadoop002 hadoop002.domain.com
192.168.1.103 hadoop003 hadoop003.domain.com
```

## 服务间免密登录

```bash
# 各服务器添加普通用户
useradd hadoop
echo password | passwd hadoop --stdin

su - hadoop

ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys

ssh-copy-id 192.168.1.102
ssh-copy-id 192.168.1.103
```

## 配置 Hadoop

```bash
# 解压下载好的安装包
tar xaf /vagrant/hadoop-3.3.4.tar.gz -C /home/hadoop
cd /home/hadoop/hadoop-3.3.4
```

#### /home/hadoop/.bashrc

```bash
export HADOOP_HOME=/home/hadoop/hadoop-3.3.4
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

#### etc/hadoop/hadoop-env.sh

```bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0
export HADOOP_OS_TYPE=${HADOOP_OS_TYPE:-$(uname -s)}

# export HDFS_NAMENODE_USER=hadoop
# export HDFS_SECONDARYNAMENODE_USER=hadoop
# export HDFS_DATANODE_USER=hadoop
# export YARN_RESOURCEMANAGER_USER=hadoop
# export YARN_NODEMANAGER_USER=hadoop
```

#### etc/hadoop/core-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop001:8020</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/hadoop/data</value>
    </property>
</configuration>
```

#### etc/hadoop/hdfs-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop002:9868</value>
    </property>
</configuration>
```

#### etc/hadoop/mapred-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
</configuration>
```

#### etc/hadoop/yarn-site.xml

```xml
<?xml version="1.0"?>
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop001</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

#### etc/hadoop/workers

```
hadoop001
hadoop002
hadoop003
```

## 分发程序包

```bash
scp -r hadoop-3.3.4/ hadoop002:$PWD
scp -r hadoop-3.3.4/ hadoop003:$PWD
```

## 格式化文件系统

```bash
bin/hdfs namenode -format
```

成功提示信息

> Storage directory /home/hadoop/data/dfs/name has been successfully formatted.

## 启动 hdfs

```bash
source .bashrc

sbin/start-all.sh


# 单个组件启停
bin/hdfs --daemon start namenode

# hadoop002 节点执行
bin/hdfs --daemon start secondarynamenode

# 各节点执行
bin/hdfs --daemon start datanode

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
