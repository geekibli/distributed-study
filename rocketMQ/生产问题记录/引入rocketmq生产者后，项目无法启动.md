# 引入rocketmq生产者后，项目无法启动


**背景：**

引入RocketMQ生产者之后，本地服务，无法启动。 错误信息如下：


Description:

A component required a bean of type 'org.apache.rocketmq.spring.core.RocketMQTemplate' that could not be found.

The following candidates were found but could not be injected:
	- Bean method 'rocketMQTemplate' in 'RocketMQAutoConfiguration' not loaded because @ConditionalOnBean (types: org.apache.rocketmq.client.producer.DefaultMQProducer; SearchStrategy: all) did not find any beans of type org.apache.rocketmq.client.producer.DefaultMQProducer


Action:

Consider revisiting the entries above or defining a bean of type 'org.apache.rocketmq.spring.core.RocketMQTemplate' in your configuration.




**解决办法：** 


在配置文件中添加： rocketmq.producer.group: *****



## 参考链接


https://www.cnblogs.com/lingyejun/p/10657346.html



