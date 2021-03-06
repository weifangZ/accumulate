

**2.4、APIs（学习中）**

**2.4.1、producer Api**

producer 发送消息采用的**异步发送**的方式，批量发送数据是与Ack设置的-1并不冲突，他们是异步的。Ack保证的时生产者数据不丢失，并不是保证同步的问题。

**两个线程**：main/sender线程通过中间的共享变量RecordAccumulator来传输数据的。

**调用顺序**：Interceptors-->serializer-->Partitioner。然后发送给RecordAccumulator共享变量后，sender线程进行发送RecordAccumulator中的批量数据。

掌握了kafkaProducer的发送数据的要领后进行实际的测试：

**普通的producer**
``` java
package com.zwf.kafa;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;


public class KafkaLn {

    public static void main(String[] args) {
        //1、创建卡夫卡生产者的配置信息
        Properties properties = new Properties();
        //2、指定的链接的kafka集群
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.131.131:9092,192.168.131.128:9092,192.168.131.129:9092");
        //3、应答级别
        properties.put(ProducerConfig.ACKS_CONFIG, "all");
        //4、重连次数
        properties.put(ProducerConfig.RETRIES_CONFIG, "5");
        //5、批次大小
        properties.put(ProducerConfig.BATCH_SIZE_CONFIG, "16384");
        //6、等待时间 默认1ms
        properties.put(ProducerConfig.LINGER_MS_CONFIG, "1");
        //7、RecordAccumulator 缓冲区大小
        properties.put(ProducerConfig.BUFFER_MEMORY_CONFIG, "33554432");
        //8、设置序列化
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        //9、创建生产者对象

        final KafkaProducer<String, String> producer = new KafkaProducer<String, String>(properties);
        //10、发送数据
        for (int i = 0; i < 10; i++) {
            producer.send(new ProducerRecord<String, String>("newzwf", 0, i+"" ,"zwftestproducer"+i));
        }
        //用完一定要关闭，否则会产生发送数据失败等等一系列问题
        producer.close();

    }
}

```
在上述代码的第十步看到了send原生的提供的接口：

![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/7D5534E95C6949D2B1489C06086AC82C/24283)

也回顾上上周的理论学习知识。

**带回调函数的的producer**
``` java
package com.zwf.kafa;

import org.apache.kafka.clients.producer.*;

import java.util.ArrayList;
import java.util.List;
import java.util.Properties;
import java.util.concurrent.Future;

public class KafkaCallBackProducer {

    public static void main(String[] args) {
        //1、创建卡夫卡生产者的配置信息
        Properties properties = new Properties();
        //2、指定的链接的kafka集群
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.131.131:9092,192.168.131.128:9092,192.168.131.129:9092");
        //3、应答级别
        properties.put(ProducerConfig.ACKS_CONFIG, "all");
        //4、重连次数
        properties.put(ProducerConfig.RETRIES_CONFIG, "5");
        //5、批次大小
        properties.put(ProducerConfig.BATCH_SIZE_CONFIG, "16384");
        //6、等待时间 默认1ms
        properties.put(ProducerConfig.LINGER_MS_CONFIG, "1");
        //7、RecordAccumulator 缓冲区大小
        properties.put(ProducerConfig.BUFFER_MEMORY_CONFIG, "33554432");
        //8、设置序列化
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        //自定义分区器，可以重写默认的partitions
        properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, MyPartitions.class);
        //9、创建生产者对象

        List<String> list = new ArrayList();
        list.add("1");
        list.add("2");
        list.add("3");
        list.add("4");

        final KafkaProducer<String, String> producer = new KafkaProducer<String, String>(properties);
        for (int i = 0; i < 10; i++) {
            Future<RecordMetadata> send = producer.send(new ProducerRecord<String, String>("mcTrade", list.get(i%4), "12345"+i), new Callback() {
                public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                    if (e == null) {
                        System.out.println(recordMetadata.partition() + "--");
                        System.out.println(recordMetadata.offset() + "--");
                    }
                }
            });
        }
        //10、关闭资源
        producer.close();
    }
}

```
得到结果如下：

![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/F26C3A080D53482AB16A7F9D30558BB4/24333)

因为知道producer调用的顺序需要经过一个分区器：我就学着进行了一项测试：加了个自定义分区器，由原来默认分区器，在不指定分区时，通过key的哈希值进行模上有效分区数得到应发分区。写死了只发给分区0。

**自定义分区器**
``` java
package com.zwf.kafa;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;
import java.util.Map;

public class MyPartitions implements Partitioner {

    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        return 0;
    }

    /**
     * 关闭的
     */
    public void close() {

    }

    /**
     * 读配置信息的
     * @param map
     */
    public void configure(Map<String, ?> map) {

    }
}

```
得到结果如下：

![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/61F19A5718F949499555B5AADFAFD431/24320)

**同步发送API**
```
//使用get会将main线程阻塞，等到sender线程发完，在进行发送，此时就像同步一样。需要配置成1个分区，且同步才会有序
producer.send(new ProducerRecord<String, String>("newzwf", 0, i + "", "zwftestproducer" + i)).get();
```




**2.4.2、consumer Api**

