


``` shell
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

1、connect-standalone.sh && connect-distributed.sh
connect-standalone.sh为单机的命令格式，connect-distributed.sh为集群命令格式。

2、 kafka-acls.sh
实际：
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.AclCommand "$@"
```

#### 概述
ACL是一种kafka 权限控制的工具，在处理一些核心的业务数据时，Kafka的ACL机制还是非常重要的，对核心业务主题进行权限管控，能够避免不必要的风险。

#### 身份认证
Kafka的认证范围包含如下：

Client与Broker之间
Broker与Broker之间
Broker与Zookeeper之间
当前Kafka系统支持多种认证机制，如SSL、SASL（Kerberos、PLAIN、SCRAM）。

##### SASL认证流程
配置流程
1、创建证书
```
kafka-configs.sh --bootstrap-server clear-node-2:9092 --alter --add-config 'SCRAM-SHA-256=[iterations=8192,password=alice-secret],SCRAM-SHA-512=[password=alice-secret]' --entity-type users --entity-name alice
kafka-configs.sh --bootstrap-server clear-node-2:9092 --alter --add-config 'SCRAM-SHA-256=[password=admin-secret],SCRAM-SHA-512=[password=admin-secret]' --entity-type users --entity-name admin
```

2、验证证书
```
kafka-configs.sh --bootstrap-server clear-node-2:9092 --describe --entity-type users --entity-name alice

```

问题：
```
could not find a ‘kafkaserver’ or ‘sasl_plaintext.kafkaserver’ entry in the jaas configuration
```
解决方法：

Kafka启动脚本中加入配置，读取第一步创建的文件,kafka_server_jaas.conf

修改kafka的kafka-server-start.sh文件，

在如下代码
```
export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
```
添加
```
 export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G -Djava.security.auth.login.config=/opt/kafka_2.11-1.1.1/config/kaf
```
在配置好SASL后，启动Zookeeper集群和Kafka集群之后，就可以使用kafka-acls.sh脚本来操作ACL机制。

　　（1）查看：在kafka-acls.sh脚本中传入list参数来查看ACL授权新
```
[hadoop@dn1 bin]$ kafka-acls.sh --list --authorizer-properties zookeeper.connect=clear-node-4:2181
```
　　（2）创建：创建待授权主题之前，在kafka-acls.sh脚本中指定JAAS文件路径，然后在执行创建操作
```
kafka-topics.sh --create --bootstrap-server clear-node-4:9092 --replication-factor 1 --partitions 1 --topic kafka_acl_topic
```
　　（3）生产者授权：对生产者执行授权操作
　　
```
[hadoop@dn1 ~]$ kafka-acls.sh --authorizer kafka.security.auth.SimpleAclAuthorizer --authorizer-properties zookeeper.connect=clear-node-4:2181 --add --allow-principal User: producer --operation Write --topic kafka_acl_topic
```
　　（4）消费者授权：对生产者执行授权后，通过消费者来进行验证
```
[hadoop@dn1 ~]$ kafka-acls.sh --authorizer kafka.security.auth.SimpleAclAuthorizer --authorizer-properties zookeeper.connect=clear-node-4:2181 --add --allow-principalUser:consumer --operation Read --topic kafka_acl_topic
```
　　（5）删除：通过remove参数来回收相关权限
```
[hadoop@dn1 bin]$    kafka-acls.sh --authorizer-properties zookeeper.connect=clear-node-4:2181 --remove --allow-principal User:producer --operation Write --topic kafka_acl_topic3
```

kafka-config.sh

![560365280f39d0516da9dd3a0a12f250_273782-20200207233722724-2083026365](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image560365280f39d0516da9dd3a0a12f250_273782-20200207233722724-2083026365.png)

语法格式：
某个topic配置对象
```
./kafka-configs.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092  --alter --entity-type topics --entity-name mcTrade  --add-config unclean.leader.election.enable=true

```
![20201217191642](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20201217191642.png)

删除配置项
```
./kafka-configs.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092  --alter --entity-type topics --entity-name mcTrade  --delete-config unclean.leader.election.enable=true

```
![20201217191830](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20201217191830.png)

列出entity配置描述
```
./kafka-configs.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092  --entity-type topics --entity-name mcTrade --describe

```
![20201217192117](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20201217192117.png)

可以进行配置限额的设置：

kafka支持配额管理，从而可以对Producer和Consumer的produce&fetch操作进行流量限制，防止个别业务压爆服务器。

配额限流简介

Kafka配额限流由3种粒度配置：

- users + clients
- users
- clients


kafka-broker-api-versions.sh 版本信息
```
./kafka-broker-api-versions.sh –bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092
```
![20201217194027](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20201217194027.png)

kafka-consumer-groups.sh 消费者组

消费组 描述
```
kafka-consumer-groups.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --all-groups --describe
```
![20201217194333](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20201217194333.png)

消费组 描述
```
kafka-consumer-groups.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --all-groups --describe
```
查看有那些 group ID 正在进行消费：

```
kafka-consumer-groups.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --group all-zwf2 --describe
```
![20201217195609](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20201217195609.png)


### ./kafka-delete-records.sh 删除低水位的日志文件
1、删除数据
```
./kafka-delete-records.sh --bootstrap-server 192.168.131.131:9092,192.168.131.129:9092,192.168.131.128:9092 --offset-json-file mcTrade.json 
```
mcTrade.json
```
{
    "partitions": [{
        "topic": "mcTrade",
        "partition": 0,
        "offset": 9
    }],
    "version": 1
}
```
![20201223171339](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20201223171339.png)
查看对应的offset
![20201223171415](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20201223171415.png)
发现数据时删除了9条，但是offset并没有降低，说明这个删除只是delete 操作，通过low_watermark: 9这样的进行的操作。

