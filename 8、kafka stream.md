
## kafka stream
由于本周讨论高频手续费实时处理，所以学习一下 kafka stream，由于之前了解过flink的知识，所以学习kafka stream 就不是那么困难。
#### 实时流式计算
近几年来实时流式计算发展迅速，主要原因是实时数据的价值和对于数据处理架构体系的影响。实时流式计算包含了
- Unlimit数据
- 低延迟、近实时
- 分布式集群数据一致性 
- 可以重复消费、可重复结果 


1、kafka处理unlimit数据：kafka因具备数据存储在磁盘且吞吐量大的特性，可以使得其数据无限提供了可能，而且kafka数据通过offset来定位数据的，也使得重复使用流数据更加方便。

而时间又分为事件时间和处理时间。

还有很多实时流式计算的相关概念，这里不做赘述。

#### Kafka Streams简介
Kafka Streams被认为是开发实时应用程序的最简单方法。它是一个Kafka的客户端API库，编写简单的java和scala代码就可以实现流式处理。

**优势：**
由于kafka的集群架构可以使得更具有弹性，高度可扩展，容错能力。

kafka 支持部署到容器，VM，裸机，云环境。

kafka适用范围更广，没那么重，同样适用于小型，中型和大型用例

可以与Kafka安全性完全集成

编写标准Java和Scala应用程序

系统支持 Mac，Linux，Windows

由于是kafka提供的所以支持 Exactly-once 语义

**Pom依赖**

```
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.5.0</version>
</dependency>
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>2.5.0</version>
</dependency>
```

**Topology**

Kafka Streams通过一个或多个拓扑定义其计算逻辑，其中拓扑是通过流（边缘）和流处理器（节点）构成的图。

1、pipe管道式的拓扑图

