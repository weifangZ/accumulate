
#### spring kafka

因为我们组项目的kafka架构是基于spring kafka的。所以为了能够运维好该项目理应在学习原生的kafka基础上去学习spring kafka。

由于有了apache kafka的学习基础在学习spring kafka时我直接上手了官网的例子，从例子出发去探索对应的api等相关知识。

**pom.xml**
``` pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>lnKafka</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>2.5.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
            <version>2.3.8.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.kafka</groupId>
                    <artifactId>kafka-clients</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>1.7.25</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>
</project>

```
**spring.kafka.autoCommit.test**
```
package com.zwf.spring.kafka;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.IntegerDeserializer;
import org.apache.kafka.common.serialization.IntegerSerializer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;
import org.springframework.kafka.listener.ContainerProperties;
import org.springframework.kafka.listener.KafkaMessageListenerContainer;
import org.springframework.kafka.listener.MessageListener;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.junit.Assert.assertTrue;

public class testAutoCommit {
    private final Logger logger = LoggerFactory.getLogger(testAutoCommit.class);
    @Test
    public void testAutoCommit() throws Exception {
        logger.info("Start auto");
        ContainerProperties containerProps = new ContainerProperties("mcTrade", "zwfnew");
        final CountDownLatch latch = new CountDownLatch(4);
        containerProps.setMessageListener(new MessageListener<Integer, String>() {
            @Override
            public void onMessage(ConsumerRecord<Integer, String> message) {
                logger.info("received: " + message);
                latch.countDown();
            }

        });
        KafkaMessageListenerContainer<Integer, String> container = createContainer(containerProps);
        container.setBeanName("testAuto");
        container.start();
        Thread.sleep(1000); // wait a bit for the container to start
        KafkaTemplate<Integer, String> template = createTemplate();
        template.setDefaultTopic("mcTrade");
        template.sendDefault(0, "foo");
        template.sendDefault(2, "bar");
        template.sendDefault(0, "baz");
        template.sendDefault(2, "qux");
        template.flush();
        assertTrue(latch.await(60, TimeUnit.SECONDS));
        container.stop();
        logger.info("Stop auto");

    }

    private KafkaMessageListenerContainer<Integer, String> createContainer(
            ContainerProperties containerProps) {
        Map<String, Object> props = consumerProps();
        DefaultKafkaConsumerFactory<Integer, String> cf =
                new DefaultKafkaConsumerFactory<Integer, String>(props);
        KafkaMessageListenerContainer<Integer, String> container =
                new KafkaMessageListenerContainer<>(cf, containerProps);
        return container;
    }

    private KafkaTemplate<Integer, String> createTemplate() {
        Map<String, Object> senderProps = senderProps();
        ProducerFactory<Integer, String> pf =
                new DefaultKafkaProducerFactory<Integer, String>(senderProps);
        KafkaTemplate<Integer, String> template = new KafkaTemplate<>(pf);
        return template;
    }

    private Map<String, Object> consumerProps() {
        Map<String, Object> props = new HashMap<>();
        String group = "zwf-new-1";
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.131.131:9092,192.168.131.129:9092,192.168.131.128:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, group);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "100");
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "15000");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, IntegerDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return props;
    }

    private Map<String, Object> senderProps() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.131.131:9092,192.168.131.129:9092,192.168.131.128:9092");
        props.put(ProducerConfig.RETRIES_CONFIG, 0);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, IntegerSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return props;
    }
}

```
执行结果：

![](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/00F70B3EE5BB49638BA029FDCFB47C76/24555)

通过分析源码发现spring kafka 的核心文件：

