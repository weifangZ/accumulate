### 学习计划
<!--![20201023112106](https://raw.githubusercontent.com/weifangZ/image/master/image/20201023112106.png)-->

### 一、准备环境
运维学习环境以清算生产环境部署为模型进行模拟，搭建三点集群环境。
```
kafka4 ： 192.168.131.131
kafka3 ： 192.168.131.129
kafka2 ： 192.168.131.128
```
### 二、kafka运维学习重点
**2.1、kafka常用运维操作指令。（学习中）**

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







工作流程\
当生产者向不存在的topic里面发送数据，会默认生产一个1个分区1个副本的topic。
每个分区都维护一个offet
forllower  要同步leader数据

**2.2、配置（学习中）**

**数据可靠性保证方式：**
为了保证producor发送的数据尽量可靠的发送到指定的topic中，topic的每个partition收到producer发送的数据后都向producer发送ack，如果producer收到ack后进行下一轮发送。如果收不到ack则进行重新发送。

这样就会出现何时发送ack呢？
- 1、leader和follower都收到了在发ack \
优点：副本少 缺点：延迟高
- 2、半数以上的follower收到后发送ack \
优点：延迟低 缺点：副本多

kafka 选用第二种方案，做了个优化提出了**ISR**


**1、ISR:in-sync replica set （同步副本）**\
应对数据可靠性保证。解决副本故障不能发送ack的问题\
kafka查看ISR中的follower是否同步完成，完成后直接发ack不在等待不在ISR中的follower是否同步完成。当然如果出现ISR中的follower出现了长时间不同步数据，就将它提出ISR重新选取follower。
选取follower \
进入ISR原则：
- leader节点挂掉了后选取新的leader尽量不丢数据。
- 尽量选offset高的，丢数据丢的少。
- replica.lag.time.max.ms 如果follower不能按时发送一个请求，或者不能消费最后一下offset时会移除isr队列，默认10秒。
- replica.lag.time.max.message（被移除了）
（两个条件的时候因为，会出现，满足1进去，后不满足2时剔除isr，这样会频繁的操作内存。，还要操作Zookeeper。0.9版本后移除）

**2、ack应答机制**
提供三种级别（acks配置）：
- 0，leader写完无论follower写没写完，producer只发一遍不等leader与follower时候写完。**会丢数据**
- 1，只等待leader，不等到follower写完就同步数据，此时leader挂了会丢数据。 **会丢数据**
- -1（all），此时等待leader与isr里面的follower全部完成后来返回ack。极限情况也会有丢数据时，isr只有leader了，follower特别慢时候isr就只有leader，退化到了1的情况。**重复数据，保证producer数据不丢失**

等于-1时会重复数据，leader与follower已经同步完成，此时leader挂了，还没发送ack，选一个follower当leader，此时producer会重复发送数据。

**3、数据存储一致性问题**

**保证消费者消费数据一致性**
- **HW** 高水位线（多个isr中最小的offset）
- **LEO** 最后一个offset （每个副本中的最新的那个offset）\
kafka对于消费者暴露的只有hw，此时能够保证消费者消费的数据一致性问题。木桶短板效应。

**存储数据一致性问题**

重新选择leader时，给所有follower发消息要截取到HW位置。高于HW的数据全部截取掉，不进行存储。

**HW**只保证数据一致性而不保证数据丢不丢与数据重复不重复问题。

 **5、Broker Configs**
 
    - broker.id
    - log.dirs
    - zookeeper.connect
![](https://github.com/weifangZ/accumulate/blob/main/images/zookeeper-connect.png)
**2.3、kafka性能调优（待学习）**

**2.4、APIs（待学习）**


**2.5、存储（学习中）**

index文件与
log文件
快速查找定位。
每个数据块大小相同，通过offset乘以块大小直接定位数据位置。






### 三、kafka常见问题汇总

**问题1**：kafka topic数据与内存数据不一致的情况。\
**首先：** 
查看消费情况
```
./kafka-consumer-groups.sh --describe --bootstrap-server 170.0.39.164:9092 --group dce-mt-realTime-wanghong12345
```
执行下面命令即可将日志数据文件内容dump出来 
```
./kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/dce/clear/allmetadata/mcTrade-0/00000000000022372103.log --print-data-log > 00000000000022372103_txt.log
```
通过对比内存与kafka中的导出数据进行对比可以解决。
