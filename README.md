# hadoop-build

构建 Hadoop 的初始化配置，详情见各分支；不含优化内容。

### 分支列表

fully-distributed
hdfs-ha-qjm
hdfs-ha-qjm-aarch64
hdfs-ha-qjm-hbase
hdfs-ha-qjm-hive
main
pseudo-distributed
single-mode

## 预备知识

hadoop 生态及其相关组件的知识

Linux

MySQL

zookeeper

---

## hadoop 配置补充

### core-site.xml

指定浏览器页面使用的账户

```xml
    <property>
        <name>hadoop.http.staticuser.user</name>
        <value>hadoop</value>
    </property>
```

开启垃圾桶机制，保留一个月，单位分钟

```xml
    <property>
        <name>fs.trash.interval</name>
        <value>43200</value>
    </property>
    <property>
        <name>fs.trash.checkpoint.interval</name>
        <value>1440</value>
    </property>
```


## TODO

自启动

监控告警

节点的新增与删除

透明加密

磁盘
