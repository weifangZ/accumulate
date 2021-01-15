

## kafka 与 zookeeper关系
1、关系图   
![1610589157(1)](https://github.com/weifangZ/image/blob/master/image1610589157(1).png)
如上图所示，kafaka集群的 broker，和 Consumer 都需要连接 Zookeeper。
Producer 直接连接 Broker。

Producer 把数据上传到 Broker，Producer可以指定数据有几个分区、几个备份。

### Zookeeper 在 Kafka 中的作用
#### 1、Broker注册
Broker是分布式部署并且相互之间相互独立，但是需要有一个注册系统能够将整个集群中的Broker管理起来，此时就使用到了Zookeeper。在Zookeeper上会有一个专门用来进行Broker服务器列表记录的节点：

/brokers/ids每个Broker在启动时，都会到Zookeeper上进行注册，即到/brokers/ids下创建属于自己的节点，如/brokers/ids/[0...N]。
![20210114150146](https://github.com/weifangZ/image/blob/master/image20210114150146.png)


Kafka使用了全局唯一的数字来指代每个Broker服务器，不同的Broker必须使用不同的Broker ID进行注册，创建完节点后，每个Broker就会将自己的IP地址和端口信息记录到该节点中去。其中，Broker创建的节点类型是临时节点，一旦Broker宕机，则对应的临时节点也会被自动删除。
kafka 第二个几点宕机
```
[zwf@clear-node-3 bin]$ ./kafka-server-stop.sh 
[zwf@clear-node-3 bin]$ 
```
显示节点数减少
![20210114162504](https://github.com/weifangZ/image/blob/master/image20210114162504.png)
如果是zookeeper第二节点宕机：
```
./zkServe.sh stop
```
显示zookeeper集群出现了问题
![1610612474(1)](https://github.com/weifangZ/image/blob/master/image1610612474(1).png)

#### 2、Topic注册
在Kafka中，同一个Topic的消息会被分成多个分区并将其分布在多个Broker上，这些分区信息及与Broker的对应关系也都是由Zookeeper在维护，由专门的节点来记录，如：

/borkers/topics
![1610621602(1)](https://github.com/weifangZ/image/blob/master/image1610621602(1).png)

Kafka中每个Topic都会以/brokers/topics/[topic]的形式被记录，如/brokers/topics/mcTrade和/brokers/topics/zwfnew等。Broker服务器启动后，会到对应Topic节点（/brokers/topics）上注册自己的Broker ID并写入针对该Topic的分区总数，如`/brokers/topics/mcTrade/partitions->0`

![1610622711(1)](https://github.com/weifangZ/image/blob/master/image1610622711(1).png)
，这个节点表示Broker有一个分区在用。

#### 3、生产者负载均衡

由于同一个Topic消息会被分区并将其分布在多个Broker上，因此，生产者需要将消息合理地发送到这些分布式的Broker上，那么如何实现生产者的负载均衡，Kafka支持传统的四层负载均衡（但是在实际使用时候无法真正的达到负载均衡效果），也支持Zookeeper方式实现负载均衡。

(1) 四层负载均衡，根据生产者的IP地址和端口来为其确定一个相关联的Broker。通常，一个生产者只会对应单个Broker，然后该生产者产生的消息都发往该Broker。这种方式逻辑简单，每个生产者不需要同其他系统建立额外的TCP连接，只需要和Broker维护单个TCP连接即可。但是，其无法做到真正的负载均衡，因为实际系统中的每个生产者产生的消息量及每个Broker的消息存储量都是不一样的，如果有些生产者产生的消息远多于其他生产者的话，那么会导致不同的Broker接收到的消息总数差异巨大，同时，生产者也无法实时感知到Broker的新增和删除。

(2) 使用Zookeeper进行负载均衡，由于每个Broker启动时，都会完成Broker注册过程，生产者会通过该节点的变化来动态地感知到Broker服务器列表的变更，这样就可以实现动态的负载均衡机制。

#### 4、消费者负载均衡
与生产者类似，Kafka中的消费者同样需要进行负载均衡来实现多个消费者合理地从对应的Broker服务器上接收消息，每个消费者分组包含若干消费者，每条消息都只会发送给分组中的一个消费者，不同的消费者分组消费自己特定的Topic下面的消息，互不干扰。
![20210115091827](https://github.com/weifangZ/image/blob/master/image20210115091827.png)

#### 5、分区 与 消费者 的关系
如上图、消费组 (Consumer Group)：consumer group 下有多个 Consumer（消费者）。对于每个消费者组 (Consumer Group)，Kafka都会为其分配一个全局唯一的Group ID，Group 内部的所有消费者共享该 ID。订阅的topic下的每个分区只能分配给某个 group 下的一个consumer(当然该分区还可以被分配给其他group)。实际也是在一个consumer 上进行消费，同组其他consumer只有在rebalance时才有机会消费。
![20210115092026](https://github.com/weifangZ/image/blob/master/image20210115092026.png)
同时，Kafka为每个消费者分配一个Consumer ID，通常采用"Hostname:UUID"形式表示。

在Kafka中，规定了每个消息分区 只能被同组的一个消费者进行消费，因此，需要在 Zookeeper 上记录 消息分区 与 Consumer 之间的关系，每个消费者一旦确定了对一个消息分区的消费权力，需要将其Consumer ID 写入到 Zookeeper 对应消息分区的临时节点上，例如：

/consumers/[group_id]/owners/[topic]/[broker_id-partition_id]

其中，[broker_id-partition_id]就是一个 消息分区 的标识，节点内容就是该 消息分区 上 消费者的Consumer ID。

#### 6、消息 消费进度Offset 记录
在消费者对指定消息分区进行消息消费的过程中，需要定时地将分区消息的消费进度Offset记录到Zookeeper上，以便在该消费者进行重启或者其他消费者重新接管该消息分区的消息消费后，能够从之前的进度开始继续进行消息消费。Offset在Zookeeper中由一个专门节点进行记录，其节点路径为:

/consumers/[group_id]/offsets/[topic]/[broker_id-partition_id]

节点内容就是Offset的值。
![d8a4d8da92d8f02cd7cae263c682c6fd_70](https://github.com/weifangZ/image/blob/master/imaged8a4d8da92d8f02cd7cae263c682c6fd_70.png)


在kafka 0.9.5 版本之后 consumer 内容 consumers 内的内容不在由zookeeper管理储存，移交到kafka存储。所以新版zookeeper 不在管理 consumers-group 与offset 内容。
![20210115104813](https://github.com/weifangZ/image/blob/master/image20210115104813.png)

![consumer](https://github.com/weifangZ/image/blob/master/imageconsumer.png)

