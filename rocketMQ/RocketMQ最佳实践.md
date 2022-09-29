# RocketMQ最佳实践

## 一、Producer最佳实践  
一个应用尽可能用一个 Topic，消息子类型用 tags 来标识，tags 可以由应用自由设置。只有发送消息设置了tags，消费方在订阅消息时，才可以利用 tags 在 broker 做消息过滤。  

每个消息在业务层面的唯一标识码，要设置到 keys 字段，方便将来定位消息丢失问题。由于是哈希索引，请务必保证 key 尽可能唯一，这样可以避免潜在的哈希冲突。  

消息发送成功或者失败，要打印消息日志，务必要打印 sendresult 和 key 字段。  

对于消息不可丢失应用，务必要有消息重发机制。例如：消息发送失败，存储到数据库，能有定时程序尝试重发或者人工触发重发。  

某些应用如果不关注消息是否发送成功，请直接使用sendOneWay方法发送消息。  


## 二、Consumer最佳实践  

尽量使用批量方式消费方式，可以很大程度上提高消费吞吐量。  

优化每条消息消费过程  

Consumer 数量要小于等于queue的总数量，由于Topic下的queue会被相对均匀的分配给Consumer，如果 Consumer 超过queue的数量，那多余的 Consumer 将没有queue可以消费消息。  

消费过程要做到幂等（即消费端去重），RocketMQ为了保证性能并不支持严格的消息去重。  

尽量使用批量方式消费，RocketMQ消费端采用pull方式拉取消息，通过consumeMessageBatchMaxSize参数可以增加单次拉取的消息数量，可以很大程度上提高消费吞吐量。另外，提高消费并行度也可以通过增加Consumer处理线程的方式，对应参数consumeThreadMin和consumeThreadMax。  

消息发送成功或者失败，要打印消息日志。  

## 三、其他配置  

线上应该关闭autoCreateTopicEnable，即在配置文件中将其设置为false。  

**RocketMQ在发送消息时，会首先获取路由信息。如果是新的消息，由于MQServer上面还没有创建对应的Topic，这个时候，如果上面的配置打开的话，会返回默认TOPIC的（RocketMQ会在每台broker上面创建名为TBW102的TOPIC）路由信息，然后Producer会选择一台Broker发送消息，选中的broker在存储消息时，发现消息的topic还没有创建，就会自动创建topic。后果就是：以后所有该TOPIC的消息，都将发送到这台broker上，达不到负载均衡的目的**。  

所以基于目前RocketMQ的设计，建议关闭自动创建TOPIC的功能，然后根据消息量的大小，手动创建TOPIC。  