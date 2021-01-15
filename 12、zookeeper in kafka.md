

## kafka 与 zookeeper关系
1、关系图   
![1610589157(1)](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image1610589157(1).png)
如上图所示，kafaka集群的 broker，和 Consumer 都需要连接 Zookeeper。
Producer 直接连接 Broker。

Producer 把数据上传到 Broker，Producer可以指定数据有几个分区、几个备份。

### Zookeeper 在 Kafka 中的作用
#### 1、Broker注册
Broker是分布式部署并且相互之间相互独立，但是需要有一个注册系统能够将整个集群中的Broker管理起来，此时就使用到了Zookeeper。在Zookeeper上会有一个专门用来进行Broker服务器列表记录的节点：

/brokers/ids每个Broker在启动时，都会到Zookeeper上进行注册，即到/brokers/ids下创建属于自己的节点，如/brokers/ids/[0...N]。
![20210114150146](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210114150146.png)


Kafka使用了全局唯一的数字来指代每个Broker服务器，不同的Broker必须使用不同的Broker ID进行注册，创建完节点后，每个Broker就会将自己的IP地址和端口信息记录到该节点中去。其中，Broker创建的节点类型是临时节点，一旦Broker宕机，则对应的临时节点也会被自动删除。
kafka 第二个几点宕机
```
[zwf@clear-node-3 bin]$ ./kafka-server-stop.sh 
[zwf@clear-node-3 bin]$ 
```
显示节点数减少
![20210114162504](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210114162504.png)
如果是zookeeper第二节点宕机：
```
./zkServe.sh stop
```
显示zookeeper集群出现了问题
![1610612474(1)](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image1610612474(1).png)

#### 2、Topic注册
在Kafka中，同一个Topic的消息会被分成多个分区并将其分布在多个Broker上，这些分区信息及与Broker的对应关系也都是由Zookeeper在维护，由专门的节点来记录，如：

/borkers/topics
![1610621602(1)](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image1610621602(1).png)

Kafka中每个Topic都会以/brokers/topics/[topic]的形式被记录，如/brokers/topics/mcTrade和/brokers/topics/zwfnew等。Broker服务器启动后，会到对应Topic节点（/brokers/topics）上注册自己的Broker ID并写入针对该Topic的分区总数，如`/brokers/topics/mcTrade/partitions->0`

![1610622711(1)](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image1610622711(1).png)
，这个节点表示Broker有一个分区在用。
