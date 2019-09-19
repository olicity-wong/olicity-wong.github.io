---
title: Kafka及Spring&Kafka整合
tags:
  - kafka
abbrlink: 85675a3e
date: 2018-09-13 08:05:26
---
&emsp;由于某项目的消息队列使用了Spring整合Kafka，开发中我需要使用kafka客户端模拟生产者和消费者。简单了解了一下Kafka，扫盲贴，先标记一下，日后再深入学习。  
<!--more-->
## 一、Kafka简介
### 1.1 简介
&emsp; kafka是一种高吞吐量的分布式发布订阅消息系统，它可以处理消费者规模的网站中的所有动作流数据。这种动作（网页浏览，搜索和其他用户的行动）是在现代网络上的许多社会功能的一个关键因素。这些数据通常是由于吞吐量的要求而通过处理日志和日志聚合来解决。      
&emsp;在大数据系统中，常常会碰到一个问题，整个大数据是由各个子系统组成，数据需要在各个子系统中高性能，低延迟的不停流转。传统的企业消息系统并不是非常适合大规模的数据处理。为了已在同时搞定在线应用（消息）和离线应用（数据文件，日志）Kafka就出现了。  
&emsp;简单点概括一下：Kafka是一个分布式的，可划分的，高性能，低延迟的，冗余备份的持久性的日志服务。它主要用于处理活跃的流式数据。
### 1.2 特点
    * 高吞吐量
    * 可进行持久化操作
    * 分布式
### 1.3 组件
&emsp;Topic，Broker，Partition，Message，Producer，Consumer,Zookpeer
#### 1.3.1 名词解释
    服务：
    Topic：主题，Kafka处理的消息的不同分类。
    Broker：消息代理，Kafka集群中的一个kafka服务节点称为一个broker，主要存储消息数据。存在硬盘中。每个topic都是有分区的。
    Partition：Topic物理上的分组，一个topic在broker中被分为1个或者多个partition，分区在创建topic的时候指定。
    Message：消息，是通信的基本单位，每个消息都属于一个partition
    服务相关：
    Producer：消息和数据的生产者，向Kafka的一个topic发布消息。
    Consumer：消息和数据的消费者，定于topic并处理其发布的消息。
    Zookeeper：协调kafka的正常运行。
### 1.4 应用场景
构建实时的流数据管道，可靠地获取系统和应用程序之间的数据。  
构建实时流的应用程序，对数据流进行转换或反应。  

## 二、Kafka搭建
### 2.1 安装
&emsp;教程很多，就不写了。 
### 2.2 配置
&emsp;配置文件放在kafka下config下

    * consumer.properites 消费者配置
    * producer.properties 生产者配置
    * server.properties kafka服务器的配置
        broker.id 申明当前kafka服务器在集群中的唯一ID，需配置为integer,并且集群中的每一个kafka服务器的id都应是唯一的
        listeners 申明此kafka服务器需要监听的端口号，如果是在本机上跑虚拟机运行可以不用配置本项，默认会使用localhost的地址，如果是在远程服务器上运行则必须配置，例如：
                  listeners=PLAINTEXT:// 192.168.180.128:9092。并确保服务器的9092端口能够访问
        zookeeper.connect 申明kafka所连接的zookeeper的地址 ，需配置为zookeeper的地址

&emsp;上面配置文件中listeners的配置尤其注意，刚开始整的时候，没注意自己编写producer和cusmer时报错，如下：

        Connection to node -1 could not be established. Broker may not be available.
   
&emsp;就是因为配置文件中的PLAINTEXT跟我请求的内容不同。

&emsp;具体配置教程很多，也不写了。

## 三、Kafka操作
### 3.1 Topic操作
#### 3.1.1 创建Topic
    kafka-topics.sh --create --topic hbase --zookeeper ip1:port --partitions 3 --replication-factor 1
    创建topic过程的问题，replication-factor个数不能超过broker的个数
    创建topic后，可以在../data/kafka目录查看到分区的目录
#### 3.1.2 查看Topic列表
    kafka-topics.sh --list --zookeeper ip:port
