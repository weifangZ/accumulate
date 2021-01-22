### 一、 自动删除数据

### kafka 数据周期性删除

#### 准备测试
```
# 一小时删除
log.retention.hours=1
# 500k后删除
log.retention.bytes=524288
```
使用自己写的kafka 工具删除topic
```
[zwf@clear-node-4 bin]$ ./kafka-util.sh -d zwfnew
============Args: -d zwfnew =================

```
#### 插曲
遇到了插曲，互信错了，原因是其中一个点的子网掩码与代理设置错误。
zookeeper集群搭建问题：
``` 
Cannot open channel to * at election address
```
发现 一个点不能ping通另外两个点了，所以导致了以上错误。

设置了代理：

![20210121183017](https://github.com/weifangZ/image/blob/master/image20210121183017.png)

#### 开始正式测试

##### 场景一 

写入1.8m的数进行分块4个，然后通过分块大小进行删除了分块，等待一个小时后将没达到512k的块也进行了删除。

向mcTradenew topic 中写入了10w条数据

![20210121183728](https://github.com/weifangZ/image/blob/master/image20210121183728.png)

kafka只会回收上个分片的数据
配置没有生效的原因就是，数据并没有分片，所以没有回收

![20210121191031](https://github.com/weifangZ/image/blob/master/image20210121191031.png)

数据分块了

![20210121191121](https://github.com/weifangZ/image/blob/master/image20210121191121.png)

数据删除了。

![20210121191201](https://github.com/weifangZ/image/blob/master/image20210121191201.png)

等待1小时后查看数据删除效果。

![1611275652](https://github.com/weifangZ/image/blob/master/image1611275652.png)

![20210122083517](https://github.com/weifangZ/image/blob/master/image20210122083517.png)

最后发现最后新生成的一个新的offset log文件，而且内容为空。此时消费数据也不能够消费成功，

![20210122083754](https://github.com/weifangZ/image/blob/master/image20210122083754.png)

##### 场景二、
一个log文件已经建立了超过1小时的时间了。此时我像里面写入10条数据，等待60s然后进行消费查看

![20210122084012](https://github.com/weifangZ/image/blob/master/image20210122084012.png)

一分钟过后数据没被删除，
在等59分钟，后进行写入10条数据，此时我们再等60s
无变化。说明按照时间删除数据的方案是按照这个分块送最新的时间开始算起。并且如果不分片就不会删除该数据。

![20210122111744](https://github.com/weifangZ/image/blob/master/image20210122111744.png)


#### kafka回收原理
kafka只会回收上个分片的数据
配置没有生效的原因就是，数据并没有分片，所以没有回收

kafka什么时候分片？有个参数log.roll.hours
log.roll.hours 设置多久滚动一次，滚动也就是之前的数据就会分片分出去
segment.bytes 设置日志文件到了多大就会自动分片
#### 最后统一解释一下自动删除数据的配置
```
  #  删除数据时间间隔
  "log.retention.hours": 1
  # 数据清理 策略 此时是删除
  "log.cleanup.policy": "delete"
  # 数据大达到xxm会被删除
  "log.retention.bytes": "536870912"
  # 数据删除延时1分钟
  "log.segment.delete.delay.ms": 60000
  # 指定日志每隔多久检查看是否可以被删除 3mins
  "log.cleanup.interval.mins": 3
  # 当前日志分段中消息的最大时间戳与当前系统的时间戳的差值允许的最大范围，小时维度
  "log.roll.hours": 1
  # 日志文件最大值
  "log.segment.bytes": "536870913"
  # 检测频率
  "log.retention.check.interval.ms": 120000
  # 日志保留时间毫秒
  "log.retention.ms": "3600000"
  # 运行保留日志文件最大值
  "log.retention.bytes": "536870912"
```
建议
log.roll.hours
retention.ms
log.retention.hours
设置的时间相同

### 二、手动删除数据

### 清算每日清理topic说明文档

提供两个文件
首先将deltopic.sh与topiclist.txt这两个文件放在kafka 服务器的$CLEAR_HOME文件夹下面

deltopic.sh
```
#deltopic.sh
for line in `cat topiclist.txt`
do
 # 删除集群上的所有元数据，因为三点集群已经为互信不在输入密码
 rm -rf  ssh zwf@clear-node-2:/home/zwf/clear/allmetadata/kafka/${line}*
 rm -rf  ssh zwf@clear-node-3:/home/zwf/clear/allmetadata/kafka/${line}*
 rm -rf  ssh zwf@clear-node-4:/home/zwf/clear/allmetadata/kafka/${line}*
 ## 在kafka/bin下面删除topic
 $KAFKA_HOME/bin/kafka-topics.sh --bootstrap-server 192.168.131.131:9092,192.168.131.129:9092,192.168.131.128:9092 --topic ${line} --delete

 ## 新建topic
 $KAFKA_HOME/bin/kafka-topics.sh --bootstrap-server 192.168.131.131:9092,192.168.131.129:9092,192.168.131.128:9092 --create --replication-factor 3 --partitions 1 --topic ${line}
 echo topic ${line} is ok!
done
```
topiclist.txt
```
mcTrade
test
bizunit
tradesys
result
```
说明：

*1、另外加上多机以及需要每日情况的topic*

*2、多个分区的topic可以手动配置到deltopic.sh中（目前应该已经没有了）*


### 工作说明：
1、先能读取文件，每行作为另一个topic进行循环，
for line in `cat topiclist.txt`
do
 echo $line
done

2、做到以上不重启kafka可以清理topic内容效果需要配置以下内容：


- 如果需要被删除topic此时正在被程序 produce和consume，则这些生产和消费程序需要停止。

- 同时，需要设置
auto.create.topics.enable = false，默认设置为true。如果设置为true，则produce或者fetch 不存在的topic也会自动创建这个topic。这样会给删除topic带来很多意向不到的问题。

- server.properties 设置 delete.topic.enable=true
如果没有设置 delete.topic.enable=true，则调用kafka 的delete命令无法真正将topic删除，而是显示（marked for deletion）

### 操作说明：

1、删除log文件
首先知道kafka下面的server.config文件里面配置的

# 删除log文件首先知道kafka下面的server.config文件里面配置的
```
 rm -rf  ssh zwf@clear-node-2:/home/zwf/clear/allmetadata/kafka/${line}*
 rm -rf  ssh zwf@clear-node-3:/home/zwf/clear/allmetadata/kafka/${line}*
 rm -rf  ssh zwf@clear-node-4:/home/zwf/clear/allmetadata/kafka/${line}*

```
     
2、删除topic
在kafka/bin下面删除topic
```
./home/zwf/clear/kafka_2.12-2.5.0/bin/kafka-topics.sh --bootstrap-server 192.168.131.131:9092,192.168.131.129:9092,192.168.131.128:9092 --topic test --delete
./home/zwf/clear/kafka_2.12-2.5.0/kafka-bin/kafka-topics.sh --bootstrap-server 192.168.131.131:9092,192.168.131.129:9092,192.168.131.128:9092 --topic mcTrade --delete
```
### 删除zookeeper 元数据
一般而言，经过上面2步以及配置就可以正常删除掉topic和topic的数据。但是，如果经过上面2步，还是无法正常删除topic，则需要对kafka在zookeeer的存储信息进行删除。具体操作如下：

3、zookeeper相关的路径
此时并没有完全删除只是把相应topic的状态改为marked for deletion
```
/home/zwf/clear/zookeeper-3.5.8/bin/zkCli.sh
rmr /brokers/topics/mcTrade
rmr /brokers/topics/test
```

执行结果如下

![deletetpic](https://github.com/weifangZ/image/blob/master/imagedeletetpic.png)