此时我也忍不住进行尝试Consumer的简单的写法了。
``` java
package com.zwf.kafa;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Collections;
import java.util.Properties;

public class MyConsumer {
    public static void main(String[] args) {
        //1、创建卡夫卡生产者的配置信息
        Properties properties = new Properties();
        //2、指定的链接的kafka集群
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.131.131:9092,192.168.131.128:9092,192.168.131.129:9092");
        //3、组
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "all-zwf");
        //4、设置反序列化
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        //5、创建消费者对象
        KafkaConsumer kafkaConsumer = new KafkaConsumer(properties);
        //6、订阅topic
        kafkaConsumer.subscribe(Collections.singleton("mcTrade"));
        //支持正则表达式
        // kafkaConsumer.subscribe("test.*"));
        //轮询topic数据
        try {
            while(true) {
                ConsumerRecords<String, String> records = kafkaConsumer.poll(100);
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println("topic: " + record.topic()+"partition: "+record.partition() + "offset: "+ record.offset()+"key: " + record.key() + "value: " + record.value());
                    int updatedCount = 1;
                    if (record.key().equals("a")){
                        System.out.println(record.value()+"a");
                    }

                    kafkaConsumer.commitAsync();
                }
            }
        } finally {
            kafkaConsumer.close();
        }
    }
}
/*
* 无论是自动提交还是手动提交都会存在问题
* 1、丢数据
* 2、重复数据 第一次我读的是100条数，假设处理时间5s中，10s才会提交offset，当第6s是程序挂掉，此时在启动程序，还会重复数据的。
* 无论自动提交和手动提交都会有以上问题
* 所以提供了一个自定义提交offset方式。
* 应用场景，事务提交一致性（处理与提交绑定到一起）
* 消费者的re-balance 分区分配原则重新触发
* 如果实现自定义提交offset 借助于consumerRebalanceListener
* 消费者要求完全不丢失数据：
* subscribe（这里的函数可以自己重写，何时提交，怎么提交数据自己维护的offset）
*
* */
```

得到的结果如下：

![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/36051DBA64E3422F8763A350FBC98FE4/24350)

**自定义拦截器 interceptor**
既然可以自定义分区器，拦截器肯定可以自定义。
比如场上网kafkatopic里面写
核心方法：

1、configs()\
2、onSend(ProducerRecord)\
3、onAcknowledgement(RecordMetadata, Exception)\
4、close()

**一个计数的拦截器**
``` java
package com.zwf.kafa.interceptor;

import org.apache.kafka.clients.producer.ProducerInterceptor;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Map;

public class CountInterceptor implements ProducerInterceptor<String, String> {

    int success;
    int error;
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> producerRecord) {
        return producerRecord;
    }

    public void onAcknowledgement(RecordMetadata recordMetadata, Exception e) {

        if(recordMetadata != null) {
            success ++;
        } else {
            error++;
        }
    }

    public void close() {
        System.out.println("success: " + success);
        System.out.println("error: " + error);
    }

    public void configure(Map<String, ?> map) {

    }
}

```
producer console print :

![](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/C49D056972B34F7082D2BCCF6B786A31/24531)


**一个加时间戳的拦截器**
``` java
package com.zwf.kafa.interceptor;

import org.apache.kafka.clients.producer.ProducerInterceptor;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Map;

public class TimerInterceptor implements ProducerInterceptor<String, String> {

    /**
     * frist config
     * @param map map
     */
    public void configure(Map<String, ?> map) {

    }
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> producerRecord) {
        long timerStmp = System.currentTimeMillis();

        ProducerRecord<String, String> producerRecordResult = new ProducerRecord<String, String>(producerRecord.topic(),producerRecord.partition(), producerRecord.key(),timerStmp + ", " + producerRecord.value());
        return producerRecordResult;
    }

    public void onAcknowledgement(RecordMetadata recordMetadata, Exception e) {

    }

    public void close() {

    }

}

```
consumer console print :

![](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/D2DEB65AC02846E1811771851B23D113/24529)

**Kafka Producer All Methods**

![](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/C34D97A030DA4C0D94FAB8585E6E9211/24396)

KafkaProducer有4个构造方法
``` java
public KafkaProducer(Map<String,Object> configs)
public KafkaProducer(Map<String,Object> configs,Serializer<K> keySerializer,Serializer<V> valueSerializer)
public KafkaProducer(Properties properties)
public KafkaProducer(Properties properties,Serializer<K> keySerializer,Serializer<V> valueSerializer)
```
          
kafkaProducer目前有12个自有的方法，1个继承父类Producer的方法close()以及继承Object的方法。
Modifier and Type | Method and Description
---|---
void | close()<br> 关闭 producer.
void | close(Duration timeOut)<br> 延时timeOut时间后关闭producer，为了生产者将数据全部发完
Future<RecordMetadata> | send(ProducerRecord<K, V> record)<br> 发送数据
Future<RecordMetadata> | send(ProducerRecord<K, V> record, Callback callback)<br> 发送数据，带回掉函数
void | abortTransaction()<br> 中止正在进行的事务
void | beginTransaction()<br> 每个事务开始前进行调用
void | commitTransaction()<br> 提交事务
void | flush()<br> 刷新，立即发送
void | initTransaction()<br> 初始化事务其他方法调用前进行调用这个方法
Map<MetricName, ? extends Metric> | metrics()<br> 返回的是生产者内部所有的指标信息。特意debug了下
List<PartitionInfo> | partitionsFor(String topic)<br>通过给定的topic获取对应的partition信息
void | sendOffsetsToTransaction(sendOffsetsToTransaction(Map<TopicPartition,OffsetAndMetadata> offsets, ConsumerGroupMetadata groupMetadata))<br>将指定的偏移量列表发送给consumer组的协调器，并将这些偏移量标记为当前事务的一部分。
void | sendOffsetsToTransaction(Map<TopicPartition,OffsetAndMetadata> offsets, String consumerGroupId)<br>向事务协调器发送一个带有groupId的提交offset请求，从而可以在内部__consumer-offsets主题中推导出该消费者组的Topic与Partition

#### 参考文献
1、[kafka官网Producer地址](http://kafka.apache.org/26/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)
