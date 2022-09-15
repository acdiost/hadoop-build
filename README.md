# 单节点模式


启动服务器

`cd vagrant`

`vagrant up`


## 环境

操作系统：CentOS 7.9 (Linux hostname 3.10.0-1160.25.1.el7.x86_64)

Java™：java-1.8.0-openjdk （ yum install -y java-1.8.0-openjdk ）

远程连接：openssh （ yum install -y openssh ）

Hadoop-3.3.4 ( https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.3.4/hadoop-3.3.4.tar.gzhttps://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.3.4 )


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

# 测试功能是否正常
mkdir input
cp etc/hadoop/*.xml input
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.4.jar grep input output 'dfs[a-z.]+'
cat output/*

# 输出结果
$> 1       dfsadmin
```
