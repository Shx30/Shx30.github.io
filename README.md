---
title: kafka
date: 2020-05-25 21:25:05
tags: Kafka
categories: Kafka
blogexcerpt: kafka在windows下使用
---

工作经历

2017.11-2018.6

温州鹿城档案一体机系统

作为两位核心开发人员之一参与前端和后台的开发任务，按时完成产品交付及项目验收。

2018.7-2019.6

基于BAP的MES系统生产模块开发

服务于四川三棵树，常州华科，传化合成、长沙关西涂料，东北制药等项目开发和现场技术支持。

输出生产计划模块、配方、工单模块、装箱清单模块、工艺分析模块

2019.7-至今

BAP4.2-BAP4.6 项目开发

为公司所有MES系统开发、部署提供平台环境解决业务开发和工程现场遇到的平台使用问题、产品迭代和BUG修复

输出消息中心功能模块、附件预览功能、excel导入导出功能

2019.9-至今

BAP5.0项目开发

原有BAP4.5基础上（SSH+virgo框架）改造为SpringCloud框架

参与系统功能迁移，网关+注册中心+配置中心+核心业务功能模块+工作流引擎，BAT脚本实现注册服务


---
title: kafka
date: 2020-05-25 21:25:05
tags: Kafka
categories: Kafka
blogexcerpt: kafka在windows下使用
---

因需求需要使用kafka作为消息中间价实现分布式事务一致性

### 准备工作

1.官网下载kafka安装包

```
kafka.apache.org/downloads
```

### 环境部署

1.kafka依赖于zookeeper启动

2.修改kafka/bin/windows/kafka-run-class.bat 文件 中160左右 

```
IF["%JAVA_HOME%"] EQU [""](
	set JAVA=指定JDK
)
```

3.修改kafka日志机制为日志压缩

在windows环境下kafka启用日志删除机制会出现log文件被另一进程占用问题，

策略：将日志机制改为压缩机制

```config
/kafka/config/server.properties 文件 末尾加上
log.cleaner.enable=false   //不启用日志清除  关闭日志清除后服务该问题会少许多，但是新增的topic依旧存在该问题 
log.cleanup.policy=compact  //日志压缩机制 解决该问题，需注意发送的kafka消息要带上key，不然kafka消息会发送失败。
```

### 配置项

```properties
spring.kafka.bootstrap-servers=127.0.0.1:9092
#生产者的配置，大部分我们可以使用默认的，这里列出几个比较重要的属性
#每批次发送消息的数量
spring.kafka.producer.batch-size=16
#设置大于0的值将使客户端重新发送任何数据，一旦这些数据发送失败。注意，这些重试与客户端接收到发送错误时的重试没有什么不同。允许重试将潜在的改变数据的顺序，如果这两个消息记录都是发送到同一个partition，则第一个消息失败第二个发送成功，则第二条消息会比第一条消息出现要早。
spring.kafka.producer.retries=0
#producer可以用来缓存数据的内存大小。如果数据产生速度大于向broker发送的速度，producer会阻塞或者抛出异常，以“block.on.buffer.full”来表明。这项设置将和producer能够使用的总内存相关，但并不是一个硬性的限制，因为不是producer使用的所有内存都是用于缓存。一些额外的内存会用于压缩（如果引入压缩机制），同样还有一些用于维护请求。
spring.kafka.producer.buffer-memory: 33554432
#key序列化方式
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
#消费者的配置
#Kafka中没有初始偏移或如果当前偏移在服务器上不再存在时,默认区最新 ，有三个选项 【latest, earliest, none】
spring.kafka.consumer.auto-offset-reset=latest
#是否开启自动提交
spring.kafka.consumer.enable-auto-commit=false
#自动提交的时间间隔
spring.kafka.consumer.auto-commit-interval=100
#key的解码方式
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
#消息者默认消费的groupId
spring.kafka.consumer.group-id=test-consumer-group
#RECORD 每处理一条commit一次
#BATCH(默认) 每次poll的时候批量提交一次，频率取决于每次poll的调用频率
#TIME 每次间隔ackTime的时间去commit(跟auto commit interval有什么区别呢？)
#COUNT 累积达到ackCount次的ack去commit
#COUNT_TIME ackTime或ackCount哪个条件先满足，就commit
#MANUAL listener负责ack，但是背后也是批量上去
#MANUAL_IMMEDIATE listner负责ack，每调用一次，就立即commit
spring.kafka.listener.ack-mode=manual_immediate
#每个@KafkaListener的并发个数
spring.kafka.listener.concurrency=3
```

### 代码使用

1.pom依赖

```XML
<dependency>
	<groupId>org.springframework.kafka</groupId>
	<artifactId>spring-kafka</artifactId>
</dependency>
```

2.生产者

当日志机制为日志压缩时key必带，否则发送消息会失败

```java
@Component
public class Producer {
    @Autowired
    private KafkaTemplate kafkaTemplate;
    private static Gson gson = new GsonBuilder().create();
    //发送消息方法Map
    public void send(String topic, Map message ) {
        kafkaTemplate.send(topic,"kafkaKey", gson.toJson(message));
    }
}
```

3.消费者

​	3.1 消费者可以处理多个topics，一个topic多个groupId可以收到消息，

​	3.2 当ack-mode为manual时才存在Acknowledgment，否则服务启动连接kafka会报错

```java
@Component
public class Consumer2 {
    @KafkaListener(topics = {"mytopic"},groupId = "msgstr")
    public void listen(ConsumerRecord<?, ?> record, Acknowledgment ack) {
        Optional<?> kafkaMessage = Optional.ofNullable(record.value());
        if (kafkaMessage.isPresent()) {
            Object message = kafkaMessage.get();
            ack.acknowledge();
            try {
                Gson gson = new Gson();
                Map<String, Object> map = new HashMap<>();
                map = gson.fromJson((String)message, map.getClass());
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }

}
```

### 概念

Kafka中发布订阅的对象是topic。我们可以为每类数据创建一个topic，把向topic发布消息的客户端称作producer，从topic订阅消息的客户端称作consumer。Producers和consumers可以同时从多个topic读写数据。一个kafka集群由一个或多个broker服务器组成，它负责持久化和备份具体的kafka消息。
Broker：Kafka节点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。
Topic：一类消息，消息存放的目录即主题，例如page view日志、click日志等都可以以topic的形式存在，Kafka集群能够同时负责多个topic的分发。
Partition：topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列
Segment：partition物理上由多个segment组成，每个Segment存着message信息
Producer : 生产message发送到topic下的partition leader.
Consumer : 订阅topic消费message, consumer作为一个线程来消费
Consumer Group：一个Consumer Group包含多个consumer, 这个是预先在配置文件中配置好的。

![1590645604004](C:\Users\huangxin2\AppData\Roaming\Typora\typora-user-images\1590645604004.png)

kafka选举机制

通过zookeeper注册中心管理kafka集群中选举一个存活的breaker作为leader否则leader为-1

![1590645768676](C:\Users\huangxin2\AppData\Roaming\Typora\typora-user-images\1590645768676.png)

消费者策略

- *partition中的每个message只能被组（Consumer group ）中的一个consumer（consumer 线程）消费*

 只允许同一个consumer group下的一个consumer线程去访问一个partition。如果觉得效率不高的时候，可以加partition的数量来横向扩展，那么再加新的consumer thread去消费。如果想多个不同的业务都需要这个topic的数据，起多个consumer group就好了，大家都是顺序的读取message，offsite的值互不影响。这样没有锁竞争，充分发挥了横向的扩展性，吞吐量极高。这也就形成了分布式消费的概念。