此时"offset": 9这里的9不能高于topic的offset，否则会报错，所以在使用这个命令时，需要知道offset位置。

### kafka-log-dirs.sh kafka消息日志目录信息

```
kafka-log-dirs.sh --bootstrap-server 192.168.131.131:9092 --topic-list mcTrade --describe

```
![20201223191726](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20201223191726.png)

### kafka-mirror-maker.sh 不同数据中心kafka集群复制工具

```
kafka-mirror-maker.sh 

Option                                   Description                           
------                                   -----------                           
--abort.on.send.failure <String: Stop    Configure the mirror maker to exit on 
  the entire mirror maker when a send      a failed send.（默认为true，决定生产者写入失败时的处理机制） (default: true)      
  failure occurs>                                                              
--consumer.config <String: config file>  Embedded consumer config for consuming
                                           from the source cluster（用于指定消费者的配置文件，配置文件里有两个必填的参数：bootstrap.servers 和 group.id）.            
--consumer.rebalance.listener <String:   The consumer rebalance listener to use
  A custom rebalance listener of type      for mirror maker consumer.（指定再均衡监听器）          
  ConsumerRebalanceListener>                                          
--help                                   Print usage information.              
--message.handler <String: A custom      Message handler which will process    
  message handler of type                  every record in-between consumer and
  MirrorMakerMessageHandler>               producer.（指定消息的处理器。这个处理器会在消费者消费到消息之后且在生产者发送消息之前被调用）                           
--message.handler.args <String:          Arguments used by custom message      
  Arguments passed to message handler      handler for mirror maker.（指定消息处理器的参数，同message.handler一起使用）           
  constructor.>                                                                
--new.consumer                           DEPRECATED Use new consumer in mirror 
                                           maker (this is the default so this  
                                           option will be removed in a future  
                                           version).                           
--num.streams <Integer: Number of        Number of consumption streams. （指定消费线程的数量）       
  threads>                                 (default: 1)                        
--offset.commit.interval.ms <Integer:    Offset commit interval in ms.         
  offset commit interval in                (default: 60000) （指定消费位移提交间隔）                   
  millisecond>                                                                 
--producer.config <String: config file>  Embedded producer config.（用于指定生产者的配置文件，配置文件里唯一必填的参数是 bootstrap.servers）             
--rebalance.listener.args <String:       Arguments used by custom rebalance    
  Arguments passed to custom rebalance     listener for mirror maker consumer. （指定再均衡监听器的参数，同consumer.rebalance.listener一起使用）
  listener constructor as a string.>                                           
--version                                Display Kafka version.                
--whitelist <String: Java regex          Whitelist of topics to mirror. （whitelist 指定需要复制的源集群中的主题。这个参数可以指定一个正则表达式，比如a|b表示复制源集群中主题a和主题b的数据。为了方便使用，这里也允许将“|”替换为“,”）       
  (String)>                                                                 
```
一、MirrorMaker介绍

MirrorMaker是Kafka附带的一个用于在Kafka集群之间制作镜像数据的工具。**该工具从源集群中消费并生产到目标群集**。这种镜像的常见用例是在另一个数据中心提供副本。
其实现原理是通过从源集群中消费消息，然后将消息生产到目标集群中，也就是普通的生产和消费消息。
用户只需要在启动Kafka Mirror Maker时指定一些简单的消费端和生产端配置就可以实现准实时的数据同步。

三、kafka-mirror-maker.sh脚本使用

3.1、先决条件

首先目标集群的kafka服务端必须开启 auto.create.topics.enable=true ，以允许自动创建topic；或者对于需要进行镜像的topic都自己手动进行创建。自动创建的topic会根据服务端配置的num.partitions、default.replication.factor决定。

3.2、演示使用

演示从集群1中将主题test-perf的数据同步到集群2中，首先创建并配置两个配置文件，参考如下：
```
#consumer.properties的配置
bootstrap.servers=kafka1:9092
group.id=groupIdMirror
client.id=sourceMirror
partition.assignment.strategy=org.apache.kafka.clients.consumer.RoundRobinAsignor
```
```
#producer.properties的配置
bootstrap.servers=kafka2:9092
client.id=sinkMirror
```
consumer.properties和producer.properties这两个配置文件中的配置对应消费者客户端和生产者客户端的配置。

下面启动Kafka Mirror Maker：
```
./kafka-mirror-maker.sh --consumer.config ../config/consumer.properties --producer.config ../config/producer.properties --whitelist 'test-perf' --num.streams 6
```

3.3、使用注意事项

kafka mirror maker中对于生产者写入失败时，会根据abort.on.send.failure的配置分两种处理方法；
abort.on.send.failure=true ，此时如果生产者经过多次重试依然无法完成消息写入，则会直接停止kafka mirror maker进程；
abort.on.send.failure=false ，此时如果生产者经过多次重试依然无法完成消息写入，则会直接跳过当前写入的批次数据，直接进行下一轮消息的写入；

源集群和目标集群是两个完全独立的实体，对每个主题而言，两个集群之间的分区数可能不同；就算分区数相同，那么经过消费再生产之后消息所规划到的分区号也有可能不同；就算分区数相同，消息所规划到的分区号也相同，那么消息所对应的offset也有可能不相同，例如，源集群中由于执行了某次日志清理操作，某个分区的logStartOffset值变为10，而目标集群中对应分区的logStartOffset还是0，那么从源集群中原封不动的复制到目标集群时，同一条消息的offset也会不相同。

参考文献

1、[脚本kafka-configs.sh用法解析](https://www.cnblogs.com/lizherui/p/12275193.html)

2、[kafka-mirror-maker.sh脚本](https://blog.csdn.net/qq_41154944/article/details/108282641)
