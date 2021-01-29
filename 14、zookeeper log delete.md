# kafka 运维学习手册2

### 一、遇到无法将期权数据进行正确的消费的情况。
```
报错：method threw 'java.lang.NoclassDefFoundError' exception. Cannot evlauate com.dce.common.bean.realTime.protobuf.OrderOptPb$OrderOpt.toString()
```

初步判断为optOrderPb无法接受到数据。

通过调查发现该错误不会影响正常消费者消费数据的，是一种强制类型转换导致的错误，无需关心。



### 二、Spring手动开启停止监听kafka

为了kafka消费者端出现了消费失败时，在闭市时自动进行数据库模式加载，索性，在出现了kafka 消费者端数据失败时，直接将kafka 监听关闭。此时不再接受topic 数据。

- 这样可以避免数据接受错误导致闭市失败。
- 关闭监听后不再进行消费也节省交易过程中清算内存开销。
- 若是需要重新消费或者备点进行消费时，可以从消费失败处重新消费或者换消费者组进行消费。
```
for (int i : i< 200;i ++) {
    try {
    
    } catch (Exception e) {
    
        LOGGER.error("kafka consumer receive data error:" e.geMessage());
        //将错误信息标记到db中。
        bizInsertDatabaseMode(tradedate);
        //kafka 监听关闭KafkaListenerEndpointRegistry 
        for (String id : kafkaListenerEndpointRegisttry.getListenerContainerIds()) {
            registry.getListenerContainer(listenerId).stop();
            logger.info(listenerId + "停止监听成功。");
        }
        //如果这200条数据中第一条就出错了。剩余的199条数据都不进行接受了。
        break;
    }
}

        
```
目前已经过测试为，达到了测试效果。



### 三、kafka 向清算kafka 中写数据

遇到序列化出现问题
```
    the root object can neither be an abstract class nor interface: “[B” with root cause
```

