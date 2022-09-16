# 在 aarch64 服务器上部署 HDFS-HA-QJM

[参考文档](https://hadoop.apache.org/docs/r3.3.4/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html)

阿里云购买的 arm 架构服务器

## 环境

操作系统：2C 4G , CentOS 7.9 (Linux hostname 4.18.0-193.28.1.el7.aarch64)

Java™：java-1.8.0-openjdk （ yum install -y java-1.8.0-openjdk-devel ）

远程连接：openssh （ yum install -y openssh ）

[hadoop-3.3.1.tar.gz](https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.3.1/hadoop-3.3.1-aarch64.tar.gz)

[zookeeper-3.4.5](https://obs-mirror-ftp4.obs.cn-north-4.myhuaweicloud.com/middleware/zookeeper-3.4.5.aarch64.tar.gz)

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
allow 172.16.0.0/16
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
server 172.16.16.100 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
logdir /var/log/chrony
```

## 防火墙配置

```bash
firewall-cmd --add-source=172.16.16.0/24 --permanent --zone=trusted
firewall-cmd --reload
```

## 配置 hosts

`/etc/hosts`

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

172.16.16.100 hadoop001 hadoop001.domain.com
172.16.16.101 hadoop002 hadoop002.domain.com
172.16.16.102 hadoop003 hadoop003.domain.com
```

## 服务器间免密登录

```bash
# 各服务器添加普通用户
useradd hadoop
echo password | passwd hadoop --stdin

# 提前创建好 zookeeper 目录
mkdir /var/lib/zookeeper
chown hadoop.hadoop /var/lib/zookeeper

su - hadoop

ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys

# 在 172.16.16.101 也执行
ssh-copy-id 172.16.16.100
ssh-copy-id 172.16.16.101
ssh-copy-id 172.16.16.102
```

## 安装 zookeeper

```bash
wget https://obs-mirror-ftp4.obs.cn-north-4.myhuaweicloud.com/middleware/zookeeper-3.4.5.aarch64.tar.gz

tar xaf zookeeper-3.4.5.aarch64.tar.gz

cat <<EOF | tee zookeeper-3.4.5/conf/zoo.cfg
tickTime=2000
dataDir=/var/lib/zookeeper/
clientPort=2181
initLimit=5
syncLimit=2
server.1=hadoop001:2888:3888
server.2=hadoop002:2888:3888
server.3=hadoop003:2888:3888
EOF

scp -r zookeeper-3.4.5 hadoop002:$PWD
scp -r zookeeper-3.4.5 hadoop003:$PWD


# id 根据配置文件修改
echo 1 > /var/lib/zookeeper/myid

~/zookeeper-3.4.5/bin/zkServer.sh start
~/zookeeper-3.4.5/bin/zkServer.sh status
```

## 配置 Hadoop

```bash
# 解压下载好的安装包
wget https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.3.1/hadoop-3.3.1-aarch64.tar.gz --no-check-certificate

tar xaf hadoop-3.3.1-aarch64.tar.gz -C /home/hadoop
```

#### /home/hadoop/.bashrc

```bash
export HADOOP_HOME=/home/hadoop/hadoop-3.3.1
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

#### etc/hadoop/hadoop-env.sh

```bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0
export HADOOP_OS_TYPE=${HADOOP_OS_TYPE:-$(uname -s)}
```

#### etc/hadoop/core-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://mycluster</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/hadoop/data</value>
    </property>
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/home/hadoop/data/journal/data</value>
    </property>
    <property>
       <name>dfs.ha.nn.not-become-active-in-safemode</name>
       <value>true</value>
    </property>
    <property> 
        <name>ha.zookeeper.quorum</name> 
        <value>hadoop001:2181,hadoop002:2181,hadoop003:2181</value> 
    </property>
</configuration>
```

#### etc/hadoop/hdfs-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>dfs.nameservices</name>
        <value>mycluster</value>
    </property>
    <property>
        <name>dfs.ha.namenodes.mycluster</name>
        <value>nn1, nn2</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.mycluster.nn1</name>
        <value>hadoop001:8020</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.mycluster.nn2</name>
        <value>hadoop002:8020</value>
    </property>
    <property>
        <name>dfs.namenode.http-address.mycluster.nn1</name>
        <value>hadoop001:9870</value>
    </property>
    <property>
        <name>dfs.namenode.http-address.mycluster.nn2</name>
        <value>hadoop002:9870</value>
    </property>
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://hadoop001:8485;hadoop002:8485/mycluster</value>
    </property>
    <property>
       <name>dfs.client.failover.proxy.provider.mycluster</name>
       <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
    <property>
        <name>dfs.ha.fencing.methods</name>
        <value>sshfence</value>
    </property>
    <property>
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/home/hadoop/.ssh/id_rsa</value>
    </property>
    <property>
        <name>dfs.ha.fencing.ssh.connect-timeout</name>
        <value>30000</value>
    </property>
    <property> 
        <name>dfs.ha.automatic-failover.enabled</name> 
        <value>true</value> 
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

参考文档

> https://hadoop.apache.org/docs/r3.3.4/hadoop-yarn/hadoop-yarn-site/ResourceManagerHA.html

```xml
<?xml version="1.0"?>
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>cluster1</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>hadoop001</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>hadoop002</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address.rm1</name>
        <value>hadoop001:8088</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address.rm2</name>
        <value>hadoop002:8088</value>
    </property>
    <property>
        <name>hadoop.zk.address</name>
        <value>hadoop001:2181,hadoop002:2181,hadoop003:2181</value>
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
cd ~
scp -r hadoop-3.3.1/ hadoop002:$PWD
scp -r hadoop-3.3.1/ hadoop003:$PWD
```

## 启动 journalnode

分别在三台主机执行

```bash
source ~/.bashrc
hdfs --daemon start journalnode
```

## 格式化文件系统

在 hadoop001 上执行并启动 namenode

```bash
hdfs namenode -format
hdfs --daemon start namenode
```

成功提示信息

> Storage directory /home/hadoop/data/dfs/name has been successfully formatted.

## 数据同步

在 hadoop002 执行数据同步命令

```bash
hdfs namenode -bootstrapStandby
```

## 初始化 zookeeper

```bash
hdfs zkfc -formatZK
```

成功提示信息

> Successfully created /hadoop-ha/mycluster in ZK.

## 启动 hdfs

```bash
start-all.sh

# 开启防火墙的端口访问
firewall-cmd --add-port=9870/tcp --permanent
firewall-cmd --reload
```

## 访问测试

ip:9870

功能性测试

```bash
hdfs dfs -mkdir -p /user/hadoop
hdfs dfs -mkdir input
hdfs dfs -put etc/hadoop/*.xml input
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar grep input output 'dfs[a-z.]+'
hdfs dfs -cat output/*
```

输出结果：

> 2       dfs.namenode.http
> 2       dfs.namenode.rpc
> 2       dfs.ha.fencing.methods
> 1       dfsadmin
> 1       dfs.server.namenode.ha.
> 1       dfs.nameservices
> 1       dfs.namenode.shared.edits.dir
> 1       dfs.journalnode.edits.dir
> 1       dfs.ha.nn.not
> 1       dfs.ha.namenodes.mycluster
> 1       dfs.ha.fencing.ssh.private
> 1       dfs.ha.fencing.ssh.connect
> 1       dfs.ha.automatic
> 1       dfs.client.failover.proxy.provider.mycluster
