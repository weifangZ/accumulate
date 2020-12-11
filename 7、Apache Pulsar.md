# Apache Pulsar

通过过往的学习kafka的知识，已经对kafka有了一定的理解，那么我们先了解一下Apache Pulsar，然后将Pulsar与kafka进行对比学习

### 1、什么是Apache Pulsar
"Pulsar is a distributed pub-sub messaging platform with a very flexible messaging model and an intuitive client API."
Pulsar是pub-sub模式的分布式消息平台，拥有灵活的消息模型和直观的客户端API。

它与Kafka、RocketMQ有着相同的topic、producer、consumer等概念。

Topic名称的URL类似如下的结构：
```
{persistent|non-persistent}://tenant/namespace/topic
```
- **persistent|non-persistent**表示数据是否持久化（Pulsar支持消息持久化和非持久化两种模式）
- **Tenant**为租户
- **Namespace**一般聚合一系列相关的Topic，一个租户下可以有多个Namespace

**租户和Namespace**
![](https://images2018.cnblogs.com/blog/471426/201807/471426-20180714162820262-1813573242.png)

上图中Property即为租户，每个租户下可以有多个Namespace，每个Namespace下有多个Topic。

Namespace是Pulsar中的操作单元，包括Topic是配置在Namespace级别的，包括多地域复制，消息过期策略等都是配置在Namespace上的。

### 2、订阅模型

Pulsar提供了灵活的消息模型，支持三种订阅类型：

Exclusive subscription：排他的，只能有一个Consumer，接收一个Topic所有的消息，可以保证消息的顺序性。

Shared subscription：共享的，可以同时存在多个Consumer，每个Consumer处理Topic中一部分消息（Shared模型是不保证消息顺序的，Consumer数量可以超过分区的数量）

Failover subscription：Failover模式，同一时刻只有一个有效的Consumer，其余的Consumer作为备用节点，在Master Consumer不可用后进行替代（看起来适用于数据量小，且解决单点故障的场景）

![](https://images2018.cnblogs.com/blog/471426/201807/471426-20180714162833476-634182507.png)

### 3、分区
为了解决吞吐等问题，Pulsar和Kafka一样，采用了分区（Partition）的机制。
![](https://images2018.cnblogs.com/blog/471426/201807/471426-20180714162843060-675502339.png)

Pulsar提供了一些策略来处理消息到Partition的路由（MessageRouter）：

- Single partitioning：Producer随机选择一个Partition并将所有消息写入到这个分区\
- Round robin partitioning ：采用Round
- robin的方式，轮训所有分区进行消息写入
- Hash partitioning：这种模式每条消息有一个Key，Producer根据消息的Key的哈希值进行分区的选择（Key相同的消息可以保证顺序）。
- Custom partitioning：用户自定义路由策略
不同于别的MQ系统，Pulsar允许Consumer的数量超过分区的数量（对于RocketMQ，超过分区数的Consumer会分配不到分区而“空跑”）。

在Shared subscription的订阅模式下，Consumer数量可以大于分区的数量，每个Consumer处理每个Partition中的一部分消息，不保证消息的顺序。

### 4、持久化
Pulsar通过**BooKeeper**来存储消息，保证消息不会丢失（BooKeeper：A scalable, fault-tolerant, and low-latency storage service optimized for real-time workloads）。

pulsar基本知识了解了那么我们来分析一下pulsar与kafka的异同点。

## 通过kafka 与 pulsar 对比

1、pulsar 与kafka 都是Zero copy 但是pulsar的零拷贝更加完全

2、pulsar是多租户的
**多租户**Pulsar 采用分层架构，租户和命名空间能够与机构或团队形成良好的逻辑映射，Pulsar 通过这种相同的机构支持简易 ACL、配额、自主服务控制，甚至也支持资源隔离，从而允许集群使用者轻松管理共享集群。这样我们可以在开发和测试的时候节省机器资源。降低部署与机器成本

3、pulsar与kafka 都自带zookeeper 不需要单独下载zookeeper：
- pulsar 具有 zookeeper、 bookeeper、broker 集群，BookKeeper 节点（bookie）存储消息与游标，ZooKeeper 则只用于为 broker 和 bookie 存储元数据。
![](https://img-blog.csdnimg.cn/20200813114654908.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYWlqaWEwMw==,size_16,color_FFFFFF,t_70#pic_center)
- kafka 具有zookeeper、broker集群，Kafka 采用单片架构模型，将服务与存储相结合，而 Pulsar 则采用了多层架构，可以在单独的层内进行管理。Pulsar 中的 broker 在一个层上进行计算，而 bookie 则在另一个层上管理有状态存储。

Pulsar 的多层架构看起来似乎比 Kafka 的单片架构更为复杂，但实际情况却没这么简单。架构设计需要权衡利弊，BookKeeper 使得 Pulsar 更具可伸缩性、操作负担更低、速度更快，性能也更一致

4、 pulsar扩展性更强 为了跟上业务增长的速度，pulsar能够更加迅捷方便的扩展集群Pulsar 的自动负载均衡功能可以自动并立即使用集群中新加的计算和存储能力。这使得 broker 之间可以迁移 topic 来平衡负载，新 bookie 节点可以立即接受新数据分片的写入流量，而无需手动重新平衡或管理 broker。

5、Pulsar自带工具更加丰富，而kafka需要依赖第三方工具来进行管理kafka，且第三方有些是收费的。

6、Pulsar具有内置的复制功能，可用于无缝跨越地理区域同步数据或复制数据到其他集群，以实现其他功能（如灾备、分析等）

7、Pulsar 目前官方支持 [7种语言](http://pulsar.apache.org/docs/en/client-libraries/)（Java C C++ Python Go .NET  Node） ，而 Kafka 支持  32 种语言（C、C++、Erlang、Java、.net、perl、PHP、Python、Ryby、Go、JavaScript，...）但是，官方客户端并不能支持这么多种语言，而且一些语言已经不再维护。


8、Apache Pulsar支持不同订阅模式。单个应用程序的订阅模式由排序和消费可扩展性需求决定。

- **独占**和**灾备订阅**模式都在分区级别支持强排序保证，支持跨 consumer 并行消费同一 topic 上的消息。
- **共享订阅**模式支持将 consumer的数量扩展至超过分区的数量，因此这种模式非常适合 worker 队列用例。
- **键共享订阅**模式结合了其他订阅模式的优点，支持将 consumer的数量扩展至超过分区的数量，也支持键级别的强排序保证。


9、topic 数据存储处理

Pulsar 旨在支持用户消费数据。应用程序可以需要选择使用原始数据或压缩数据。Pulsar 通过这种按需选择的方式，允许未压缩数据通过保留策略控制无限制增长，但仍允许通过周期性压缩生成最新的实物化视图。内置的分层存储特性支持 Pulsar 从 BooKeeper 卸载未压缩数据到云存储中，因而降低长期存储事件的成本。

相比于 Pulsar，Kafka 不支持用户使用原始数据。并且，在数据压缩后，Kafka 会立即删除原始数据。


10、消息队列：
Pulsar 与kafka 同时支持队列和流


### 参考文献
[1、Pulsar 与 Kafka 全方位对比（上篇）：功能、性能、用例](https://blog.csdn.net/zhaijia03/article/details/107976374)\
[2、Pulsar 与 Kafka 全方位对比（下篇）：案例、特性、社区](https://apachepulsar.blog.csdn.net/article/details/110608326)\
[3、Pulsar集群搭建部署](https://blog.51cto.com/536410/2408686)\
[4、Pulsar官网](http://pulsar.apache.org/docs/en/client-libraries/)