#### 3.1.3 查看某一个具体的Topic
    kafka-topics.sh --describe xxx --zookeeper ip:port
#### 3.1.4 修改Topic
    kafka-topics.sh --alter --topic topic-test --zookeeper ip:port --partitions 3
    不能修改replication-factor，以及只能对partition个数进行增加，不能减少
#### 3.1.5 删除Topic
    kafka-topics.sh --delete --topic topic-test --zookeeper ip:port
    彻底删除一个topic，需要在server.properties中配置delete.topic.enable=true，否则只是标记删除
    配置完成之后，需要重启kafka服务。
### 3.2 生产者操作
    sh kafka-console-producer.sh --broker-list ip1:port,ip2:port,ip3:port --sync --topic kafka-topic-test
    生产数据的时候需要指定：当前数据流向哪个broker，以及哪一个topic
### 3.3 消费者操作
    sh kafka-console-consumer.sh --zookeeper ip1:port,ip2:port,ip3:port --topic kafka-topic-test --from-beginning
    --from-begining 获取最新以及历史数据
    
    黑白名单（暂时未用到）
    --blacklist 后面跟需要过滤的topic的列表，使用","隔开，意思是除了列表中的topic之外，都能接收其它topic的数据
    --whitelist 后面跟需要过滤的topic的列表，使用","隔开，意思是除了列表中的topic之外，都不能接收其它topic的数据

## 四、Springboot整合Kafka
这个只是个人使用的简单的测试环境搭建，可能有很多地方有问题，以后深入学习时再检查。
### 4.1 整合
&emsp;springboot集成kafka的默认配置都在org.springframework.boot.autoconfigure.kafka包里面。直接使用即可。flag=深入学习kafka。
### 4.2 pom.xml配置
    <dependency>
       <!--引入spring和kafka整合的jar-->
    			<groupId>org.springframework.cloud</groupId>
    			<artifactId>spring-cloud-starter-stream-kafka</artifactId>
    			<exclusions>
    				<exclusion>
    					<groupId>org.apache.kafka</groupId>
    					<artifactId>kafka_2.11</artifactId>
    				</exclusion>
    				<exclusion>
    					<groupId>org.apache.kafka</groupId>
    					<artifactId>kafka-clients</artifactId>
    				</exclusion>
    				<exclusion>
    					<groupId>org.springframework.kafka</groupId>
    					<artifactId>spring-kafka</artifactId>
    				</exclusion>
    			</exclusions>
    		</dependency>
    		<dependency>
    			<groupId>org.springframework.cloud</groupId>
    			<artifactId>spring-cloud-starter-hystrix</artifactId>
    		</dependency>
    		<dependency>
    			<groupId>org.apache.kafka</groupId>
    			<artifactId>kafka_2.11</artifactId>
    			<version>1.0.1</version>
    		</dependency>
    		<dependency>
    		    <groupId>org.springframework.cloud</groupId>
    		    <artifactId>spring-cloud-task-core</artifactId>
    		</dependency>
    		<dependency>
    			<groupId>org.springframework.kafka</groupId>
    			<artifactId>spring-kafka</artifactId>
    			<version>1.3.5.RELEASE</version><!--$NO-MVN-MAN-VER$-->
    			</dependency>
    		<dependency>
### 4.3 Producer配置
    @Configuration
    @EnableKafka
    public class KafkaProducer {
    
        public Map<String, Object> producerConfigs() {
            Map<String, Object> props = new HashMap<>();
            props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, KafkaConfig.BOOTSTRAP_SERVERS);
            props.put(ProducerConfig.RETRIES_CONFIG, KafkaConfig.PRODUCER_RETRIES);
            props.put(ProducerConfig.BATCH_SIZE_CONFIG, KafkaConfig.PRODUCER_BATCH_SIZE);
            props.put(ProducerConfig.LINGER_MS_CONFIG, KafkaConfig.PRODUCER_LINGER_MS);
            props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, KafkaConfig.PRODUCER_BUFFER_MEMORY);
            props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            props.put("advertised.host.name",KafkaConfig.BOOTSTRAP_SERVERS);
            props.put(ConsumerConfig.GROUP_ID_CONFIG, "1");
    
            System.out.println("KafkaConfig.BOOTSTRAP_SERVERS:"+KafkaConfig.BOOTSTRAP_SERVERS);
            return props;
        }
    
        /** 获取工厂 */
        public ProducerFactory<String, String> producerFactory() {
            return new DefaultKafkaProducerFactory<>(producerConfigs());
        }
    
        /** 注册实例 */
        @Bean
        public KafkaTemplate<String, String> kafkaTemplate() {
            return new KafkaTemplate<>(producerFactory());
        }
    
    }
