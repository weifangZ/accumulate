
### kafka通过什么技术使得其性能如此之快？(时间：2020年11月09-13日)

**1、数据写入**

Kafka的消息是保存或缓存在磁盘上的，一般认为在磁盘上读写数据是会降低性能的，因为寻址会比较消耗时间，但是实际上，Kafka就是利用磁盘而进行数据高吞吐率。

针对Kafka的基准测试可以参考，Apache Kafka基准测试：每秒写入2百万（在三台廉价集群上）

**kafka为什么能写入磁盘的速度那么快呢？**

Kafka会把收到的消息都写入到硬盘中，它绝对不会丢失数据。为了优化写入速度Kafka采用了两个技术， **顺序写入 和 MMFile** 。

**顺序写入**

磁盘读写的快慢取决于你怎么使用它，在顺序读写的情况下，磁盘的顺序读写速度能和内存持平。

因为硬盘是机械结构，每次读写都会寻址->写入，其中寻址是一个“机械动作”，它是最耗时的。所以硬盘最讨厌随机I/O，最喜欢顺序I/O。为了提高读写硬盘的速度，Kafka就是使用顺序I/O。

而且Linux对于磁盘的读写优化也比较多，包括read-ahead和write-behind，磁盘缓存等。如果在内存做这些操作的时候会存在两个问题，一个是JAVA对象的内存开销很大<html>
<font color='green'>*（Kafka给出的结论是内存存储对象使用的空间是磁盘存储的两倍甚至更多，也能说明我们clear系统什么要求那么大的内存）*</font></html>，另一个是随着堆内存数据的增多，JAVA的GC时间会变得很长，而使用磁盘操作有以下几个好处：

- 1、顺序写入磁盘顺序读写速度某些条件下会超过内存随机读写

- 2、顺序写入系统,即使服务重新启动，缓存依旧可用

下图就展示了Kafka是如何写入数据的， 每一个Partition其实都是一个文件 ，收到消息后Kafka会把数据插入到文件末尾（虚框部分)

