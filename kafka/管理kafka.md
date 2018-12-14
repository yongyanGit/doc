### 管理kafka

#### 1. 创建topic

创建主题需要指定三个参数

主题名称：想要创建的主题名称。主题的名字开头包含两个下划线是合法的，但是不建议这么做，具有这种格式的主题一般都是集群内部主题(比如：_consumer_offsets主题，用来保存消费者群组的偏移量)。如果主题名称中含有句号，当被用在度量指标上，句号会被替换成下划线，所以也不推荐名字中使用句号。

复制系数：主题的副本数量

分区：主题的分区数量

命令如下：

```
./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test2
```

#### 2. 增加分区

增加分区主要是为了扩展主题容量或者降低单个分区的吞吐量。

```
./kafka-topics.sh --zookeeper localhost:2181 --alter --topic test2 --partitions 3
```

#### 3. 删除主题

如果主题不再使用，只有它还存在于集群中，就会占用一定数量的磁盘空间和文件句柄。为了能够删除主题，broker 的delete.topic.enable参数必须设置为true，如果该参数被设置为false，删除主题的请求会被忽略。

```
./kafka-topics.sh --zookeeper localhost:2181 --delete --topic test2
```

#### 4. 列出所有主题

```
./kafka-topics.sh --zookeeper localhost:2181 --list
```

获取主题的详细信息，不指定topic 就列出所有的主题信息

```
./kafka-topics.sh --zookeeper localhost:2181 --describe --topic test
```



#### 5. 查看消费者组情况

```
./kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
```

可以使用describe来代替--list查看群组详细信息。



