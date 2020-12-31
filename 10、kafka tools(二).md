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
![20201230094943](https://github.com/weifangZ/accumulate/image/image20201230094943.png)

### kafka-server-stop.sh kafka服务停止

```
kafka-server-stop.sh 
```
kafka 服务停止
### kafka-streams-application-reset.sh用于给Kafka Streams应用程序重设位移，以便重新消费数据
1、启动steams程序
![20201230103227](https://github.com/weifangZ/accumulate/image/image20201230103227.png)
2、进行生产数据
![20201230104332](https://github.com/weifangZ/accumulate/image/image20201230104332.png)
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
![20201230112847](https://github.com/weifangZ/accumulate/image/image20201230112847.png)

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
![20201230114223](https://github.com/weifangZ/accumulate/image/image20201230114223.png)

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
此时这些新的服务器不会自动分配到任何数据分区，除手动将消费者消费的分区指定到这些新增加的kafka分区，否则会等到创建新 topic 时才会提供服务