![20201218143216](https://github.com/weifangZ/image/blob/master/image20201218143216.png)

```
package com.zwf.pipe;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.Topology;
import org.apache.kafka.streams.kstream.KStream;

import java.util.Properties;
import java.util.concurrent.CountDownLatch;

/**
 * 流式操作管道
 *
 * 以上构造了有2个处理节点的kafka流计算拓扑结构，源节点：KSTREAM-SOURCE-0000000000，
 * 汇聚(Sink)节点：KSTREAM-SINK-0000000001，源节点持续的读取topic为
 * streams-plaintext-input的有序记录并输送到下游Sink节点，Sink节点再将记录写入
 * topic为streams-pipe-output的流，--> 和
 * <-- 指示左右端对象的上游和下游关系，图中有换行，导致显示不连贯拓扑展示如下：
 * PipeApplication starting .........
 * Topologies:
 *    Sub-topology: 0
 *     Source: KSTREAM-SOURCE-0000000000 (topics: [streams-plaintext-input])
 *       --> KSTREAM-SINK-0000000001
 *     Sink: KSTREAM-SINK-0000000001 (topic: streams-pipe-output)
 *       <-- KSTREAM-SOURCE-0000000000
 *
 *
 * @author zhangweifang
 * 2020年12月16日18:04:12
 */
public class PipeApplication {
    public static void main(String[] args) {
        System.out.println("PipeApplication starting .........");
        Properties props = new Properties();
        // StreamsConfig已经预定义了很多参数名称，运行时console会输出所有StreamsConfig values
        // 这里没有使用springboot的application.properties来配置
        props.put(StreamsConfig.APPLICATION_ID_CONFIG,"streams-pipe");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,"192.168.131.131:9092,192.168.131.128:9092,192.168.131.129:9092");
        // kafka流都是byte[],必须有序列化，
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,Serdes.String().getClass());

        // kafka流计算是一个各broker连接的拓扑结构，以下使用builder来构造拓扑
        final StreamsBuilder builder = new StreamsBuilder();
        // 构建一个KStream流对象，元素是<String, String>类型的key-value对值，
        KStream<String, String> source = builder.stream("streams-plaintext-input");
        // 将前面的topic："streams-plaintext-input"写入另一个topic："streams-pipe-output"
        source.to("streams-pipe-output");
        // 以上两行等同以下一行
        // builder.stream("streams-plaintext-input").to("streams-pipe-output");

        // 查看具体构建的拓扑结构
        final Topology topology = builder.build();
        System.out.println(topology.describe());

        final KafkaStreams streams = new KafkaStreams(topology,props);
        // 控制运行次数，一次后就结束
        final CountDownLatch latch = new CountDownLatch(1);

        Runtime.getRuntime().addShutdownHook(new Thread("streams-shutdown-hook"){
            @Override
            public void run() {
                streams.close();
                latch.countDown();
            }
        });

        try{
            streams.start();
            latch.await();
        }catch (Throwable e){
            System.exit(1);
        }
        System.exit(0);
    }
}

```
2、存储中间结果的双流水线
![20201218143633](https://github.com/weifangZ/image/blob/master/image20201218143633.png)

```
package com.zwf.linesplit;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.Topology;
import org.apache.kafka.streams.kstream.KStream;

import java.util.Arrays;
import java.util.Properties;
import java.util.concurrent.CountDownLatch;

/**
 * 以上构造了有3个处理节点的kafka流计算拓扑结构，源节点：KSTREAM-SOURCE-0000000000，
 * 处理节点：KSTREAM-FLATMAPVALUES-0000000001，汇聚节点：KSTREAM-SINK-0000000002，
 * 处理节点从源节点取得流元素，进行处理，再将结果传输给汇聚节点，注意这个过程是无状态的，拓扑展示如下：
 * Topologies:
 *    Sub-topology: 0
 *     Source: KSTREAM-SOURCE-0000000000 (topics: [streams-plaintext-input])
 *       --> KSTREAM-FLATMAPVALUES-0000000001
 *     Processor: KSTREAM-FLATMAPVALUES-0000000001 (stores: [])
 *       --> KSTREAM-SINK-0000000002
 *       <-- KSTREAM-SOURCE-0000000000
 *     Sink: KSTREAM-SINK-0000000002 (topic: streams-linesplit-output)
 *       <-- KSTREAM-FLATMAPVALUES-0000000001
 *
 */
public class LineSplitApplication {
    public static void main(String[] args) {
        System.out.println("LineSplitApplication starting .........");
        Properties props = new Properties();
        // StreamsConfig已经预定义了很多参数名称，运行时console会输出所有StreamsConfig values
        // 这里没有使用springboot的application.properties来配置
        props.put(StreamsConfig.APPLICATION_ID_CONFIG,"streams-line-split");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,"192.168.1.221:9092");
        // kafka流都是byte[],必须有序列化，不同的对象使用不同的序列化器
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,Serdes.String().getClass());

        // kafka流计算是一个各broker连接的拓扑结构，以下使用builder来构造拓扑
        final StreamsBuilder builder = new StreamsBuilder();
        // 构建一个KStream流对象，元素是<String, String>类型的key-value对值，
        KStream<String, String> source = builder.stream("streams-plaintext-input");
        /*
        // 以source为输入，产生一条新流words，这里使用了流的扁平化语法，我的前篇文章有讲此基础
        KStream<String, String > words = source.flatMapValues(value -> Arrays.asList("\\W+"));
        // 将前面的topic："streams-plaintext-input"写入另一个topic："streams-pipe-output"
        words.to("streams-pipe-output");*/

        // 以上两行使用stream链式语法+lambda等同以下一行，我的前篇文章有讲此基础
        source.flatMapValues(value -> Arrays.asList(value.split("\\W+")))
                .to("streams-linesplit-output");

        // 查看具体构建的拓扑结构
        final Topology topology = builder.build();
        System.out.println(topology.describe());

        final KafkaStreams streams = new KafkaStreams(topology,props);
        // 控制运行次数，一次后就结束
        final CountDownLatch latch = new CountDownLatch(1);

        Runtime.getRuntime().addShutdownHook(new Thread("streams-shutdown-hook"){
            @Override
            public void run() {
                streams.close();
                latch.countDown();
            }
        });
        try{
            streams.start();
            latch.await();
        }catch (Throwable e){
            System.exit(1);
        }
        System.exit(0);
    }
}

```
3、动态拓扑

以上两种都固定的拓扑结构，kafka stream还支持动态的拓扑结构。
```
KStreamBuilder builderD = new KStreamBuilder();
        builder.addSource("SOURCE", "chinaws__contents")
                // 添加第一个PROCESSOR，param1 定义一个processor名称，param2 processor实现类，param3 指定一个父名称 .addProcessor("PROCESS1", p , "SOURCE")
                // 添加第二个PROCESSOR，param1 定义一个processor名称， param2 processor实现类，param3 指定一个父名称 .addProcessor("PROCESS2", MyProcessor2::new, "PROCESS1")
                // 添加第三个PROCESSOR，param1 定义一个processor名称， param2 processor实现类，param3 指定一个父名称
                // .addProcessor("PROCESS3", MyProcessorC::new, "PROCESS2")
                // 最后添加SINK位置，param1 定义一个sink名称，param2 指定一个输出TOPIC，param3 指定接收哪一个PROCESSOR的数据
                .addSink("SINK1", "topicA", "PROCESS2");
                //.addSink("SINK2", "topicB", "PROCESS2")
                //.addSink("SINK3", "topicC", "PROCESS3");
        KafkaStreams streams = new KafkaStreams(builder, config);
        streams.start();
```

拓扑中有两种特殊的处理器

source：源处理器是一种特殊类型的流处理器，没有任何上游处理器。它通过使用来自这些主题的记录并将它们转发到其下游处理器，从一个或多个Kafka主题为其拓扑生成输入流。
sink：接收器处理器是一种特殊类型的流处理器，没有下游处理器。它将从其上游处理器接收的任何记录发送到指定的Kafka主题。
在正常处理器节点中，还可以把数据发给远程系统。因此，处理后的结果可以流式传输回Kafka或写入外部系统。

Kafka在这当中提供了最常用的数据转换操作，例如map，filter，join和aggregations等，简单易用。