application.yml
```
stream:
  kafka:
    producer:
      # kafka brooker 服务器端口
      bootstrap-servers: 170.0.50.10:9092,170.0.50.11:9092,170.0.50.12:9092
      # 键的序列化方式
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      # 值的序列化方式
      value-serializer: org.apache.kafka.common.serialization.ByteArraySerializer
      #当有多个消息需要被发送到同一个分区时，生产者会把它们放在同一个批次里。该参数指定了一个批次可以使用的内存大小，按照字节数计算。
      batch-size: 16384
      # 设置生产者内存缓冲区的大小。
      buffer-memory: 33554432
      # acks=0 ： 生产者在成功写入消息之前不会等待任何来自服务器的响应。
      # acks=1 ： 只要集群的首领节点收到消息，生产者就会收到一个来自服务器成功响应。
      # acks=all ：只有当所有参与复制的节点全部收到消息时，生产者才会收到一个来自服务器的成功响应。
      acks: all
      #为特点的topic指定一个最终压缩类型。此配置接受的标准压缩编码方式有（'gzip','snappy', 'lz4'）。此外还有'uncompressed'相当于不压缩；'producer'意味着压缩类型由'producer'决定。
      # lz4 经过实验对比非常快压缩和解压都很快，只是压缩完的数据包较大
      # snappy 压缩/解压速度也比较快，合理的压缩率,不支持split，支持hadoop native库，需要自己安装。可以用于map中间结果的压缩。
      compression-type: lz4
      properties:
        #batch-size 与linger.ms配合起来就是在100ms内攒的批量数据打包发送到topic上。满足batch.size和ling.ms之一，就会激活sender线程来发送消息。
        #情况1、当数据已经到了batch-size（16k）量，但是时间没到100ms，进行发送数据。
        #情况2、当数据不满batch-size时间到了100ms也会发送数据。
        #情况3、当发生情况一的时候再来数据时超过了batch-size此时会报错（warning）不让其发给topic。
        linger.ms: 100
        #enable.idempotence设置成true后，Producer自动升级成幂等性Producer。Kafka会自动去重。Broker会多保存一些字段。当Producer发送了相同字段值的消息后，Broker能够自动知晓这些消息已经重复了。
        enable.idempotence: true
    consumer:
      bootstrap-servers: 170.0.50.10:9092,170.0.50.11:9092,170.0.50.12:9092
      # 是否自动提交偏移量，默认值是true,为了避免出现重复数据和数据丢失，可以把它设置为false,然后手动提交偏移量
      enable-auto-commit: false
       # 该属性指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下该作何处理：
      # latest（默认值）在偏移量无效的情况下，消费者将从最新的记录开始读取数据（在消费者启动之后生成的记录）
      # earliest ：在偏移量无效的情况下，消费者将从起始位置读取分区的记录
      auto-offset-reset: earliest
      # 组id 组号相同的consumer会分别消费不同的partition，当发生一个消费者的消费能力不足时，kafka会rebalance到组内其他consumer上
      group-id: dce-mt-realtime-wanghong12345
      # 键的反序列化方式
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      # 值的反序列化方式
      value-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
      # 一次调用poll()操作时返回的最大记录数，默认值为500
      max-poll-records: 10000
      # 启动程序自动开启kafka监听 true|false
      auto-startup: false
      #它表示最大的poll数据间隔，如果超过这个间隔没有发起pool请求，但heartbeat仍旧在发，就认为该consumer处于 livelock状态。就会将该consumer退出consumer group。所以为了不使Consumer 自己被退出，Consumer 应该不停的发起poll(timeout)操作。而这个动作 KafkaConsumer Client是不会帮我们做的，这就需要自己在程序中不停的调用poll方法了。
      max-poll-interval-ms: 500
```
![](https://www.pianshen.com/images/825/4c4e8906981b5e1f3bb5f95792350a81.png)
producer:kafkaTemplate
这个类的接口前面也提过,这里详细介绍一下：

1、接口入参参数解释：

- topic：topic name
- partition：分区 id
- timestamp：时间戳
- key：消息的键
- data：消息的数据
- ProducerRecord：消息对应的封装类可以包含（topic、partition、timestamp、key、data等信息）
- Message<?>：Spring自带的Message封装类，包含消息及消息头

2、kafkaTemplate内部变量：
- producerFactory\
producerFactory用来创建KafkaProducer。getProducerFactory(topic)方法通过传入的topic来确定producerFactory，默认是返回this.producerFactory
- autoFlush\
在等待的过程中，你可能想要调用flush()方法，为了简便，template类提供了一个autoFlush，这个属性将会在每次send的时候，立即去flush掉sending thread.不过autoFlush将会明显降低性能(boolean autoFlush）。 


```
//sendDefault 实际也是send()只是先通过setDefaultTopic方法将topic定义好，这样就会向指定的topic发送数据
ListenableFuture<SendResult<K, V>> sendDefault(V data);
//同send(String topic, K key, V data)
ListenableFuture<SendResult<K, V>> sendDefault(K key, V data);
//同send(String topic, Integer partition, K key, V data);
ListenableFuture<SendResult<K, V>> sendDefault(Integer partition, K key, V data);
//同send(String topic, Integer partition, Long timestamp, K key, V data);
ListenableFuture<SendResult<K, V>> sendDefault(Integer partition, Long timestamp, K key, V data);
//该方法也是封装了apache kafka原生的 send方法 ,通过getTheProducer获得KafkaProducer对象，KafkaProducer对象发送数据ProducerRecord
ListenableFuture<SendResult<K, V>> send(String topic, V data);
ListenableFuture<SendResult<K, V>> send(String topic, K key, V data);
ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, K key, V data);
ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, Long timestamp, K key, V data);
ListenableFuture<SendResult<K, V>> send(ProducerRecord<K, V> record);
ListenableFuture<SendResult<K, V>> send(Message<?> message);
```
**comsumer:@KafkaListener**

@KafkaListener注解提供了以下配置功能
![](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/0FDDD42301444AFC9057547E57218571/24676)

对应参数解释如下：

参数名称 | 参数类型 | 解释
---|--- |---
autoStartup | String | true、false 是否自动开启监听
beanRef | String | 此注解中SpEL表达式中使用的伪bean名，用于指向此监听器的当前bean，从而允许访问封装bean中的属性和方法。
clientIdPrefix| String | 用来重写客户端Id的功能
concurrency | String | 重新设置并发设置
containerFactory | String | 设置监听容器工厂类
containerGroup| String |设置了这个属性，当前的监听器会被加进设置的这个容器组里面，后面你可以通过遍历这个集合来启动或终止一组监听器集合。
errorHandler | String |异常处理器，如果监听器处理方法抛出异常，你可以设置一个实现了KafkaListenerErrorHandler的异常处理类来处理抛出的异常。
groupId | String |重新设置消费者所属组
id| String |代表当前节点的唯一标识，不配置的话会自动分配一个id，主动配置的话，groupId会被设置成id的值（前提是idIsGroup这个属性值没有被设置成false）。
idIsGroup | boolean |id是否能用作groupId 默认 true
properties| String[]| 消费者属性，将替换在消费者工厂中定义的具有相同名称的任何属性(如果消费者工厂支持属性覆盖)。
splitIterables| boolean |当设置为false且返回类型为Iterable时，返回结果作为单个返回记录的值，而不是每个元素的单个记录。
topicPartitions|TopicPartition[]|可以设置更加详细的监听信息，包括topic、partitions和partitionOffsets。
topicPattern| String |Topic主题，支持属性占位符，或者是正则表达式。批量主题订阅
topics | String[] |可以订阅多个主题


参考文献：

1、[kafka官网](http://kafka.apache.org/documentation/)\
2、[深入浅出Linux-零拷贝技术sendfile](https://www.jianshu.com/p/028cf0008ca5)\
3、[KafkaTemplate发送消息及结果回调](http://blog.seasedge.cn/archives/15.html)\
4、[kafkaProducerAPIs](https://kafka.apachecn.org/10/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)\
5、[Spring for Apache Kafka](https://docs.spring.io/spring-kafka/docs/current/reference/html/)\
6、[Spring  KafkaListener](https://docs.spring.io/spring-kafka/api/org/springframework/kafka/annotation/KafkaListener.html)
