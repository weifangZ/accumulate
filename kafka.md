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

**1、ISR:**\
应对数据可靠性保证。
解决副本故障不能发送ack的问题\
in-sync replica set  （同步副本）\
kafka查看ISR中的follower是否同步完成，完成后直接发ack不在等待不在ISR中的follower是否同步完成。当然如果出现ISR中的follower出现了长时间不同步数据，就将它提出ISR重新选取follower。
选取follower \
进入ISR原则：
- leader节点挂掉了后选取新的leader尽量不丢数据。
- 尽量选offset高的，丢数据丢的少。


**2.3、kafka性能调优（待学习）**

**2.4、APIs（待学习）**






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
