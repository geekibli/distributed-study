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






## [RocketMQ应用之普通消息]()





## 其他链接
- [两万字、三十图、二十三问，搞定RocketMQ!](https://juejin.cn/post/7083848490379378702)



## 课程链接
- [【尚硅谷】RocketMQ教程丨深度掌握MQ消息中间件](https://www.bilibili.com/video/BV1cf4y157sz?p=1&vd_source=3ff1db20d26ee8426355e893ae553d51)
- [1天刷完面试核心45问消息队列面试题（Kafka&RabbitMQ&RocketMQ）](https://www.bilibili.com/video/BV1hP4y1Y7W1/?vd_source=3ff1db20d26ee8426355e893ae553d51)





















