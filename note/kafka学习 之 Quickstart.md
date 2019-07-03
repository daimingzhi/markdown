kafka学习 之 Quickstart

##### 第一步：安装启动kafka

官网链接：https://www.apache.org/dyn/closer.cgi?path=/kafka/2.3.0/kafka_2.11-2.3.0.tgz

1. `进入指定目录：cd /usr`

2. 创建文件夹：`mkdir kafka`
3. 进入目录：`cd kafka`
4. 下载：`wget http://mirror.bit.edu.cn/apache/kafka/2.0.0/kafka_2.11-2.0.0.tgz `
5. 解压：`tar -zxvf  kafka_2.12-2.3.0.tgz`
6. 进入kafka目录中：`cd kafka_2.12-2.3.0`

由于kafka依赖于zookeeper,所以我们必须先启动zk,这里我们可以直接使用跟kafka一起打包的一个脚本去启动一个应急的zk实例

7. `bin/zookeeper-server-start.sh config/zookeeper.properties`
8. 启动zk后，新开一个窗口，启动kafka服务器，`bin/kafka-server-start.sh config/server.properties`

到这里我们的kafka就已经安装并启动完毕了

##### 第二步：创建Topic

1. 创建一个没有备份只有一个分区的topic---test

```shell
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
```

2. 创建完后，我们可以查看一下

```shell
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
test
```

##### 第三步：创建生产者，发送消息

```shell
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
>this is a message
>this is a another message
```

##### 第四步：创建一个消费者

```shell
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
This is a message
This is another message
```

到这一步，我们可以看到消息能够正常的发送跟接收了

##### 第五步：创建多节点的集群

搭建一个三个节点的集群：

```shell
> cp config/server.properties config/server-1.properties
> cp config/server.properties config/server-2.properties	
```

编辑我们拷贝好的这两个文件：

```shell
vim -r /usr/kafka/kafka_2.12-2.3.0/config/server-1.properties
# 加入下面这段配置
broker.id=1
listeners=PLAINTEXT://:9093
log.dirs=/tmp/kafka-logs-1

vim -r /usr/kafka/kafka_2.12-2.3.0/config/server-2.properties
# 加入下面这段配置
broker.id=2
listeners=PLAINTEXT://:9094
log.dirs=/tmp/kafka-logs-2
```

`broker.id`是节点在集群中的唯一标志，我们之所以去覆盖端口跟日志目录是因为我们在同一台机器上启动了多个节点。

我们之前已经启动了zk跟一个单节点的kafka,接下来我们启动另外两个节点

```shell
bin/kafka-server-start.sh config/server-1.properties &
bin/kafka-server-start.sh config/server-2.properties &
```

创建一个三个复制因子的topic

```shell
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```

现在我们已经有了一个三个节点的集群，但是我们怎么知道哪个节点在做什么呢？

```shell
bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic my-replicated-topic	

Topic:my-replicated-topic       PartitionCount:1        ReplicationFactor:3     Configs:segment.bytes=1073741824
        Topic: my-replicated-topic      Partition: 0    Leader: 1       Replicas: 0,1,2 Isr: 0,1,2
```

**第一行总结了所有分区，每一行都给出了关于一个分区的信息。因为这个topic只有一个分区，所以只有一行。**

- “leader”是负责给定分区的所有读写的节点。每个节点都是分区中随机选择的一部分的领导者。
- “replicas”是复制这个分区日志的节点列表，无论它们是主节点还是当前活动节点。
- “isr”是一组“同步”副本。它是“replicas”的子集，并且是实时变动并被”leader“抓取的。

启动生产者发送消息：

```shell
 bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
>send
>send message
```

启动消费者接收消息：

```shell
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
send
send message
```

**测试容错性：**

集群当前的leader节点是1号节点，我们可以killi掉，然后再发送消息

```shell
ps -aux | grep server-1.properties
root      31132  0.0  0.0 112708   992 pts/4    S+   21:50   0:00 grep --color=auto server-2.properties
root      64547  0.6  3.6 6867572 606204 pts/1  Sl+  17:38   1:38 
```

```shell
 kill -9 64547
```

此时查看我们的集群状态:

```shell
 bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic my-replicated-topic
 
 Topic:my-replicated-topic       PartitionCount:1        ReplicationFactor:3     Configs:segment.bytes=1073741824
        Topic: my-replicated-topic      Partition: 0    Leader: 0       Replicas: 0,1,2 Isr: 0,2
```

可以看到在在同步的节点列表中（”Isr“），1号节点已经挂掉了，但是这个时候我们还是能正常的收发消息



