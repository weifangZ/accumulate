
### kafka 工具（三）
经过分析kafka提供的工具都是根据kafka-run-class.sh基础上进行处理。所以我们学习kafka工具时候我们只需要看懂kafka-run-class.sh就可以了。
以下工具都是对kafka-run-class.sh的封装
![1610169856(1)](https://github.com/weifangZ/image/blob/master/image1610169856(1).png)

1.kafka-topic.sh  topic 相关操作运行的kafka.admin.TopicCommand进行处理的
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.TopicCommand "$@"
```

2.kafka-console-consumer.sh 控制台进行消费数据kafka.tools.ConsoleConsumer使用这个类进行处理
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleConsumer "$@"
# Add jaas file
export KAFKA_OPTS="-Djava.security.auth.login.config=/home/zwf/clear/kafka_2.12-2.5.0/config
/kafka_client_jaas.conf"
```

3.kafka-console-producer.sh 消费者控制台命令 调用这个类kafka.tools.ConsoleProducer进行处理 通过学习kafka源码会发现kafka 提供的工具是这么处理这些命令的。
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleProducer "$@"
# Add jaas file
export KAFKA_OPTS="-Djava.security.auth.login.config=/home/zwf/clear/kafka_2.12-2.5.0/config
/kafka_client_jaas.conf"

```
4.kafka-dump-log.sh 使用DumpLogSegments类进行处理数据dump导出功能
```
exec $(dirname $0)/kafka-run-class.sh kafka.tools.DumpLogSegments "$@"

```
5.connect-distributed.sh  启动分布式模式使用。

```
exec $(dirname $0)/kafka-run-class.sh $EXTRA_ARGS org.apache.kafka.connect.cli.ConnectDistributed "$@"

```
6.connect-mirror-maker.sh  Kafka跨集群迁移方案 
```
if [ $# -lt 1 ];
then
        echo "USAGE: $0 [-daemon] mm2.properties"
        exit 1
fi

base_dir=$(dirname $0)

if [ "x$KAFKA_LOG4J_OPTS" = "x" ]; then
    export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:$base_dir/../config/connect-log4j.properties"
fi

if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
  export KAFKA_HEAP_OPTS="-Xms256M -Xmx2G"
fi

EXTRA_ARGS=${EXTRA_ARGS-'-name mirrorMaker'}

COMMAND=$1
case $COMMAND in
  -daemon)
    EXTRA_ARGS="-daemon "$EXTRA_ARGS
    shift
    ;;
  *)
    ;;
esac

exec $(dirname $0)/kafka-run-class.sh $EXTRA_ARGS org.apache.kafka.connect.mirror.MirrorMaker "$@"
```
7.connect-standalone.sh  对应集群命令，是执行单机模式的命令
```
if [ $# -lt 1 ];
then
        echo "USAGE: $0 [-daemon] connect-standalone.properties"
        exit 1
fi

base_dir=$(dirname $0)

if [ "x$KAFKA_LOG4J_OPTS" = "x" ]; then
    export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:$base_dir/../config/connect-log4j.properties"
fi

if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
  export KAFKA_HEAP_OPTS="-Xms256M -Xmx2G"
fi

EXTRA_ARGS=${EXTRA_ARGS-'-name connectStandalone'}

COMMAND=$1
case $COMMAND in
  -daemon)
    EXTRA_ARGS="-daemon "$EXTRA_ARGS
    shift
    ;;
  *)
    ;;
esac

exec $(dirname $0)/kafka-run-class.sh $EXTRA_ARGS org.apache.kafka.connect.cli.ConnectStandalone "$@"
```
8.kafka-acls.sh kafka 权限设置
```
xec $(dirname $0)/kafka-run-class.sh kafka.admin.AclCommand "$@"
```
9.kafka-broker-api-versions.sh  查看broker API版本信息
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.BrokerApiVersionsCommand "$@"
```
10.kafka-configs.sh  增加kafka 信息 配置信息
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.ConfigCommand "$@"
```
11.kafka-console-consumer.sh 消费者 控制台 数据指令
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi

exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleConsumer "$@"
```
12.kafka-console-producer.sh 生产者 控制台 数据指令
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleProducer "$@"
```
13.kafka-consumer-groups.sh 消费者组 控制台 数据指令
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.ConsumerGroupCommand "$@"
```
14.kafka-consumer-perf-test.sh 性能测试指令
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsumerPerformance "$@"
```
15.kafka-delegation-tokens.sh  kafka-delegation-tokens.sh用于管理Delegation Token
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.DelegationTokenCommand "$@"
```
16.kafka-delete-records.sh 按照offset 删除topic 数据指令
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.DeleteRecordsCommand "$@"
```
17.kafka-leader-election.sh  重新选举新的leader
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.LeaderElectionCommand "$@"
```
18.kafka-log-dirs.sh 数据路径管理
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.LogDirsCommand "$@"
```
19.kafka-preferred-replica-election.sh   副本重定向指令
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.PreferredReplicaLeaderElectionCommand "$@"
```