在一个btye[] 类型数据进行序列化时class[B 类型 
原因为： 使用的序列化器并不是 btye[] 导致了以上错误。
将序列化器改正为ByteArrayDeserializer 即可。

目前可以达到，任意切换状态，发送闭市信号，向kafka 中发送 委托，成交等数据。


### 四、zookeeper 日志分片。

```
zookeeper-zwf-server-clear-node-2.out
```
清算项目跑了一段时间后，发现该日志未作分割也没有删除已经达到了47gb大小了严重增加磁盘负担。

![20210129155339](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210129155339.png)

所以我们要通过以下几种方式进行日志清理。
在清理前，我们先简单的了解下zookeeper都有哪些日志：
##### 1.事务日志和快照日志概述
配置文件：/home/zwf/clear/zookeeper-3.5.8/conf/zoo.cfg

事务日志目录：dataDir=/home/zwf/clear/allmetadata/zookeeper

快照日志目录：dataLogDir=/home/zwf/clear/zookeeper-3.5.8/logs

**事务日志:** 指zookeeper系统在正常运行过程中，针对所有的更新操作，在返回客户端“更新成功”的响应前，zookeeper会保证已经将本次更新操作的事务日志已经写到磁盘上，只有这样，整个更新操作才会生效，在dataDir=/home/zwf/clear/allmetadata/zookeeper目录下生成一个version-2目录，该目录下面是一堆格式如log.****事务日志，文件大小为64MB，****表示写入该日志的第一个事务的ID，十六进制表示比如log.3a00000001。
![20210129170958](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210129170958.png)

**快照日志:** zookeeper的数据在内存中是以树形结构进行存储的，而快照就是每隔一段时间就会把整个DataTree的数据序列化后存储在磁盘中，这就是zookeeper的快照文件。在/home/zwf/clear/zookeeper-3.5.8/logs目录下有一个version-2目录，下面是一对格式snapshot.**的快照文件，比如：snapshot.740000014a，其中**表示zookeeper触发快照的那个瞬间，提交的最后一个事务的ID。
![20210129181816](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210129181816.png)

**事务日志可视化:**事务日志为二进制文件，不能通过vim等工具直接访问。其实可以通过zookeeper自带的jar包读取事务日志文件。首先将libs中的slf4j-api-1.7.25.jar文件和zookeeper根目录下的zookeeper-3.5.8.jar文件复制到临时文件夹loginfo中，然后执行如下命令：
```
java -classpath .:slf4j-api-1.7.25.jar:zookeeper-3.5.8.jar:zookeeper-jute-3.5.8.jar org.apache.zookeeper.server.LogFormatter /home/zwf/clear/allmetadata/zookeeper/version-2/log.6300000001
```
结果如下：
![20210129192741](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210129192741.png)

### 3.四种日志清理
#### 1、手动清理
执行ZK_HOME/bin/zkClean.sh

```
./zkCleanup.sh 参数1 -n 参数2。其中：
	参数1，zk data目录，即zoo.cfg文件中dataDir值
	参数2，保存最近的多少个快照
Usage:
PurgeTxnLog dataLogDir [snapDir] -n count
	dataLogDir -- path to the txn log directory
	snapDir -- path to the snapshot directory
	count -- the number of old snaps/logs you want to keep,
	value should be greater than or equal to 3

```

```
ZOODATADIR="$(grep "^[[:space:]]*dataDir=" "$ZOOCFG" | sed -e 's/.*=//')"
ZOODATALOGDIR="$(grep "^[[:space:]]*dataLogDir=" "$ZOOCFG" | sed -e 's/.*=//')"

ZOO_LOG_FILE=zookeeper-$USER-cleanup-$HOSTNAME.log

if [ "x$ZOODATALOGDIR" = "x" ]
then
"$JAVA" "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" "-Dzookeeper.log.file=${ZOO_LOG_FILE}" \
     -cp "$CLASSPATH" $JVMFLAGS \
     org.apache.zookeeper.server.PurgeTxnLog "$ZOODATADIR" $*
else
"$JAVA" "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" "-Dzookeeper.log.file=${ZOO_LOG_FILE}" \
     -cp "$CLASSPATH" $JVMFLAGS \
     org.apache.zookeeper.server.PurgeTxnLog "$ZOODATALOGDIR" "$ZOODATADIR" $*
fi

```
清理命令：
![20210129170229](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210129170229.png)

清理后就剩下3个快照，命令要求至少保留三个快照。

2、通过配置清理

```
vim zoo.cfg
autopurge.snapRetainCount=3
autopurge.purgeInterval=48
```
修改 conf/log4j.properites
```
#声明属性
zookeeper.root.logger=INFO, DAYROLLINGAppender
zookeeper.log.dir=../logs
zookeeper.log.file=zookeeper-zwf-server-clear-node-2.out
 
# ZooKeeper 日志配置  默认INFO级别 输出器CONSOLE
log4j.rootLogger=${zookeeper.root.logger}
 
#DAYROLLING appender
log4j.appender.DAYROLLINGAppender=org.apache.log4j.DailyRollingFileAppender
log4j.appender.DAYROLLINGAppender.DatePattern='.'yyyy-MM-dd-HH
log4j.appender.DAYROLLINGAppender.File=${zookeeper.log.dir}/${zookeeper.log.file}
log4j.appender.DAYROLLINGAppender.Threshold=INFO
log4j.appender.DAYROLLINGAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.DAYROLLINGAppender.layout.ConversionPattern=%d{ISO8601} [myid:%X{myid}] - %-5p [%t:%C{1}@%L] - %m%n
```
修改bin/zkEnv.sh
 
```
if [ "x${ZOO_LOG_DIR}" = "x" ]
then
    ZOO_LOG_DIR="$ZOOBINDIR/../logs"
fi
 
if [ "x${ZOO_LOG4J_PROP}" = "x" ]
then
    ZOO_LOG4J_PROP="INFO,DAYROLLINGAppender"
fi
```
分别是日志目录和日志属性

3、设置定时器清理

使用定时删除日志脚本 推荐使用这一种 结合`sudo crontab -e`，每天定时清理: 
```
0 0 2 * * ? /home/zwf/clear/zookeeper-3.5.8/bin/cleanUplog.sh 
```
```
#!/bin/bash

#snapshot file dir
dataLogDir=/home/zwf/clear/zookeeper-3.5.8/logs/version-2
#transction file dir
dataDir=/home/zwf/clear/allmetadata/zookeeper/version-2
#zk log dir
logDir=/home/zwf/clear/zookeeper-3.5.8/logs/
#保留最新的3个文件 至少保存3个文件
count=3
count=$[$count+1]
##按照时间正序排列|展示从头开始第count行开始|传入执行参数
#事务日志
LOGNUM=`ls -l /home/zwf/clear/allmetadata/zookeeper/version-2/log.* |wc -l`
#将所有满足条件的日志文案进行rm -f 删除
if [ $LOGNUM -gt 0 ]; then
    ls -t $dataDir/log.* | tail -n +$count | xargs rm -f
fi

#快照日志
SNAPSHOTNUM=`ls -l /home/zwf/clear/zookeeper-3.5.8/logs/version-2/snapshot.* |wc -l`
if [ $SNAPSHOTNUM -gt 0 ]; then
    ls -t $dataLogDir/snapshot.* | tail -n +$count | xargs rm -f
fi

#zookeeper-zwf-server-clear-node-2.log
ZKLOGNUM=`ls -l /home/zwf/clear/zookeeper-3.5.8/logs/zookeeper-zwf-server-clear-node-2.log.* |wc -l`
if [ $ZKLOGNUM -gt 0 ]; then
    ls -t $logDir/zookeeper-zwf-server-clear-node-2.log.* | tail -n +$count |xargs rm -f
fi

#zookeeper-zwf-server-clear-node-2.out
if [ -e "$logDir/zookeeper.out" ]; then
    rm -f /home/zwf/clear/zookeeper-3.5.8/logs/zookeeper-zwf-server-clear-node-2.out
fi
```
