### kafka 工具学习（二）
```
 connect-distributed.sh
 connect-mirror-maker.sh
 connect-standalone.sh
 kafka-acls.sh
 kafka-broker-api-versions.sh
 kafka-configs.sh
 kafka-console-consumer.sh
 kafka-console-producer.sh
 kafka-consumer-groups.sh
 kafka-consumer-perf-test.sh
 kafka-delegation-tokens.sh
 kafka-delete-records.sh
 kafka-dump-log.sh
 kafka-leader-election.sh
 kafka-log-dirs.sh
 kafka-mirror-maker.sh
 kafka-preferred-replica-election.sh
 kafka-producer-perf-test.sh
 kafka-reassign-partitions.sh
 kafka-replica-verification.sh
 kafka-run-class.sh
 kafka-server-start.sh
 kafka-server-stop.sh
 kafka-streams-application-reset.sh
 kafka-topics.sh
 kafka-verifiable-consumer.sh
 kafka-verifiable-producer.sh
 nohup.out
 trogdor.sh
 zookeeper-security-migration.sh
 zookeeper-server-start.sh
 zookeeper-server-stop.sh
 zookeeper-shell.sh
```
### kafka-server-start.sh kafka服务启动
查看帮助文档
```
kafka-server-start.sh --help
```
kafka 服务启动 以nohup方式启动 使用server.properties配置文件启动
```
nohup ./kafka-server-start.sh ../config/server.properties
```
![20201230094943](https://github.com/weifangZ/image/blob/master/image20201230094943.png)

### kafka-server-stop.sh kafka服务停止

```
kafka-server-stop.sh 
```
kafka 服务停止
### kafka-streams-application-reset.sh用于给Kafka Streams应用程序重设位移，以便重新消费数据
1、启动steams程序
![20201230103227](https://github.com/weifangZ/image/blob/master/image20201230103227.png)
2、进行生产数据
![20201230104332](https://github.com/weifangZ/image/blob/master/image20201230104332.png)
3、重新获取reset数据
```
bin/kafka-streams-application-reset.sh --application-id my-streams-app --input-topics my-input-topic --intermediate-topics rekeyed-topic
```

### kafka-topics.sh topics的操作

1、创建topic
```
./kafka-topics.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --create --replication-factor 3 --partitions 1 --topic mcTrade
```
以上命令代表分区1副本3\
分区的原因：\
1、提高扩展性\
2、提高并发性

生产者的分区原则：\
1、指明对应的partition就操作对应的partition。\
2、没指明，但是有key时候使用key对topic的partition数量进行取余数，得到的值取操作相应的partition。\
3、没partition与没key时随机出一个整数，然后按照这个值递增，对topic的partition数量进行取余数，得到的值取操作相应的partition。\
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/3A82DC842F2A4675BCB4249E58168D00/23460)

2、查看topic
```
./kafka-topics.sh --list --bootstrap-server 192.168.131.128:9092
```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/E0BE9D583C454B139DA9D063172A9D59/23351)
3、删除topic
```
./kafka-topics.sh --delete --bootstrap-server 192.168.131.128:9092 --topic mcTrade
```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/55160F3A36DF4FC1A74A875D8CE46152/23361)


4、描述
```
./kafka-topics.sh --describe --bootstrap-server 192.168.131.128:9092 --topic mcTrade
```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/728A18674D094E0190A9E62E1A26CFE4/23368)

5、修改分区副本
```
./kafka-topics.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --alter --replication-factor 4 --partitions 2 --topic mcTrade
```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/C518E57A18184C6E850583D3ECA41B7C/23389)
变为
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/BF6B8AF12CA34CD99AA2E339BF998986/23393)

6、生产者写数据与消费者消费数据命令productor&consumer
```
//productor
./kafka-console-producer.sh --topic mcTrade --bootstrap-server 192.168.131.128:9092
//consumer
./kafka-console-consumer.sh --topic mcTrade --bootstr-server 192.168.131.128:9092 --from-beginning

```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/6E1598CEA6F149B29C120175C4E0BE6B/23377)



