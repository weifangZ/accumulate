
### kafka 工具（三）
经过分析kafka提供的工具都是根据
```
kafka-run-class.sh
```
基础上进行处理。
所以我们学习kafka工具时候我们只需要看懂
```
kafka-run-class.sh
```
就可以了。
![1610169856(1)](https://github.com/weifangZ/image/blob/master/image1610169856(1).png)
1.kafka-topic.sh
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.TopicCommand "$@"
```

2.kafka-console-consumer.sh
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleConsumer "$@"
# Add jaas file
export KAFKA_OPTS="-Djava.security.auth.login.config=/home/zwf/clear/kafka_2.12-2.5.0/config
/kafka_client_jaas.conf"
```

3.kafka-console-producer.sh
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleProducer "$@"
# Add jaas file
export KAFKA_OPTS="-Djava.security.auth.login.config=/home/zwf/clear/kafka_2.12-2.5.0/config
/kafka_client_jaas.conf"

```
4.kafka-dump-log.sh
```
exec $(dirname $0)/kafka-run-class.sh kafka.tools.DumpLogSegments "$@"

```
5.connect-distributed.sh 

```
exec $(dirname $0)/kafka-run-class.sh $EXTRA_ARGS org.apache.kafka.connect.cli.ConnectDistributed "$@"

```
6.connect-mirror-maker.sh
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
7.connect-standalone.sh 
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
8.kafka-acls.sh 
```
xec $(dirname $0)/kafka-run-class.sh kafka.admin.AclCommand "$@"
```
9.kafka-broker-api-versions.sh 
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.BrokerApiVersionsCommand "$@"
```
10.kafka-configs.sh 
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.ConfigCommand "$@"
```
11.kafka-console-consumer.sh
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi

exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleConsumer "$@"
```
12.kafka-console-producer.sh 
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleProducer "$@"
```
13.kafka-consumer-groups.sh
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.ConsumerGroupCommand "$@"
```
14.kafka-consumer-perf-test.sh
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsumerPerformance "$@"
```
15.kafka-delegation-tokens.sh 
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.DelegationTokenCommand "$@"
```
16.kafka-delete-records.sh
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.DeleteRecordsCommand "$@"
```
17.kafka-leader-election.sh 
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.LeaderElectionCommand "$@"
```
18.kafka-log-dirs.sh
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.LogDirsCommand "$@"
```
19.kafka-preferred-replica-election.sh 
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.PreferredReplicaLeaderElectionCommand "$@"
```

20.kafka-producer-perf-test.sh 
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh org.apache.kafka.tools.ProducerPerformance "$@"
```
21.kafka-reassign-partitions.sh
```
exec $(dirname $0)/kafka-run-class.sh kafka.admin.ReassignPartitionsCommand "$@"
```
22.kafka-replica-verification.sh
```
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ReplicaVerificationTool "$@"
```
23.kafka-server-start.sh 
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
24.kafka-streams-application-reset.sh
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi

exec $(dirname $0)/kafka-run-class.sh kafka.tools.StreamsResetter "$@"

```
25.kafka-verifiable-consumer.sh
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh org.apache.kafka.tools.VerifiableConsumer "$@"

```
26.kafka-verifiable-producer.sh
```
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh org.apache.kafka.tools.VerifiableProducer "$@"
```