20.kafka-producer-perf-test.sh  生产者性能测试，功能测试
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh org.apache.kafka.tools.ProducerPerformance "$@"
```
21.kafka-reassign-partitions.sh 重新分配新增的kafka节点信息指令
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.ReassignPartitionsCommand "$@"
```
22.kafka-replica-verification.sh 分配后进行验证
```
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ReplicaVerificationTool "$@"
```
23.kafka-server-start.sh  服务器启动
```
if [ $# -lt 1 ];
then
        echo "USAGE: $0 [-daemon] server.properties [--override property=value]*"
        exit 1
fi
base_dir=$(dirname $0)

if [ "x$KAFKA_LOG4J_OPTS" = "x" ]; then
    export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:$base_dir/../config/log4j.properties"
fi

if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
fi

EXTRA_ARGS=${EXTRA_ARGS-'-name kafkaServer -loggc'}

COMMAND=$1
case $COMMAND in
  -daemon)
    EXTRA_ARGS="-daemon "$EXTRA_ARGS
    shift
    ;;
  *)
    ;;
esac

exec $base_dir/kafka-run-class.sh $EXTRA_ARGS kafka.Kafka "$@"

```
24.kafka-streams-application-reset.sh stream 流式计算应用重新执行
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi

exec $(dirname $0)/kafka-run-class.sh kafka.tools.StreamsResetter "$@"

```
25.kafka-verifiable-consumer.sh 消费者验证
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh org.apache.kafka.tools.VerifiableConsumer "$@"

```
26.kafka-verifiable-producer.sh 生产者验证
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh org.apache.kafka.tools.VerifiableProducer "$@"
```

### kafka通信协议 protobuf

由于我们清算中使用了 protocol buf协议传输数据，所以学习一下。

#### 安装
![20210109173801](https://github.com/weifangZ/image/blob/master/image20210109173801.png)
```
wget https://github.com/google/protobuf/releases/download/v3.6.1/protobuf-all-3.6.1.tar.gz 
tar zxvf protobuf-all-3.6.1.tar.gz
cd protobuf-3.6.1
./configure
make
make install
protoc   -h
protoc --version
```
![20210109193811](https://github.com/weifangZ/image/blob/master/image20210109193811.png)
```
[zwf@clear-node-4 protobuf-3.6.1]$ /usr/local/bin/protoc --version
libprotoc 3.6.1
```
安装 composer
```
curl -sS https://getcomposer.org/installer | /usr/local/php-7.0.14/bin/php
```
接下来拷贝到可执行文件目录/usr/local/bin目录
```
mv composer.phar /usr/local/bin/composer
```
![20210109232207](https://github.com/weifangZ/image/blob/master/image20210109232207.png)

3.composer在项目中引入protobuf：
新建文件夹app,然后在app文件夹内新建composer.json文件，文件内容如下：
```
{
    "require":{
        "google/protobuf": "^3.6.1"
    }
}
```
保存之后，在app文件夹下执行composer install安装命令:
```
composer install
ls (查看文件内容)
composer.json  composer.lock  vendor （表示安装成功）
```


![20210109235536](https://github.com/weifangZ/image/blob/master/image20210109235536.png)

编译好proto文件后

![20210110115553](https://github.com/weifangZ/image/blob/master/image20210110115553.png)

就可以在java、cpp等项目工程去使用了。
![20210110115741](https://github.com/weifangZ/image/blob/master/image20210110115741.png)

这样一来就知道了我们项目中proto文件是这么编译成为java文件后进行使用的了。
