# RocketMq本地安装和启动




## 下载安装

http://rocketmq.apache.org/release_notes/



## 本地启动


下面是我的安装路径 **/usr/local/rocketmq/bin**  


1. 首先启动nameservce  

`nohup sh ./mqnamesrv &`  


2. 启动broker

`nohup sh ./mqbroker -n localhost:9876 &`



## 遇到的问题

ERROR: Please set the JAVA_HOME variable in your environment, We need java(x64)! !!


启动nameserver时 和 启动broker时 出现了上面的错误  

我们看一下 broker脚本的内容：  

`sh ${ROCKETMQ_HOME}/bin/runbroker.sh org.apache.rocketmq.broker.BrokerStartup $@`  


它是调用了 /bin/runbroker.sh  

我们来看一下 runbroker.sh 是如何配置环境变量的。


```
#[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_333.jdk/Contents/Home
#[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"
```

上面是我已经修改好的，可以把你的java_home配置上。