### kafka-verifiable-consumer.sh用于测试验证消费者功能
```
usage: verifiable-consumer [-h] --topic TOPIC --group-id GROUP_ID [--group-instance-id GROUP_INSTANCE_ID] [--max-messages MAX-MESSAGES]
                           [--session-timeout TIMEOUT_MS] [--verbose] [--enable-autocommit] [--reset-policy RESETPOLICY]
                           [--assignment-strategy ASSIGNMENTSTRATEGY] [--consumer.config CONFIG_FILE]
                           (--bootstrap-server HOST1:PORT1[,HOST2:PORT2[...]] | --broker-list HOST1:PORT1[,HOST2:PORT2[...]])
```
执行验证消费者功能
```
kafka-verifiable-consumer.sh --bootstrap-server 192.168.131.131:9092 --topic mcTrade --group-id zwf
```
![20201230112847](https://github.com/weifangZ/image/blob/master/image20201230112847.png)

### kafka-verifiable-producer.sh 用于测试验证生产者功能
```
usage: verifiable-producer [-h] --topic TOPIC [--max-messages MAX-MESSAGES] [--throughput THROUGHPUT] [--acks ACKS]
                           [--producer.config CONFIG_FILE] [--message-create-time CREATETIME] [--value-prefix VALUE-PREFIX]
                           [--repeating-keys REPEATING-KEYS] (--bootstrap-server HOST1:PORT1[,HOST2:PORT2[...]] |
                           --broker-list HOST1:PORT1[,HOST2:PORT2[...]])

```
执行验证生产者功能
```
kafka-verifiable-consumer.sh --bootstrap-server 192.168.131.131:9092 --topic mcTrade --group-id zwf
```
![20201230114223](https://github.com/weifangZ/image/blob/master/image20201230114223.png)

### zookeeper-security-migration.sh
```
zookeeper-security-migration --zookeeper.acl=secure --zookeeper.connection=localhost:2181

```
### zookeeper-server-start.sh 启动 zk 服务
### zookeeper-server-stop.sh停止 zk 服务
### zookeeper-shell.sh zk 客户端脚本   


其他：

1、如何扩展我们的集群

将新的服务器添加到Kafka集群，需为其分配唯一的 broker ID并在您的新服务器上启动Kafka即可。 zookeeper 不需要扩展吗？
```
# The id of the broker. This must be set to a unique integer for each broker.
broker.id=2

```
此时这些新的服务器不会自动分配到任何数据分区，除手动将消费者消费的分区指定到这些新增加的kafka分区，否则会等到创建新 topic 时才会提供服务。
报错1:
```
[2020-12-31 21:46:41,709] ERROR Fatal error during KafkaServer startup. Prepare to shutdown (kafka.server.KafkaServer)
kafka.common.InconsistentBrokerIdException: Configured broker.id 4 doesn't match stored broker.id 2 in meta.properties. If you moved your data, make sure your configured broker.id matches. If you intend to create a new broker, you should remove all data in your data directories (log.dirs).
	at kafka.server.KafkaServer.getOrGenerateBrokerId(KafkaServer.scala:767)
	at kafka.server.KafkaServer.startup(KafkaServer.scala:226)
	at kafka.server.KafkaServerStartable.startup(KafkaServerStartable.scala:44)
	at kafka.Kafka$.main(Kafka.scala:82)
	at kafka.Kafka.main(Kafka.scala)

```

这种情况只有在一台机器上部署两个broker服务时才有可能发生，而且是server.properties中的log.dir配置采用默认配置/tmp/kafka-logs。删除就行。

错误2：
```
[2020-12-31 22:05:14,347] ERROR [KafkaServer id=4] Fatal error during KafkaServer startup. Prepare to shutdown (kafka.server.KafkaServer)
org.apache.kafka.common.KafkaException: Socket server failed to bind to 192.168.131.129:9092: Cannot assign requested address.

```

解决办法，配置文件 server.properties  listeners=PLAINTEXT://192.168.131.128:9092  192.168.131.133 请不要写死地址

但是问题来了，新添加的Kafka节点并不会自动地分配数据，所以无法分担集群的负载，除非我们新建一个topic。但是现在我们想手动将部分分区移到新添加的Kafka节点上，Kafka内部提供了相关的工具来重新分布某个topic的分区。在重新分布topic分区之前，我们先来看看现在topic的各个分区的分布位置：

```
[zwf@clear-node-3 bin]$ ./kafka-topics.sh --describe --bootstrap-server 192.168.131.128:9092 --topic mcTrade
Topic: mcTrade	PartitionCount: 1	ReplicationFactor: 3	Configs: segment.bytes=1073741824,file.delete.delay.ms=3000
	Topic: mcTrade	Partition: 0	Leader: 1	Replicas: 1,2,3	Isr: 2,3,1
```
新增一个topic看看：
```
[zwf@clear-node-3 bin]$ ./kafka-topics.sh --create --bootstrap-server 192.168.131.128:9092 --topic mcTradenew
Created topic mcTradenew.
[zwf@clear-node-3 bin]$ ./kafka-topics.sh --describe --bootstrap-server 192.168.131.128:9092 --topic mcTradenew
Topic: mcTradenew	PartitionCount: 1	ReplicationFactor: 3	Configs: segment.bytes=1073741824,file.delete.delay.ms=3000
	Topic: mcTradenew	Partition: 0	Leader: 1	Replicas: 1,4,2	Isr: 1,4,2
[zwf@clear-node-3 bin]$ 

```

如何将mcTrade 的数据重新分配数据呢？

使用Kafka自带的kafka-reassign-partitions.sh工具来重新分布分区。该工具有三种使用模式：

　　1、generate模式，给定需要重新分配的Topic，自动生成reassign plan（并不执行）
　　
　　2、execute模式，根据指定的reassign plan重新分配Partition
　　
　　3、verify模式，验证重新分配Partition是否成功
　　
　　
　　
```
kafka-reassign-partitions.sh --zookeeper 192.168.131.128:2181,192.168.131.129:2181,192.168.131.131:2181/clear/kafka_2.12-2.5.0 --topics-to-move-json-file topics-to-move.json --broker-list "1,2,3,4" --generate
```
topics-to-move.json
```
{"topics": [{"topic": "mcTrade"}],
 "version":1
}
```
结果如下：
```
[zwf@clear-node-3 bin]$ kafka-reassign-partitions.sh --zookeeper 192.168.131.128:2181,192.168.131.129:2181,192.168.131.131:2181/clear/kafka_2.12-2.5.0 --topics-to-move-json-file topics-to-move.json --broker-list "1,2,3,4" --generate
Current partition replica assignment
{"version":1,"partitions":[{"topic":"mcTrade","partition":0,"replicas":[1,2,3],"log_dirs":["any","any","any"]}]}

Proposed partition reassignment configuration
{"version":1,"partitions":[{"topic":"mcTrade","partition":0,"replicas":[1,4,2],"log_dirs":["any","any","any"]}]}

```

execute 数据迁移

```
kafka-reassign-partitions.sh --zookeeper 192.168.131.128:2181,192.168.131.129:2181,192.168.131.131:2181/clear/kafka_2.12-2.5.0 --reassignment-json-file result.json --execute
```

```
{"version":1,"partitions":[{"topic":"mcTrade","partition":0,"replicas":[1,2,3],"log_dirs":["any","any","any"]}]}
```

出现问题：

```
[zwf@clear-node-3 bin]$ kafka-reassign-partitions.sh --zookeeper 192.168.131.131:2181 --reassignment-json-file result.json --execute
Partitions reassignment failed due to The proposed assignment contains non-existent partitions: ListBuffer(mcTrade-0)
kafka.common.AdminCommandFailedException: The proposed assignment contains non-existent partitions: ListBuffer(mcTrade-0)
	at kafka.admin.ReassignPartitionsCommand$.parseAndValidate(ReassignPartitionsCommand.scala:342)
	at kafka.admin.ReassignPartitionsCommand$.executeAssignment(ReassignPartitionsCommand.scala:209)
	at kafka.admin.ReassignPartitionsCommand$.executeAssignment(ReassignPartitionsCommand.scala:205)
	at kafka.admin.ReassignPartitionsCommand$.main(ReassignPartitionsCommand.scala:65)
	at kafka.admin.ReassignPartitionsCommand.main(ReassignPartitionsCommand.scala)

```
原因为：
```
--zookeeper 192.168.131.131:218 改为192.168.131.128:2181,192.168.131.129:2181,192.168.131.131:2181/clear/kafka_2.12-2.5.0
```
解决后成功：
```
[zwf@clear-node-3 bin]$ kafka-reassign-partitions.sh --zookeeper 192.168.131.128:2181,192.168.131.129:2181,192.168.131.131:2181/clear/kafka_2.12-2.5.0 --reassignment-json-file result.json --execute
Current partition replica assignment

{"version":1,"partitions":[{"topic":"mcTrade","partition":0,"replicas":[1,2,3],"log_dirs":["any","any","any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started reassignment of partitions.

```

verify 数据迁移验证
```
kafka-reassign-partitions.sh --zookeeper 192.168.131.128:2181,192.168.131.129:2181,192.168.131.131:2181/clear/kafka_2.12-2.5.0 --reassignment-json-file result.json --verify
```

结果如下：

```
[zwf@clear-node-3 bin]$ kafka-reassign-partitions.sh --zookeeper 192.168.131.128:2181,192.168.131.129:2181,192.168.131.131:2181/clear/kafka_2.12-2.5.0 --reassignment-json-file result.json --verify
Status of partition reassignment: 
Reassignment of partition mcTrade-0 completed successfully

```

![20210101185600](https://github.com/weifangZ/image/blob/master/image20210101185600.png)