### 4.4 使用生产者
     @Autowired
     private KafkaTemplate<String, String> kafkaTemplate;
     kafkaTemplate.send("kafka-topic-test", "helloWorld");
### 4.5 Consumer配置
    @Configuration
    @EnableKafka
    public class KafkaConsumer {
        private final static Logger log = LoggerFactory.getLogger(KafkaConsumer .class);
  
        @KafkaListener(topics = {"kafka-topic-test"})
        public void consume(ConsumerRecord<?, ?> record) {
            String topic = record.topic();
            String value = record.value().toString();
    
            System.out.println("partitions:"+record.partition()+","+"offset:"+record.offset()+",value="+value);
            MqConsumerRunnable runnable = new MqConsumerRunnable(topic,value);
            executor.execute(runnable);
        }
    
        public Map<String, Object> consumerConfigs() {
            Map<String, Object> props = new HashMap<>();
            props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, KafkaConfig.BOOTSTRAP_SERVERS);
            props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
            props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "100");
            props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            props.put(ConsumerConfig.GROUP_ID_CONFIG, "1");
            props.put("auto.offset.reset", "latest");// 一般配置earliest 或者latest 值
    
            return props;
        }
    
        /** 获取工厂 */
        public ConsumerFactory<String, String> consumerFactory() {
            return new DefaultKafkaConsumerFactory<>(consumerConfigs());
        }
    
        /** 获取实例 */
        @Bean
        public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> kafkaListenerContainerFactory() {
            ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
            factory.setConsumerFactory(consumerFactory());
            factory.setConcurrency(3);
            factory.getContainerProperties().setPollTimeout(3000);
            return factory;
        }
    
    }
### 4.6 使用消费者
    public class KafkaMessageListener implements MessageListener<String, String> {
    
        private static Logger LOG = LoggerFactory.getLogger(KafkaMessageListener.class);
        @Autowired
        private AppProperties appProperties;
    
        @Override
        public void onMessage(ConsumerRecord<String, String> data) {
            LOG.info("消费消息topic：{} value {}", data.topic(), data.value());
            String topic = data.topic();
            String content = data.value();
            //可同时监听多个topic，根据不同topic处理不同的业务
            if (topic.equals("topica")) {           
                LOG.info("###############topic:{} value:{}" ,topic,content);
            } else if (topic.equals("topicb")) {
             LOG.info("###############topic:{} value:{}" ,topic,content);
            } 
        }
    }
### 4.7 注意
    kafkaTemplate.send("kafka-topic-test", "helloWorld");
    @KafkaListener(topics = {"kafka-topic-test"})
    topic需要对应
#### 4.8 使用
&emsp;本地运行以后，到kafka服务器上可以进行消费者和生产者的模拟发送与接收信息。

## 五、总结
&emsp;上述方法进行模拟测试，可以测试，但是总感觉问题很大，却又找不出问题，这个后期再说吧，先凑合用。  
&emsp;有关Kafka的具体学习，后期补上。

## 六、相关链接
感谢各位大佬：
- [kafka 基础知识梳理](https://www.cnblogs.com/yangxiaoyi/p/7359236.html)
- [kafka实战](https://www.cnblogs.com/hei12138/p/7805475.html)
- [kafka笔记整理](http://blog.51cto.com/xpleaf/2090847)
- [kafka介绍](https://www.cnblogs.com/yepei/p/6197236.html)