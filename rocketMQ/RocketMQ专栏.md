# RocketMQ专栏

> 在CSND上搜索RocketMQ的时候，发现一个专栏，觉得不错 [专栏地址](https://blog.csdn.net/qq_33333654/category_10798011.html)


下面是在学习过程中，整理的一些重要的笔记。  


## [1.MQ概述](https://blog.csdn.net/qq_33333654/article/details/126137332)


### MQ用途

- 异步
- 削峰
- 解耦


### MQ常见协议


- JMS (Java消息服务)
- STOMP (Streaming Text Orientated Message Protocol)
- AMQP (Advanced Message Queuing Protocol)
- MQTT (Message Queuing Telemetry Transport) 


### RocketMQ 简述


官⽹地址：http://rocketmq.apache.org



## [2.RocketMQ 的基本概念](https://blog.csdn.net/qq_33333654/article/details/126139833)


- 消息 Message
- 主题 Topic
- 标签 Tag    （**有没有数量限制？**）
- 队列 Queue 
- 消息标识（MessageId/Key）


## [3.RocketMQ 的系统架构](https://blog.csdn.net/qq_33333654/article/details/126141879)

- 生产者/生产者组
- 消费者/消费者组
- Name Server (独立 无状态)
- Broker (存储&转发消息)
- 路由注册
- 路由剔除
- 路由发现
- 客户端（Producer与Consumer）NameServer选择策略
- 读/写队列 （一般情况下，读/写队列数量是相同的）



### 问题


#### 每个Name Server节点上保存了哪些数据？ 

（全量Broker和Topic的注册信息）

#### Broker是如何注册到Name Server上的？

在Broker节点启动时，轮询NameServer列表，与每个NameServer节点建立长连接，发起注册请求。在NameServer内部维护着⼀个Broker列表，用来动态存储Broker的信息。


#### Name Server 无状态且保存全量注册数据的设计有什么优缺点？
- 优点：Name Server集群搭建简单
- 缺点：每个Broker必须指定Name Server，如果不指定，不会注册到上面，因此，Name Server不能随便扩容


#### Name Server的路由剔除是如何实现的？

NameServer中有⼀个定时任务，每隔10秒就会扫描⼀次Broker表，查看每一个Broker的最新心跳时间戳距离当前时间是否超过120秒，如果超过，则会判定Broker失效，然后将其从Broker列表中剔除。

#### 对于RocketMQ日常运维工作，例如Broker升级，需要停掉Broker的工作。OP需要怎么做？

- OP需要将Broker的读写权限禁掉。一旦clientConsumer或Producer向broker发送请求，都会收到broker的NO_PERMISSION响应
- 然后client会进行对其它Broker的重试。
- 当OP观察到这个Broker没有流量后，再关闭它，实现Broker从NameServer的移除。

#### RocketMQ的路由机制是怎样的？

RocketMQ的路由发现采用的是Pull模型。当Topic路由信息出现变化时，NameServer不会主动推送给客户端，而是客户端定时拉取主题最新的路由。默认客户端每30秒会拉取一次最新的路由。


#### Broker如何判断节点是Master还是Slave?

Master与Slave 的对应关系是通过指定相同的BrokerName、不同的BrokerId 来确定的。BrokerId为0表
示Master，非0表示Slave。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信
息到所有NameServer。


#### RockerMQ 工作流程

- 启动NameServer，NameServer启动后开始监听端口，等待Broker、Producer、Consumer连接。
- 启动Broker时，Broker会与所有的NameServer建立并保持长连接，然后每30秒向NameServer定时发送心跳包。
- 发送消息前，可以先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，当然，在创建Topic时也会将Topic与Broker的关系写入到NameServer中。不过，这步是可选的，也可以在发送消息时自动创建Topic。
- Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取路由信息，即当前发送的Topic消息的Queue与Broker的地址（IP+Port）的映射关系。然后根据算法策略从队选择一个Queue，与队列所在的Broker建立长连接从而向Broker发消息。当然，在获取到路由信息后，Producer会首先将路由信息缓存到本地，再每30秒从NameServer更新一次路由信息。
- Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取其所订阅Topic的路由信息，然后根据算法策略从路由信息中获取到其所要消费的Queue，然后直接跟Broker建立长连接，开始消费其中的消息。Consumer在获取到路由信息后，同样也会每30秒从NameServer更新一次路由信息。不过不同于Producer的是，Consumer还会向Broker发送心跳，以确保Broker的存活状态。





## [4.RocketMQ 单机安装及启动](https://blog.csdn.net/qq_33333654/article/details/126224308)

## [5.RocketMQ 可视化控制台的安装与启动](https://blog.csdn.net/qq_33333654/article/details/126241535)

## [6.集群搭建理论](https://blog.csdn.net/qq_33333654/article/details/126248995)

### 数据复制

- 同步复制： 消息写入master后，master会等待slave同步数据成功后才向producer返回成功ACK
- 异步复制： 消息写入master后，master立即向producer返回成功ACK，无需等待slave同步数据成功

#### 刷盘策略

刷盘策略指的是broker中消息的落盘方式，即消息发送到broker内存后消息持久化到磁盘的方式

- 同步刷盘：当消息持久化到broker的磁盘后才算是消息写入成功。
- 异步刷盘：当消息写入到broker的内存后即表示消息写入成功，无需等待消息持久化到磁盘。


### Broken集群模式

- 单Master
- 多Master （一个master宕机，可能导致大量消息丢失）
- 多Master + 多Slave + 异步复制 （Master宕机，Slave可以升级为Master，可能存在少量消息丢失）
- 多Master + 多Slave + 同步双写 （性能10%的差别，**Master宕机，Slave不会切换成Master**）



## [7.磁盘阵列RAID](https://blog.csdn.net/qq_33333654/article/details/126250421)


RAID 的基本思想是将多个容量较小、相对廉价的磁盘进行有机组合，从而以较低的成本获得与昂贵大容量磁盘相当的容量、性能、可靠性。  

RAID 主要利用镜像、数据条带和数据校验三种技术来获取高性能、可靠性、容错能力和扩展性。  


## [8.集群搭建](https://blog.csdn.net/qq_33333654/article/details/126251946)


## [9.mqadmin命令](https://blog.csdn.net/qq_33333654/article/details/126262713)


## [10.RocketMQ工作原理之消息生产及存储](https://blog.csdn.net/qq_33333654/article/details/126263247)

![](https://oscimg.oschina.net/oscnet/up-226ff0d0fc2e343296317fbbbbd1c1b5d3a.png)


### 消息生产过程

- Producer发送消息之前，会先向NameServer发出获取消息Topic的路由信息的请求
- NameServer返回该Topic的路由表及Broker列表
- Producer根据代码中指定的Queue选择策略，从Queue列表中选出一个队列，用于后续存储消息
- Produer对消息做一些特殊处理，例如，消息本身超过4M，则会对其进行压缩
- Producer向选择出的Queue所在的Broker发出RPC请求，将消息发送到选择出的Queue



### Queue选择算法

#### 轮询算法

默认选择算法。该算法保证了每个Queue中可以均匀的获取到消息。

该算法存在一个问题：由于某些原因，在某些Broker上的Queue可能投递**延迟较严重**。从而导致Producer的缓存队列中出现较大的**消息积压**，影响消息的投递性能。

#### 最小投递延迟算法

该算法会统计每次消息投递的时间延迟，然后根据统计出的结果将消息投递到时间延迟最小的Queue。如果延迟相同，则采用轮询算法投递。该算法可以有效提升消息的投递性能。


该算法也存在一个问题：消息在Queue上的分配不均匀。投递延迟小的Queue其可能会存在大量的消息。而对该Queue的消费者压力会增大，降低消息的消费能力，可能会导致MQ中消息的堆积。


### 消息存储


#### abort文件

该文件在Broker启动后会自动创建，正常关闭Broker，**该文件会自动消失**。若在没有启动Broker的情况下，发现这个文件是存在的，则说明之前Broker的关闭是非正常关闭。



#### commitlog 文件

一个Broker中仅包含一个commitlog目录，所有的mappedFile文件都是存放在该目录中的。即无论当前Broker中存放着多少Topic的消息，这些消息都是被顺序写入到了mappedFile文件中的。也就是说，这些消息在Broker中存放时并没有被按照Topic进行分类存放。


mappedFile文件是**顺序读写**的文件，所有其访问效率很高无论是SSD磁盘还是SATA磁盘，通常情况下，顺序存取效率都会高于随机存取。




#### mappedFile文件消息单元格式


mappedFile文件内容由一个个的消息单元构成。每个消息单元中包含消息总长度MsgLen、消息的物理位置physicalOffset、消息体内容Body、消息体长度BodyLength、消息主题Topic、Topic长度TopicLength、消息生产者BornHost、消息发送时间戳BornTimestamp、消息所在的队列QueueId、消息在Queue中存储的偏移量QueueOffset等近20余项消息相关属性。



#### 索引条目

每个consumequeue文件可以包含30w个索引条目，每个索引条目包含了三个消息重要属性：消息在mappedFile文件中的偏移量CommitLogOffset、消息长度、消息Tag的hashcode值。这三个属性占20个字节，所以每个文件的大小是固定的30w * 20字节。

一个consumequeue文件中所有消息的Topic一定是相同的。但每条消息的Tag可能是不同的。



#### 消息写入

一条消息进入到Broker后经历了以下几个过程才最终被持久化。

- Broker根据queueId，获取到该消息对应索引条目要在consumequeue目录中的写入偏移量，即QueueOffset

- 将queueId、queueOffset等数据，与消息一起封装为消息单元

- 将消息单元写入到commitlog ，同时，形成消息索引条目，将消息索引条目分发到相应的consumequeue。


#### 消费者拉取消息流程


当Consumer来拉取消息时会经历以下几个步骤：

- Consumer获取到其要消费消息所在Queue的消费偏移量offset ，计算出其要消费消息的消息offset

> 消费offset即消费进度，consumer对某个Queue的消费offset，即消费到了该Queue的第几条消息   消息offset = 消费offset + 1

- Consumer向Broker发送拉取请求，其中会包含其要拉取消息的Queue、消息offset及消息Tag。

> Broker计算在该consumequeue中的queueOffset。queueOffset = 消息offset * 20字节

- 从该queueOffset处开始向后查找第一个指定Tag的索引条目。解析该索引条目的前8个字节，即可定位到该消息在commitlog中的commitlog offset

- 从对应commitlog offset中读取消息单元，并发送给Consumer


#### rocketmq如何保证读写性能


RocketMQ中，无论是消息本身还是消息索引，都是存储在磁盘上的。其不会影响消息的消费吗？ 当然不会。

其实RocketMQ的性能在目前的MQ产品中性能是非常高的。因为系统通过一系列相关机制大大提升了性能。

首先，RocketMQ对文件的读写操作是通过**mmap零拷贝**进行的，将对**文件的操作转化为直接对内存地址**进行操作，从而极大地提高了文件的读写效率。

其次，consumequeue中的数据是顺序存放的，还引入了PageCache的预读取机制，使得对consumequeue文件的读取几乎接近于内存读取，即使在有消息堆积情况下也不会影响性能。



## [11.RocketMQ工作原理之indexFile](https://blog.csdn.net/qq_33333654/article/details/126269637)


### 索引文件

```
lei.gao@jidu003370 index % pwd
/Users/lei.gao/store/index
lei.gao@jidu003370 index % ls
20221102203421807
lei.gao@jidu003370 index % 
```

![](https://oscimg.oschina.net/oscnet/up-481a1ed7167b291e551b3f1ab5d2f46fc2f.png)


每个Broker中会包含一组indexFile，每个indexFile都是以一个时间戳命名的（这个indexFile被创建时的时间戳）。

每个indexFile文件由三部分构成：indexHeader，slots槽位，indexes索引数据。

每个indexFile文件中包含500w个slot槽。而每个slot槽又可能会挂载很多的index索引单元。


**索引文件结构组成**

- indexHeader
- slots槽位
- indexes索引数据



### indexHeader文件结构


- beginTimestamp：该indexFile中第一条消息的存储时间
- endTimestamp：该indexFile中最后一条消息存储时间
- beginPhyoffset：该indexFile中第一条消息在commitlog中的偏移量commitlog offset
- endPhyoffset：该indexFile中最后一条消息在commitlog中的偏移量commitlog offset
- hashSlotCount：已经填充有index的slot数量（并不是每个slot槽下都挂载有index索引单元，这里统计的是所有挂载了index索引单元的slot槽的数量）
- indexCount：该indexFile中包含的索引单元个数（统计出当前indexFile中所有slot槽下挂载的所有index索引单元的数量之和）



### index文件创建

**indexFile的文件名为当前文件被创建时的时间戳。这个时间戳有什么用处呢？**

根据业务key进行查询时，查询条件除了key之外，**还需要指定一个要查询的时间戳**，表示要查询不大于该时间戳的最新的消息，即查询指定时间戳之前存储的最新消息。这个时间戳文件名可以简化查询，提
高查询效率。具体后面会详细讲解。

#### indexFile文件是何时创建的？

其创建的条件（时机）有两个：

- 当第一条带key的消息发送来后，系统发现没有indexFile，此时会创建第一个indexFile文件

- 当一个indexFile中挂载的index索引单元数量超出2000w个时，会创建新的indexFile。当带key的消息发送到来后，系统会找到最新的indexFile，并从其indexHeader的最后4字节中读取到
indexCount。若indexCount >= 2000w时，会创建新的indexFile。

> 由于可以推算出，一个indexFile的最大大小是：(40 + 500w * 4 + 2000w * 20)字节


### index 文件查询过程

![](https://oscimg.oschina.net/oscnet/up-4543d66b83e8ccc016d52089b1ef6205d6c.png)




## [12.RocketMQ工作原理之消息的消费](https://blog.csdn.net/qq_33333654/article/details/126350496)


消费者从Broker中获取消息的方式有两种：pull拉取方式和push推动方式。

消费者组对于消息消费的模式又分为两种：集群消费Clustering和广播消费Broadcasting。


### 获取消息

#### 拉取式消费

Consumer主动从Broker中拉取消息，主动权由Consumer控制。一旦获取了批量消息，就会启动消费过程。不过，该方式的**实时性较弱**，即Broker中有了新的消息时消费者并不能及时发现并消费。

由于拉取时间间隔是由用户指定的，所以在设置该间隔时需要注意平稳：**间隔太短，空请求比例会增加；间隔太长，消息的实时性太差**


#### 推送式消费
该模式下Broker收到数据后会主动推送给Consumer。该获取方式一般实时性较高。

该获取方式是典型的**发布-订阅**模式，即Consumer向其关联的Queue注册了监听器，一旦发现有新的消息到来就会触发回调的执行，回调方法是Consumer去Queue中拉取消息。而这些都是基于**Consumer
与Broker间的长连接的**。长连接的维护是需要消耗系统资源的。

#### 对比

pull：需要应用去实现对关联Queue的遍历，实时性差；但便于应用控制消息的拉取  

push：封装了对关联Queue的遍历，实时性强，但会占用较多的系统资源


### 消费模式


#### 广播模式

广播消费模式下，相同Consumer Group的每个Consumer实例都接收同一个Topic的全量消息。即每条消息都会被发送到Consumer Group中的**每个**Consumer。  

#### 集群模式

集群消费模式下，相同Consumer Group的每个Consumer实例**平均分摊**同一个Topic的消息。即每条消息只会被发送到Consumer Group中的**某个**Consumer。



### 消费进度如何保存

- 广播模式：消费进度保存在consumer端。因为广播模式下consumer group中每个consumer都会
消费所有消息，但它们的消费进度是不同。所以consumer各自保存各自的消费进度。

- 集群模式：消费进度保存在broker中。consumer group中的所有consumer共同消费同一个Topic中的消息，同一条消息只会被消费一次。 消费进度会参与到了消费的负载均衡中，故消费进度是
需要共享的。下图是broker中存放的各个Topic的各个Queue的消费进度。




### Rebalance机制



#### 什么是Rebalance

Rebalance即再均衡，指的是，**将⼀个Topic下的多个Queue在同⼀个Consumer Group中的多个Consumer间进行重新分配的过程**


**Rebalance机制的本意是为了提升消息的并行消费能力**。

例如，⼀个Topic下5个队列，在只有1个消费者的情况下，这个消费者将负责消费这5个队列的消息。如果此时我们增加⼀个消费者，那么就可以给其中⼀个消费者分配2个队列，给另⼀个分配3个队列，从而提升消息的并行消费能力。



#### Rebalance限制

由于⼀个队列最多分配给⼀个消费者，因此当某个消费者组下的消费者实例数量大于队列的数量时，
多余的消费者实例将分配不到任何队列。


#### Rebalance危害

Rebalance的在提升消费能力的同时，也带来一些问题：

**消费暂停**：在只有一个Consumer时，其负责消费所有队列；在新增了一个Consumer后会触发Rebalance的发生。此时原Consumer就需要暂停部分队列的消费，等到这些队列分配给新的Consumer
后，这些暂停消费的队列才能继续被消费。

**消费重复**：Consumer在消费新分配给自己的队列时，必须接着之前Consumer提交的消费进度的offset继续消费。

然而默认情况下，offset是异步提交的，这个异步性导致提交到Broker的offset与Consumer实际消费的消息并不一致。这个不一致的差值就是可能会重复消费的消息。

**消费突刺**：由于Rebalance可能导致重复消费，如果需要重复消费的消息过多，或者因为Rebalance暂停时间过长从而导致积压了部分消息。那么有可能会导致在Rebalance结束之后瞬间需要消费很多消息。


#### 消费者offset提交方式


##### 同步提交
consumer提交了其消费完毕的**一批消息**的offset给broker后，需要等待broker的成功ACK。当收到ACK后，consumer才会继续获取并消费下一批消息。在等待ACK期间，consumer
是阻塞的。

##### 异步提交

consumer提交了其消费完毕的一批消息的offset给broker后，不需要等待broker的成功ACK。consumer可以直接获取并消费下一批消息。

**对于一次性读取消息的数量**，需要根据具体业务场景选择一个相对均衡的是很有必要的。

因为数量过大，系统性能提升了，但产生重复消费的消息数量可能会增加；数量过小，系统性能会下降，但被重复消费的消息数量可能会减少。


#### Rebalance产生的原因


导致Rebalance产生的原因，无非就两个：

- 消费者所订阅Topic的Queue数量发生变化
	- Broker扩容或缩容
	- Broker升级运维
	- Broker与NameServer间的网络异常
	- Queue扩容或缩容
- 消费者组中消费者的数量发生变化
	- Consumer Group扩容或缩容
	- Consumer升级运维
	- Consumer与NameServer间网络异常



#### Rebalance过程


在Broker中维护着多个Map集合，这些集合中动态存放着当前Topic中Queue的信息、Consumer Group中Consumer实例的信息。一旦发现消费者所订阅的Queue数量发生变化，或消费者组中消费者的数量
发生变化，立即向Consumer Group中的每个实例发出Rebalance通知。

Consumer实例在接收到通知后会采用Queue分配算法自己获取到相应的Queue，即由Consumer实例自主进行Rebalance。



### Queue分配算法


一个Topic中的Queue只能由Consumer Group中的一个Consumer进行消费，而一个Consumer可以同时消费多个Queue中的消息。那么Queue与Consumer间的配对关系是如何确定的，即Queue要分配给哪
个Consumer进行消费，也是有算法策略的。常见的有四种策略。这些策略是通过在创建Consumer时的构造器传进去的。


#### 平均分配策略

该算法是要根据avg = QueueCount / ConsumerCount 的计算结果进行分配的。如果能够整除，则按顺序将avg个Queue逐个分配Consumer；如果不能整除，则将多余出的Queue按照Consumer顺序
逐个分配。

该算法即，先计算好每个Consumer应该分得几个Queue，然后再依次将这些数量的Queue逐个分配个Consumer。

queue0,queue1,queue2 --> comsumer1  

queue3,queue4,queue5 --> comsumer2  

queue6,queue7,queue8 --> comsumer3  




#### 环形平均策略


环形平均算法是指，根据消费者的顺序，依次在由queue队列组成的环形图中逐个分配。  

该算法不用事先计算每个Consumer需要分配几个Queue，直接一个一个分即可。



#### 一致性hash策略


该算法会将consumer的hash值作为Node节点存放到hash环上，然后将queue的hash值也放到hash环上，通过顺时针方向，距离queue最近的那个consumer就是该queue要分配的consumer。


该算法存在的问题：**分配不均**。 因为不涉及到大数据量。

**一致性hash算法存在的意义**：其可以有效减少由于消费者组扩容或缩容所带来的大量的Rebalance。


#### 同机房策略


该算法会根据queue的部署机房位置和consumer的位置，过滤出当前consumer相同机房的queue。然后按照平均分配策略或环形平均策略对同机房queue进行分配。如果没有同机房queue，则按照平均分
配策略或环形平均策略对所有queue进行分配。



## [13.RocketMQ工作原理之订阅关系的一致性](https://blog.csdn.net/qq_33333654/article/details/126360140)


订阅关系的一致性指的是，同一个消费者组（Group ID相同）下所有Consumer实例所订阅的Topic与Tag及对消息的处理逻辑必须完全一致。否则，消息消费的逻辑就会混乱，甚至导致消息丢失。




## [14.RocketMQ工作原理之offset管理](https://blog.csdn.net/qq_33333654/article/details/126360904)

### offset用途


消费者是如何从最开始持续消费消息的？消费者要消费的第一条消息的起始位置是用户自己通过consumer.setConsumeFromWhere()方法指定的。

在Consumer启动后，其要消费的第一条消息的起始位置常用的有三种，这三种位置可以通过枚举类型常量设置。这个枚举类型为ConsumeFromWhere。


```java
public enum ConsumeFromWhere {
    CONSUME_FROM_LAST_OFFSET,
    /** @deprecated */
    @Deprecated
    CONSUME_FROM_LAST_OFFSET_AND_FROM_MIN_WHEN_BOOT_FIRST,
    /** @deprecated */
    @Deprecated
    CONSUME_FROM_MIN_OFFSET,
    /** @deprecated */
    @Deprecated
    CONSUME_FROM_MAX_OFFSET,
    CONSUME_FROM_FIRST_OFFSET,
    CONSUME_FROM_TIMESTAMP;

    private ConsumeFromWhere() {
    }
}
```




## [15.RocketMQ工作原理之消费幂等](https://blog.csdn.net/qq_33333654/article/details/126362099)


### 15.1 什么是消息幂等


当出现消费者对某条消息重复消费的情况时，重复消费的结果与消费一次的结果是相同的，并且多次消费并未对业务系统产生任何负面影响，那么这个消费过程就是消费幂等的。



### 15.2 消息重复的场景分析

什么情况下可能会出现消息被重复消费呢？最常见的有以下三种情况：

#### 15.2.1 发送时消息重复

当一条消息已被成功发送到Broker并完成持久化，此时出现了网络闪断，从而导致Broker对Producer应答失败。 如果此时Producer意识到消息发送失败并尝试再次发送消息，此时Broker中就可能会出现两条内容相同并且Message ID也相同的消息，那么后续Consumer就一定会消费两次该消息。

#### 15.2.2 消费时消息重复

消息已投递到Consumer并完成业务处理，当Consumer给Broker反馈应答时网络闪断，Broker没有接收到消费成功响应。为了保证消息至少被消费一次的原则，Broker将在网络恢复后再次尝试投递之前已被处理过的消息。此时消费者就会收到与之前处理过的内容相同、Message ID也相同的消息。

#### 15.2.3 Rebalance时消息重复

当Consumer Group中的Consumer数量发生变化时，或其订阅的Topic的Queue数量发生变化时，会触发Rebalance，此时Consumer可能会收到曾经被消费过的消息。




### 15.3 通用解决方案


幂等解决方案的设计中涉及到两项要素：幂等令牌，与唯一性处理。只要充分利用好这两要素，就可以设计出好的幂等解决方案。

- **幂等令牌**：是生产者和消费者两者中的既定协议，通常指具备唯⼀业务标识的字符串。例如，订单号、流水号。一般由Producer随着消息一同发送来的。
- **唯一性处理**：服务端通过采用⼀定的算法策略，保证同⼀个业务逻辑不会被重复执行成功多次。

例如，对同一笔订单的多次支付操作，只会成功一次。





## [16.RocketMQ工作原理之消息堆积与消费延迟](https://blog.csdn.net/qq_33333654/article/details/126370632)

消息处理流程中，如果Consumer的消费速度跟不上Producer的发送速度，MQ中未处理的消息会越来越多（进的多出的少），这部分消息就被称为堆积消息。消息出现堆积进而会造成消息的消费延迟。

以下场景需要重点关注消息堆积和消费延迟问题：

- 业务系统上下游能力不匹配造成的持续堆积，且无法自行恢复。
- 业务系统对消息的消费实时性要求较高，即使是短暂的堆积造成的消费延迟也无法接受。


消息堆积的主要瓶颈在于客户端的消费能力，而消费能力由消费耗时和消费并发度决定。注意，消费耗时的优先级要高于消费并发度。即在保证了**消费耗时**的合理性前提下，再考虑消费并发度问题。



### 16.1 消息耗时

影响消息处理时长的主要因素是代码逻辑。而代码逻辑中可能会影响处理时长代码主要有两种类型：

- CPU内部计算型代码
- 外部I/O操作型代码。

通常情况下代码中如果没有复杂的递归和循环的话，内部计算耗时相对外部I/O操作来说几乎可以忽略。所以外部IO型代码是影响消息处理时长的主要症结所在。



### 16.2 消息并发度

一般情况下，消费者端的消费并发度由单节点线程数和节点数量共同决定，其值为单节点线程数节点数量。不过，通常需要优先调整单节点的线程数，若单机硬件资源达到了上限，则需要通过横向扩展来提高消费并发度。

- 单节点线程数，即单个Consumer所包含的线程数量
- 节点数量，即Consumer Group所包含的Consumer数量



### 16.3 如何避免消息积压

为了避免在业务使用时出现非预期的消息堆积和消费延迟问题，需要在前期设计阶段对整个业务逻辑进行完善的排查和梳理。其中最重要的就是梳理消息的消费耗时和设置消息消费的并发度。

#### 16.3.1 梳理消息的消费耗时
梳理消息的消费耗时通过压测获取消息的消费耗时，并对耗时较高的操作的代码逻辑进行分析。梳理消息的消费耗时需要关注以下信息：

- 消息消费逻辑的计算复杂度是否过高，代码是否存在无限循环和递归等缺陷。
- 消息消费逻辑中的I/O操作是否是必须的，能否用本地缓存等方案规避。
- 消费逻辑中的复杂耗时的操作是否可以做异步化处理。如果可以，是否会造成逻辑错乱。


#### 16.3.2 设置消费并发度

对于消息消费并发度的计算，可以通过以下两步实施：

- 逐步调大单个Consumer节点的线程数，并观测节点的系统指标，得到单个节点最优的消费线程数和消息吞吐量。

- 根据上下游链路的流量峰值计算出需要设置的节点数

> 节点数 = 流量峰值 / 单个节点消息吞吐量



## [17.RocketMQ工作原理之消息的清理](https://blog.csdn.net/qq_33333654/article/details/126380097)


**消息被消费过后会被清理掉吗？**不会的。

消息是被顺序存储在commitlog文件的，且消息大小不定长，所以消息的清理是不可能以消息为单位进行清理的，而是以commitlog文件为单位进行清理的。否则会急剧下降清理效率，并实现逻辑复杂。

commitlog文件存在一个过期时间，默认为72小时，即三天。除了用户手动清理外，在以下情况下也会被自动清理，无论文件中的消息是否被消费过：

- 文件过期，且到达清理时间点（默认为凌晨4点）后，自动清理过期文件
- 文件过期，且磁盘空间占用率已达过期清理警戒线（默认75%）后，无论是否达到清理时间点，都会自动清理过期文件
- 磁盘占用率达到清理警戒线（默认85%）后，开始按照设定好的规则清理文件，无论是否过期。默认会从最老的文件开始清理
- 磁盘占用率达到系统危险警戒线（默认90%）后，Broker将拒绝消息写入



**需要注意以下几点：**

- 对于RocketMQ系统来说，删除一个1G大小的文件，是一个压力巨大的IO操作。在删除过程中，系统性能会骤然下降。所以，其默认清理时间点为凌晨4点，访问量最小的时间。也正因如果，我们要保障磁盘空间的空闲率，不要使系统出现在其它时间点删除commitlog文件的情况。

- 官方建议RocketMQ服务的Linux文件系统采用ext4。因为对于文件删除操作，ext4要比ext3性能更好。






## [18.RocketMQ应用之普通消息](https://blog.csdn.net/qq_33333654/article/details/126381974)


- 同步消息
- 异步消息
- 单向消息


## [19.RocketMQ应用之顺序消息](https://blog.csdn.net/qq_33333654/article/details/126388207)


### 19.1 顺序消息的定义

顺序消息指的是，严格按照消息的发送顺序进行消费的消息(FIFO)。

默认情况下生产者会把消息以Round Robin轮询方式发送到不同的Queue分区队列；而消费消息时会从多个Queue上拉取消息，这种情况下的发送和消费是不能保证顺序的。**如果将消息仅发送到同一个Queue中，消费时也只从这个Queue上拉取消息，就严格保证了消息的顺序性。**


### 19.2 有序性分类

#### 19.2.1 全局有序性

当发送和消费参与的Queue只有一个时所保证的有序是整个Topic中消息的顺序， 称为全局有序。

在创建Topic时指定Queue的数量。有三种指定方式：

1. 在代码中创建Producer时，可以指定其自动创建的Topic的Queue数量
2. 在RocketMQ可视化控制台中手动创建Topic时指定Queue数量
3. 使用mqadmin命令手动创建Topic时指定Queue数量

#### 19.2.2 分区有序性

如果有多个Queue参与，其仅可保证在该Queue分区队列上的消息顺序，则称为分区有序。


**如何实现Queue的选择？**

在定义Producer时我们可以指定**消息队列选择器**，而这个选择器是我们自己实现了`MessageQueueSelector`接口定义的。
在定义选择器的选择算法时，一般需要使用选择key。这个选择key可以是消息key也可以是其它数据。但无论谁做选择key，都不能重复，都是唯一的。

**一般性的选择算法是，让选择key（或其hash值）与该Topic所包含的Queue的数量取模，其结果即为选择出的Queue的QueueId。**

取模算法存在一个问题：不同选择key与Queue数量取模结果可能会是相同的，即不同选择key的消息可能会出现在相同的Queue，即同一个Consuemr可能会消费到不同选择key的消息。

这个问题如何解决？一般性的作法是，从消息中获取到选择key，对其进行判断。若是当前Consumer需要消费的消息，则直接消费，否则，什么也不做。这种做法要求选择key要能够随着消息一起被Consumer获取到。此时使用消息key作为选择key是比较好的做法。

以上做法会不会出现如下新的问题呢？

不属于那个Consumer的消息被拉取走了，那么应该消费该消息的Consumer是否还能再消费该消息呢？同一个Queue中的消息不可能被同一个Group中的不同Consumer同时消费。所以，消费现一个Queue的不同选择key的消息的Consumer一定属于不同的Group。而不同的Group中的Consumer间的消费是相互隔离的，互不影响的。



## [20.RocketMQ应用之延时消息](https://blog.csdn.net/qq_33333654/article/details/126390649)

### 20.1 什么是延时消息

**当消息写入到Broker后，在指定的时长后才可被消费处理的消息，称为延时消息。**

采用RocketMQ的延时消息可以实现定时任务的功能，而无需使用定时器。典型的应用场景是，电商交易中超时未支付关闭订单的场景，12306平台订票超时未支付取消订票的场景。

> 在电商平台中，订单创建时会发送一条延迟消息。这条消息将会在30分钟后投递给后台业务系统（Consumer），后台业务系统收到该消息后会判断对应的订单是否已经完成支付。如果未完成，则取消订单，将商品再次放回到库存；如果完成支付，则忽略。  

> 在12306平台中，车票预订成功后就会发送一条延迟消息。这条消息将会在45分钟后投递给后台业务系统（Consumer），后台业务系统收到该消息后会判断对应的订单是否已经完成支付。如果未完成，则取消预订，将车票再次放回到票池；如果完成支付，则忽略。



### 20.2 延时消息实现原理

![](https://img-blog.csdnimg.cn/23ede6b155e44f488c6147b027e53db4.png)


#### 20.2.1 修改消息


Producer将消息发送到Broker后，Broker会首先将消息写入到commitlog文件，然后需要将其分发到相应的consumequeue。不过，在分发之前，系统会先判断消息中是否带有延时等级。若没有，则直接正常分发；若有则需要经历一个复杂的过程：

1. 修改消息的Topic为SCHEDULE_TOPIC_XXXX
2. 根据延时等级，在consumequeue目录中SCHEDULE_TOPIC_XXXX主题下创建出相应的queueId目录与consumequeue文件（如果没有这些目录与文件的话）。


> 延迟等级delayLevel与queueId的对应关系为queueId = delayLevel -1
需要注意，在创建queueId目录时，并不是一次性地将所有延迟等级对应的目录全部创建完毕，而是用到哪个延迟等级创建哪个目录


3. 修改消息索引单元内容。索引单元中的Message Tag HashCode部分原本存放的是消息的Tag的Hash值。现修改为消息的投递时间。投递时间是指该消息被重新修改为原Topic后再次被写入到commitlog中的时间。投递时间 = 消息存储时间 + 延时等级时间。消息存储时间指的是消息被发送到Broker时的时间戳。

4. 将消息索引写入到SCHEDULE_TOPIC_XXXX主题下相应的consumequeue中



#### 20.2.2 投递延时消息

Broker内部有⼀个延迟消息服务类ScheuleMessageService，其会消费SCHEDULE_TOPIC_XXXX中的消息，即按照每条消息的投递时间，将延时消息投递到⽬标Topic中。不过，在投递之前会从commitlog中将原来写入的消息再次读出，并将其原来的延时等级设置为0，即原消息变为了一条不延迟的普通消息。然后再次将消息投递到目标Topic中。

> ScheuleMessageService在Broker启动时，会创建并启动一个定时器TImer，用于执行相应的定时任务。系统会根据延时等级的个数，定义相应数量的TimerTask，每个TimerTask负责一个延迟等级消息的消费与投递。每个TimerTask都会检测相应Queue队列的第一条消息是否到期。若第一条消息未到期，则后面的所有消息更不会到期（消息是按照投递时间排序的）；若第一条消息到期了，则将该消息投递到目标Topic，即消费该消息。



#### 20.2.3 将消息重新写入commitlog

延迟消息服务类ScheuleMessageService将延迟消息再次发送给了commitlog，并再次形成新的消息索引条目，分发到相应Queue。

> 这其实就是一次**普通消息发送**。只不过这次的消息Producer是延迟消息服务类ScheuleMessageService。


## [21.RocketMQ应用之事务消息](https://blog.csdn.net/qq_33333654/article/details/126480488)


对于分布式事务，通俗地说就是，一次操作由若干分支操作组成，这些分支操作分属不同应用，分布在不同服务器上。分布式事务需要保证这些分支操作要么全部成功，要么全部失败。分布式事务与普通事务一样，就是为了保证操作结果的一致性。

分布式事务可以参照 👉 [链接](https://github.com/geekibli/distributed-study/tree/main/%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1)


### 21.1 RocketMQ中的消息回查设置

关于消息回查，有三个常见的属性设置。它们都在broker加载的配置文件中设置，例如：

- transactionTimeout=20，指定TM在20秒内应将最终确认状态发送给TC，否则引发消息回查。默认为60秒
- transactionCheckMax=5，指定最多回查5次，超过后将丢弃消息并记录错误日志。默认15次。
- transactionCheckInterval=10，指定设置的多次消息回查的时间间隔为10秒。默认为60秒。



## [22.RocketMQ应用之批量消息](https://blog.csdn.net/qq_33333654/article/details/126509990)


## [23.RocketMQ应用之消息过滤](https://blog.csdn.net/qq_33333654/article/details/126520624)

## [24.RocketMQ应用之消息发送重试机制](https://blog.csdn.net/qq_33333654/article/details/126525629)


### 24.1 消息发送重试机制

Producer对发送失败的消息进行重新发送的机制，称为消息发送重试机制，也称为消息重投机制。

对于消息重投，需要注意以下几点：

1. 生产者在发送消息时，若采用同步或异步发送方式，发送失败会重试，但oneway消息发送方式发送失败是没有重试机制的
2. 只有普通消息具有发送重试机制，顺序消息是没有的
3. 消息重投机制可以保证消息尽可能发送成功、不丢失，但可能会造成消息重复。消息重复在RocketMQ中是无法避免的问题
4. 消息重复在一般情况下不会发生，当出现消息量大、网络抖动，消息重复就会成为大概率事件
5. producer主动重发、consumer负载变化（发生Rebalance，不会导致消息重复，但可能出现重复消费）也会导致重复消息
6. 消息重复无法避免，但要避免消息的重复消费。

避免消息重复消费的解决方案是，为消息添加唯一标识（例如消息key），使消费者对消息进行消费判断来避免重复消费

消息发送重试有三种策略可以选择：同步发送失败策略、异步发送失败策略、消息刷盘失败策略


### 24.2 同步发送失败策略

对于普通消息，消息发送默认采用**round-robin**策略来选择所发送到的队列。如果发送失败，默认重试2次。但在重试时是不会选择上次发送失败的Broker，而是选择其它Broker。当然，若只有一个Broker其也只能发送到该Broker，但其会尽量发送到该Broker上的其它Queue。


同时，Broker还具有失败隔离功能，使Producer尽量选择未发生过发送失败的Broker作为目标Broker。其可以保证其它消息尽量不发送到问题Broker，为了提升消息发送效率，降低消息发送耗时。


#### 24.2.1 思考：让我们自己实现失败隔离功能，如何来做？


**方案一**：Producer中维护某JUC的Map集合，其key是发生失败的时间戳，value为Broker实例。Producer中还维护着一个Set集合，其中存放着所有未发生发送异常的Broker实例。选择目标Broker是从该Set集合中选择的。再定义一个定时任务，定期从Map集合中将长期未发生发送异常的Broker清理出去，并添加到Set集合。  


**方案二**：为Producer中的Broker实例添加一个标识，例如是一个AtomicBoolean属性。只要该Broker上发生过发送异常，就将其置为true。选择目标Broker就是选择该属性值为false的Broker。再定义一个定时任务，定期将Broker的该属性置为false。


**方案三**：为Producer中的Broker实例添加一个标识，例如是一个AtomicLong属性。只要该Broker上发生过发送异常，就使其值增一。选择目标Broker就是选择该属性值最小的Broker。若该值相同，采用轮询方式选择。

如果超过重试次数，则抛出异常，由Producer去保证消息不丢。当然当生产者出现RemotingException、MQClientException和MQBrokerException时，Producer会自动重投消息。


### 24.3 异步发送失败策略

异步发送失败重试时，异步重试不会选择其他broker，仅在同一个broker上做重试，所以该策略无法保证消息不丢。

```java
DefaultMQProducer producer = new DefaultMQProducer("pg");
producer.setNamesrvAddr("rocketmqOS:9876");
// 指定异步发送失败后不进行重试发送
producer.setRetryTimesWhenSendAsyncFailed(0); // true
```


### 24.4 消息刷盘失败策略
消息刷盘超时（Master或Slave）或slave不可用（slave在做数据同步时向master返回状态不是SEND_OK）时，默认是不会将消息尝试发送到其他Broker的。不过，对于重要消息可以通过在Broker的配置文件设置`retryAnotherBrokerWhenNotStoreOK`属性为true来开启。



## [25.RocketMQ应用之消息消费重试机制](https://blog.csdn.net/qq_33333654/article/details/126527152)



### 25.1 顺序消息的消费重试

对于顺序消息，当Consumer消费消息失败后，为了保证消息的顺序性，其会自动不断地进行消息重试，直到消费成功。消费重试默认间隔时间为1000毫秒。**重试期间应用会出现消息消费被阻塞的情况。**


```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("cg");
// 顺序消息消费失败的消费重试时间间隔，单位毫秒，默认为1000，其取值范围为[10,30000]
consumer.setSuspendCurrentQueueTimeMillis(100);
```

由于对顺序消息的重试是无休止的，不间断的，直至消费成功，所以，对于顺序消息的消费，务必要保证应用能够及时监控并处理消费失败的情况，避免消费被永久性阻塞。


> **注意，顺序消息没有发送失败重试机制，但具有消费失败重试机制**


### 25.2 无序消息的消费重试

对于无序消息（普通消息、延时消息、事务消息），当Consumer消费消息失败时，可以通过设置返回状态达到消息重试的效果。不过需要注意，无序消息的重试只对**集群消费**方式生效，**广播消费方式不提供失败重试特性**。即对于广播消费，消费失败后，失败消息不再重试，继续消费后续消息。



### 25.3 消费重试次数与间隔

对于无序消息集群消费下的重试消费，每条消息默认最多重试16次，但每次重试的间隔时间是不同的，会逐渐变长.  

若一条消息在一直消费失败的前提下，将会在正常消费后的第4小时46分后进行第16次重试。若仍然失败，则将消息投递到死信队列


**修改重复消费的次数**

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("cg");
// 修改消费重试次数
consumer.setMaxReconsumeTimes(10);
```

对于修改过的重试次数，将按照以下策略执行：

- 若修改值小于16，则按照指定间隔进行重试
- 若修改值大于16，则超过16次的重试时间间隔均为2小时

> 对于Consumer Group，若仅修改了一个Consumer的消费重试次数，则会应用到该Group中所有其它Consumer实例。若出现多个Consumer均做了修改的情况，则采用覆盖方式生效。即最后被修改的值会覆盖前面设置的值。



### 25.4 重试队列


对于需要重试消费的消息，并不是Consumer在等待了指定时长后再次去拉取原来的消息进行消费，而是将这些需要重试消费的消息放入到了一个特殊Topic的队列中，而后进行再次消费的。这个特殊的队列就是**重试队列**。

当出现需要进行重试消费的消息时，Broker会为**每个消费组**都设置一个Topic名称为%RETRY%consumerGroup@consumerGroup 的重试队列。


> 
- 这个重试队列是针对消息才组的，而不是针对每个Topic设置的（一个Topic的消息可以让多个消费者组进行消费，所以会为这些消费者组各创建一个重试队列）
- 只有当出现需要进行重试消费的消息时，才会为该消费者组创建重试队列。


**注意，消费重试的时间间隔与延时消费的延时等级十分相似，除了没有延时等级的前两个时间外，其它的时间都是相同的**

**Broker对于重试消息的处理是通过延时消息实现的**。先将消息保存到SCHEDULE_TOPIC_XXXX延迟队列中，延迟时间到后，会将消息投递到%RETRY%consumerGroup@consumerGroup重试队列中。


集群消费方式下，消息消费失败后若希望消费重试，则需要在消息监听器接口的实现中明确进行如下三种方式之一的配置：

方式1：返回ConsumeConcurrentlyStatus.RECONSUME_LATER（**推荐**）。  
方式2：返回Null  
方式3：抛出异常  


集群消费方式下，消息消费失败后若不希望消费重试，则在捕获到异常后同样也返回与消费成功后的相同的结果，即**ConsumeConcurrentlyStatus.CONSUME_SUCCESS**，则不进行消费重试。


## [26.RocketMQ应用之死信队列](https://blog.csdn.net/qq_33333654/article/details/126538081)



### 26.1 什么是死信队列


当一条消息初次消费失败，消息队列会自动进行消费重试；达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正确地消费该消息，此时，消息队列不会立刻将消息丢弃，而是将其发送到该消费者对应的特殊队列中。这个队列就是死信队列（Dead-Letter Queue，DLQ），而其中的消息则称为死信消息（Dead-Letter Message，DLM）。

**死信队列是用于处理无法被正常消费的消息的。**



### 26.2 死信队列的特征

- 死信队列中的消息不会再被消费者正常消费，即DLQ对于消费者是不可见的
- 死信存储有效期与正常消息相同，均为 3 天（commitlog文件的过期时间），3 天后会被自动删除
- 死信队列就是一个特殊的Topic，名称为%DLQ%consumerGroup@consumerGroup ，即**每个消费者组都有一个死信队列**
- 如果⼀个消费者组未产生死信消息，则不会为其创建相应的死信队列



### 26.3 死信队列的处理

实际上，当⼀条消息进入死信队列，就意味着系统中某些地方出现了问题，从而导致消费者无法正常消费该消息，比如代码中原本就存在Bug。因此，对于死信消息，通常需要开发人员进行特殊处理。最关键的步骤是要排查可疑因素，解决代码中可能存在的Bug，然后再将原来的死信消息再次进行投递消费。


## [27.RocketMQ 消费幂等](https://blog.csdn.net/qq_33333654/article/details/127423245)


### 27.1 什么是消息幂等
如果有⼀个操作，多次执⾏与⼀次执⾏所产⽣的影响是相同的，我们就称这个操作是幂等的。
当出现消费者对某条消息重复消费的情况时，重复消费的结果与消费⼀次的结果是相同的，并且多次消费并未对业务系统产⽣任何负⾯影响，那么这整个过程就可实现消息幂等。


### 27.2 消息重复的场景有哪些


- 发送时消息重复
- 消费消息时，ack失败，再次重试消费
- 负载均衡时消息重复（包括但不限于⽹络抖动、Broker 重启以及消费者应⽤重启）


### 27.3 消费端常⻅的幂等操作


#### 27.3.1 业务操作之前进⾏状态查询

消费端开始执⾏业务操作时，通过幂等id⾸先进⾏业务状态的查询，如：修改订单状态环节，当订单状态为成功/失败则不需要再进⾏处理。那么我们只需要在消费逻辑执⾏之前通过订单号进⾏订单状态查询，⼀旦获取到确定的订单状态则对消息进⾏提交，通知broker消息状态为：ConsumeConcurrentlyStatus.CONSUME_SUCCESS 。


#### 27.3.2 唯⼀性约束保证最后⼀道防线

上述第⼆点操作并不能保证⼀定不出现重复的数据，如：并发插⼊的场景下，如果没有乐观锁、分布式锁作为保证的前提下，很有可能出现数据的重复插⼊操作，因此我们务必要对幂等id添加唯⼀性索引，这样就能够保证在并发场景下也能保证数据的唯⼀性。

#### 27.3.3 引⼊锁机制

上述的第⼀点中，如果是并发更新的情况，没有使⽤悲观锁、乐观锁、分布式锁等机制的前提下，进⾏更新，很可能会出现多次更新导致状态的不准确。如：对订单状态的更新，业务要求订单只能从初始化->处理中，处理中->成功，处理中->失败，不允许跨状态更新。

如果没有锁机制，很可能会将初始化的订单更新为成功，成功订单更新为失败等异常的情况。⾼并发下，建议通过**状态机**的⽅式定义好业务状态的变迁，通过乐观锁、分布式锁机制保证多次更新的结果是确定的，悲观锁在并发环境不利于业务吞吐量的提⾼因此不建议使⽤。



## [28.RocketMQ 消息查询](https://blog.csdn.net/qq_33333654/article/details/127424326)





#### 28.1 消息查询方式



**Message Key 查询**：消息的key是业务开发在发送消息之前⾃⾏指定的，通常会把具有业务含义，区分度⾼的字段作为消息的key，如⽤户id，订单id等。

**Unique Key查询**：除了业务开发明确的指定消息中的key，RocketMQ⽣产者客户端在发送发送消息之前，会⾃动⽣成⼀个UNIQ_KEY，设置到消息的属性中，从逻辑上唯⼀代表⼀条消息。

**Message Id 查询**：Message Id 是消息发送后，在Broker端⽣成的，其包含了Broker的地址，和在CommitLog中的偏移信息，并会将Message Id作为发送结果的⼀部分进⾏返回。Message Id中属于精确匹配，可以唯⼀定位⼀条消息，不需要使⽤哈希索引机制，查询效率更⾼。










## 其他链接
- [两万字、三十图、二十三问，搞定RocketMQ!](https://juejin.cn/post/7083848490379378702)



## 课程链接
- [【尚硅谷】RocketMQ教程丨深度掌握MQ消息中间件](https://www.bilibili.com/video/BV1cf4y157sz?p=1&vd_source=3ff1db20d26ee8426355e893ae553d51)
- [1天刷完面试核心45问消息队列面试题（Kafka&RabbitMQ&RocketMQ）](https://www.bilibili.com/video/BV1hP4y1Y7W1/?vd_source=3ff1db20d26ee8426355e893ae553d51)




## 博客园

- [RocketMQ之一：RocketMQ介绍](https://www.cnblogs.com/weifeng1463/p/12889300.html)  
- [RocketMQ之二：分布式开放消息系统RocketMQ的原理与实践（消息的顺序问题、重复问题、可靠消息/事务消息）](https://www.cnblogs.com/duanxz/p/6053598.html)
- [RocketMQ之三：NameServier代码结构](https://www.cnblogs.com/duanxz/p/5081547.html)  
- [RocketMQ之四：RocketMq事务消息 ](https://www.cnblogs.com/duanxz/p/5063377.html)
- [RocketMQ之六：RocketMQ消息存储 ](https://www.cnblogs.com/duanxz/p/5020398.html)
- [RocketMQ之七：RocketMQ管理与监控](https://www.cnblogs.com/duanxz/p/3890046.html)
- [RocketMQ之八：重试队列，死信队列，消息轨迹](https://www.cnblogs.com/duanxz/p/3896994.html)
- [RocketMQ之九：RocketMQ消息发送流程解读](https://www.cnblogs.com/duanxz/p/4707107.html)
- [RocketMQ之十：RocketMQ消息接收源码](https://www.cnblogs.com/duanxz/p/4706352.html)

