![](https://ss2.baidu.com/6ON1bjeh1BF3odCf/it/u=2130536210,1147308270&fm=27&gp=0.jpg)

这种方法有一个缺陷——没有办法删除数据 ，所以Kafka是不会删除数据的，它会把所有partitions的数据都保留下来，这样就涉及到了删除log文件的数据，Kakfa提供了两种策略来删除数据：

- 基于时间 默认配置168小时（7天） log.retention.hours。
- 基于partition文件大小 默认为1Gb  log.segment.bytes

**Memory Mapped Files**

即便是顺序写入硬盘，硬盘的访问速度还是不可能追上内存。所以Kafka的数据并不是实时的写入硬盘 ，它充分利用了现代操作系统**分页缓存**（pagecache）来利用内存提高I/O效率。

Memory Mapped Files(后面简称mmap)也就是内存映射文件，在64位操作系统中一般可以表示20G的数据文件，它的工作原理是直接利用操作系统的Page来实现**文件到物理内存的直接映射**。

完成映射之后你对物理内存的操作会被同步到硬盘上（操作系统在合适的时候进行flush）。

通过mmap，进程像读写硬盘一样读写内存（当然是虚拟机内存），也不必关心内存的大小有虚拟内存为我们兜底。

使用这种方式可以获取很大的I/O提升，省去了用户空间到内核空间复制的开销（调用文件的read会把数据先放到内核空间的内存中，然后再复制到用户空间的内存中。）

但也有一个很明显的缺陷——不可靠，写到mmap中的数据并没有被真正的写到硬盘，操作系统会在程序主动调用flush的时候才把数据真正的写到硬盘。

Kafka提供了一个参数——producer.type（0.10.0后的版本kafka利用：flush.messages、flush.ms的参数进行控制，分别是按照条数和时间进行fsync的）来控制是不是主动flush，如果Kafka写入到mmap之后就立即flush然后再返回Producer叫 同步 (sync)；写入mmap之后立即返回Producer不调用flush叫异步 (async)。默认是sync，即同步发送模式。

**2、数据读取**

数据读取时候kafka进行了数据的批量压缩与结合ZeroCopy技术进行的优化。

1、将多条消息**打包**（批量压缩）后进行进行发送数据。从发送到存储在磁盘都是压缩格式，直到消费者消费才进行解压

2、broker、consumer、producer同样使用**标准二进制格式数据**，使得数据不需要修改就可以进行数据传递。

3、sendfile结合pagecache实现零拷贝

为了理解sendfile的意义，了解数据从文件到套接字的常见数据传输路径就非常重要：

- 操作系统从磁盘读取数据到内核空间的 pagecache
- 应用程序读取内核空间的数据到用户空间的缓冲区
- 应用程序将数据(用户空间的缓冲区)写回内核空间到套接字缓冲区(内核空间)
- 操作系统将数据从套接字缓冲区(内核空间)复制到通过网络发送的 NIC 缓冲区

有四次 copy 操作和两次系统调用。使用sendfile，没有用户态和内核态之间的切换，也没有内核缓冲区和用户缓冲区之间的拷贝，大大提升了传输性能。效果如下图：\
![](https://github.com/weifangZ/accumulate/blob/main/images/zero-copy.png)

又组合了操作系统提供的**pagecache** 大大的提升了数据传输性能。
Kafka重度依赖底层操作系统提供的PageCache功能。当上层有写操作时，操作系统只是将数据写入PageCache，同时标记Page属性为Dirty。当读操作发生时，先从PageCache中查找，如果发生缺页才进行磁盘调度，最终返回需要的数据。实际上PageCache是把尽可能多的空闲内存都当做了磁盘缓存来使用。同时如果有其他进程申请内存，回收PageCache的代价又很小，所以现代的OS都支持PageCache。
使用PageCache功能同时可以避免在JVM内部缓存数据，大大的提升性能

另外kafka消费者能够快速的消费指定offset以后的数据的秘诀取决于kafka数据存储的设计。

**3、kafka采取了分片(segment)+索引(index) 的机制。**


partition分区下面会分为多个segment片段（分片），每个分片对应着.log（存放实际数据）和.index（存放索引）文件。如下图示例：：
![](https://img2020.cnblogs.com/blog/1275415/202010/1275415-20201005144640768-151523346.png)

![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/D8A918A86CF546A19FEE5D9AE69A3E7F/23950)

文件不断的写入会将数据按照1Gb大小进行分割成类似下列文件：
```
00000000000000000000.index
00000000000000000000.log
00000000000000170410.index
00000000000000170410.log
00000000000000239430.index
00000000000000239430.log
```
kafka里面的topic数据存储在每个partition上如下图：

![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/D8F30AF4C4864451A4BD29EBFAEC4025/23942)

consumer通过offset（举例offset = 3）指定寻找对应的message的过程如下图：

![](https://img2020.cnblogs.com/blog/1275415/202010/1275415-20201005144757136-1003173611.png)

左侧index文件中的左边一列存放的时offset偏移量，每一条偏移量大小都是固定的。右侧存的是消息在log文件的起始位置及文件大小。

- 首先通过二分法找到对应的xxxx.index文件。
- 然后在index中通过offset*index -1文件分割大小块就能找到对应offset=3的message的数据位置及大小，这样就是可以直接定位数据的位置了。

**小结**

kafka为什么性能高
- 分区
- 顺序写磁盘
- Zero Copy

### kafka topic的pull VS push
kafka架构
![](https://ss0.baidu.com/6ON1bjeh1BF3odCf/it/u=120726874,1885633316&fm=27&gp=0.jpg)

**生产者push消息到broker，采用推模式**，生产者将消息推送给消费者。

push模式的目标是尽可能以最快速度传递消息。生产者采用pull的话，不是很适合有成千上万的生产者的情况，假如生产者写入日志，broker从日志中pull，当生产者非常多，成千上万的磁盘系统并不是时时可靠的，那样大大增加了系统的复杂性。

**broker到消费者采用pull，拉模式**。消费者主动到服务器拉取消息。

push模式很难适应消费速率不同的消费者，因为消息发送速率是由broker决定的。push模式的目标是尽可能以最快速度传递消息，但是这样很容易造成消费者来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。而pull模式则可以根据consumer的消费能力以适当的速率消费消息。

另外它有助于消费者合理的批处理消息。不同的消费者消费速率，外部硬件环境都不一样，交由消费者自己决定以何种频率拉取消息更合适。

基于pull模式不足之处在于，如果broker没有数据，消费者会轮询，忙等待数据直到数据到达，为了避免这种情况，kafka提供消费者在pull请求时候使用“long poll”进行阻塞，直到数据到达 。


举例：自助餐厅，厨师们（producers）在哪里不停地工作，有的烤羊肉串，有的切生鱼片...，做好了就放（push）在取餐区（broker/其中多个柜台相当于partition），顾客（consumers）花了69块钱进了餐厅，就相当于消费者订阅了这个topic，客户们坐在座子上准备开吃，这时候他么要去取餐区主动去拿（pull），这种模式就是kafka的生产消费者的模式。


目前kafka提供KafkaTemplate中的接口：
``` java
//topic：这里填写的是Topic的名字
//partition：这里填写的是分区的id，其实也是就第几个分区，id从0开始。表示指定发送到该分区中
//timestamp：时间戳，一般默认当前时间戳
//key：消息的键
//data：消息的数据
//ProducerRecord：消息对应的封装类，包含上述字段
//Message<?>：Spring自带的Message封装类，包含消息及消息头

//1、指定一个topic 和对应的message，他会随机选一个partition作为开始，然后将新来的数据轮询的发送到不同的partition上
ListenableFuture<SendResult<K, V>> send(String topic, V data);
//2、指定一个topic 和对应的key 与message，他会随机选一个partition作为开始，然后将新来的数据轮询的发送到不同的partition上
ListenableFuture<SendResult<K, V>> send(String topic, K key, V data);
//3、指定一个topic 和对应的key 与message，他会将新来的数据发送到指定的partition上
ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, K key, V data);
//4、指定一个topic 和对应的key 与message，他会将新来的数据发送到指定的partition上，并带指定的时间戳
ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, Long timestamp, K key, V data);
//5、通过record信息进行序列化获取topic 和对应的key 与message
ListenableFuture<SendResult<K, V>> send(ProducerRecord<K, V> record);
//6、Spring自带的Message封装类，包含消息及消息头
ListenableFuture<SendResult<K, V>> send(Message<?> message);
```

### 三、kafka常见问题汇总


**问题3**：(时间：2020年11月09-13日)
删除topic数据后，生产者写入topic（自动创建该topic）的数据很快就会自动删除。

最初清算的高可用的设计，两个点分别都在kafka中获取交易数据，当一个点坏了后，直接使用另一个点，所以配置中设置：
```
//当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据
auto-offset-reset:latest
enable-auto-commit: true
```
但是经过与场上沟通，他们的BG模块需要每日清前日的topic才能往里面写新的数据，那么我可以设置：
```
//当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
auto-offset-reset:earliest
enable-auto-commit: false
```
这样单点故障后这个点起来还可以重新将数据拉回来。



参考文献：

1、[kafka官网](http://kafka.apache.org/documentation/)\
2、[深入浅出Linux-零拷贝技术sendfile](https://www.jianshu.com/p/028cf0008ca5)\
3、[KafkaTemplate发送消息及结果回调](http://blog.seasedge.cn/archives/15.html)
