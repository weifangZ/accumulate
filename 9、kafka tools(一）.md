## kafka tools(一）
目前kafka 自带tools已经越来越丰富了。
不仅可以直接使用这里面的工具，还可以对其封装否自定义使用。

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



参考文献

1、[脚本kafka-configs.sh用法解析](https://www.cnblogs.com/lizherui/p/12275193.html)
