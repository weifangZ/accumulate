
## kafka leader选举机制原理

#### kafka 选举leader的发展

在Kafka早期版本，对于分区和副本的状态的管理依赖于zookeeper的Watcher和队列：每一个broker都会在zookeeper注册Watcher，所以zookeeper就会出现大量的Watcher, 如果宕机的broker上的partition很多比较多，会造成多个Watcher触发，造成集群内大规模调整；每一个replica都要去再次zookeeper上注册监视器，当集群规模很大的时候，zookeeper负担很重。这种设计很容易出现脑裂和羊群效应以及zookeeper集群过载。

新的版本中该变了这种设计，使用KafkaController，只有KafkaController，Leader会向zookeeper上注册Watcher，其他broker几乎不用监听zookeeper的状态变化。

Kafka集群中多个broker，有一个会被选举为controller leader，负责管理整个集群中分区和副本的状态，比如partition的leader 副本故障，由controller 负责为该partition重新选举新的leader 副本；当检测到ISR列表发生变化，有controller通知集群中所有broker更新其MetadataCache信息；或者增加某个topic分区的时候也会由controller管理分区的重新分配工作

#### Kafka集群Leader选举原理

Kafka的Leader选举是通过在zookeeper上创建/controller临时节点来实现leader选举，并在该节点中写入当前broker的信息
{“version”:1,”brokerid”:1,”timestamp”:”1512018424988”}
利用Zookeeper的强一致性特性，一个节点只能被一个客户端创建成功，创建成功的broker即为leader，即先到先得原则，leader也就是集群中的controller，负责集群中所有大小事务。
当leader和zookeeper失去连接时，临时节点会删除，而其他broker会监听该节点的变化，当节点删除时，其他broker会收到事件通知，重新发起leader选举。

### topic leader 选举测试
通过以下代码可以查询某topic的leader。
```
package com.zwf.kafa;

import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.Node;
import org.apache.kafka.common.PartitionInfo;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;

public class findLeader {

    public static void main(String[] args) {
        Map<String, Integer> brokers = new HashMap<>();
        String topic = "test";
        int partition = 0;
        Node node = findLeader(brokers, topic, partition);
        System.out.println("topic is :" + topic);
        System.out.println("leader is :" + node.host());
    }
    /**
     * <p>
     * 用于寻找指定Brokers-Topic-Partition对应的Leader.
     * 主要是封装了{@link KafkaConsumer#partitionsFor}方法.
     *
     * @param brokers   指定的Kafka集群,格式为<"host","port">
     * @param topic     指定的Topic
     * @param partition 指定的Partition ID
     * @return Node {@link Node}
     * @date 2021/2/3
     */
    public static Node findLeader(Map<String, Integer> brokers, String topic, int partition) {
        // 1.通过接收到的配置信息创建KafkaConsumer实例
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.131.131:9092,192.168.131.128:9092,192.168.131.129:9092");
        //3、应答级别2
        // 指定Consumer所属的ConsumerGroup
        props.put("group.id", "test");
        // 设置手动提交Offset,由于只是获取部分数据所以不需要自动提交Offset
        props.put("enable.auto.commit", "false");
        // 设置键值对<K,V>反序列化器,使用String对象反序列化器
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        // 2.定义KafkaConsumer实例,采用try{}catch{}finally{}的方式获取资源
        KafkaConsumer<String, String> kafkaConsumer = null;
        try {
            // 3.创建KafkaConsumer实例
            kafkaConsumer = new KafkaConsumer<>(props);
            // 4.通过KafkaConsumer实例获取指定Topic对应的Partition信息
            List<PartitionInfo> partitionInfos = kafkaConsumer.partitionsFor(topic);
            // 5.遍历返回的指定Topic的所有Partition信息
            for (PartitionInfo partitionInfo : partitionInfos) {
                // 若当前Partition是指定的Partition,则保存此Partition的Replication并将其Leader返回
                if (partitionInfo.partition() == partition) {
                    /*// 保存指定Partition的所有副本
                    replications = partitionInfo.replicas();*/
                    // 6.返回指定的TopicPartition对应的Leader
                    return partitionInfo.leader();
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (kafkaConsumer != null) kafkaConsumer.close();
        }
        // 如果没有找到Leader则返回null
        return null;
    }
}

```
结果如下：

![20210203195251](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210203195251.png?ynotemdtimestamp=1612497286584)


如果

![20210203195700](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210203195700.png?ynotemdtimestamp=1612497286584)

如果其中leader 节点的kafka 挂了

![20210204084623](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210204084623.png?ynotemdtimestamp=1612497286584)

此时的kafka与zookeeper日志如下：
```
# 首先抛出
java.io.IOException: Connection to 192.168.131.129:9092 (id: 2 rack: null) failed.
```

然后选举新的leader

![20210204085318](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210204085347.png?ynotemdtimestamp=1612497286584)

![20210204085347](https://cdn.jsdelivr.net/gh/weifangZ/image@master/image20210204085347.png?ynotemdtimestamp=1612497286584)

会发现id = 3 的节点接手了topic test 的leader 的位置
并且调整topic的hw
```
 newzwf-0 starts at leader epoch 167 from offset 163 with high watermark 163. Previous leader epoch was 166. (kafka.cluster.Partition)
```

测试2
