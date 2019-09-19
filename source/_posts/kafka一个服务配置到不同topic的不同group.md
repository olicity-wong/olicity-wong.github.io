---
title: kafka一个服务配置到不同topic的不同group
tags:
  - kafka
abbrlink: b033fd85
date: 2019-02-27 12:45:33
---
&emsp;标题比较长，实在想不出什么好的描述。大概要解决的问题就是，同一个服务同时监听多个topic，且在每个topic中的group都不相同，具体看问题描述吧。
<!--more-->
## 一、问题背景
&emsp;前几天部署了一套系统，每个服务都搭建了多个节点，而且是没有主从关系的节点。每个服务中有很多东西是放到缓存中的，配置多节点之后，相同服务的不同节点出现了缓存不一致的问题。  

## 二、问题描述
&emsp;刚开始想出一种解决方案，监听同一个topic1，每个节点分到一个group中，这样每次生产者生产消息后，kafka会将消息分发到所有group中，消息中带一个消息类型字段（mq_type）。  
各个节点由于处于不同group中都会消费此消息，然后根据mq_type判断是否该处理此消息。  
&emsp;然而，pass。原因:由于此系统（系统B）中的服务1还与系统A有消费与生产消息的关系，都放到一个topic下数据不规范。而且如果多个服务1同时消费消息，会进行读表改表操作，还得做处理。

&emsp;emmm，又想出了一种解决方案，系统B中每个节点还是分到不同的group中，当某个服务1消费到系统A发送的消息，需要刷新缓存时，该节点对所有节点通过系统B内部的消息队列topic2进行广播，各个服务接收到消费消息后根据消息类型进行缓存的更新。  
具体系统图如下：
![系统图](https://github.com/olicity-wong/Resource/blob/master/system.png?raw=true)
[图片备用链接](https://github.com/olicity-wong/Resource/blob/master/system.png?raw=true)  
&emsp;**ps：以上区分两个topic是为了规范来自不同的渠道的数据走不同的topic，如果没有这种要求完全没有必要做如下这种操作，可以直接通过group和消息内容去做区分**
&emsp;如上图，系统A通过topic1向系统B中的服务1发送消息，系统B中服务1和服务2以及他们的其他节点在系统B中通过topic2发送消息。  
&emsp;可以看出，系统B中的服务1扮演了三个角色：系统A发送消息的消费者，系统B内部消息的生产者和消费者。可以得出如下问题：
```
对于服务1，需要将其配置为监听两个topic，分别监听topic1和topic2
系统A向系统B发送消息时，服务1以及他的其他节点处于topic1的同一个group下，即只有一个服务1节点会去消费系统A发来的消息
系统B内部之间发送消息时，每个服务和节点都处于topic2的不同group下
```
&emsp;说到这里，其实就清楚很多了。其实就是想让服务1-1和他的其他节点在topic1中都处于group-A2B中，服务1-1在topic2中处于group-service1-1中，服务1-2在topic2中处于group-service1-2中。

## 三、需求实现
### 3.1 代码基础
&emsp;kafka的基础代码请参照我的以下两篇博客,本次修改都是基于这些代码的基础上改造的  
&emsp;[Kafka及Spring&Kafka整合](https://olicity-wong.github.io/post/85675a3e.html)  
&emsp;[kafka动态配置topic](https://olicity-wong.github.io/post/bf5e9970.html)
### 3.2 生产者
&emsp;kafka生产者发送消息时，会向该topic下的所有group发送消息，而每个group只会有一个消费者进行消费。所以生产者不用进行更改。
### 3.3 消费者
#### 3.3.1 消费者的配置
&emsp;以下消费者以服务1-1为例，其他节点服务同理。  
&emsp;由于同一个服务要扮演两个消费者，所以我们需要不同的配置文件用来生成不同的消费者
```
//首先是获取公共配置方法
    public Map<String, Object> getCommonPropertis(){
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, KafkaConfig.BOOTSTRAP_SERVERS);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "100");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put("auto.offset.reset", "latest");// 一般配置earliest 或者latest 值
        return props;
    }

//然后不同的用来生成不同消费者的工厂
    //topic1的消费者
    public ConsumerFactory<String, String> consumerFactoryA2B() {
        Map<String, Object> properties = getCommonPropertis();
        //所在group
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "group-A2B");
        return new DefaultKafkaConsumerFactory<String, String>(properties);
    }
    
    
    //系统B内topic2的每个服务的group我这里用服务名+ip+端口命名
    String GROUP_NAME = "service1-1-"+serviceInfoUtil.getIpAddress()+"-"+serviceInfoUtil.getLocalPort();
    
    //topic2的消费者
    public ConsumerFactory<String, String> consumerFactoryB2B(){
            Map<String, Object> properties = getCommonPropertis();
            //所在group
            properties.put(ConsumerConfig.GROUP_ID_CONFIG, GROUP_NAME);
            return new DefaultKafkaConsumerFactory<String, String>(properties);
    }
    
//再通过不同的配置工厂生成实例bean
    //topic1的消费者bean
    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> kafkaListenerContainerFactoryA2B() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactoryA2B());//通过不同工厂获取实例
        factory.setConcurrency(3);
        factory.getContainerProperties().setPollTimeout(3000);
        return factory;
    }  
    
    //topic2的消费者bean
    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> kafkaListenerContainerFactoryB2B() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactoryB2B());//通过不同工厂获取实例
        factory.setConcurrency(3);
        factory.getContainerProperties().setPollTimeout(3000);
        return factory;
    }  
    
```
#### 3.3.2 消费者的使用  
&emsp;以上消费者的配置就算完成了，接下来就可以直接使用了。
```
     /**
     * 监听B2B所有消息
     * @param record
     */
    @KafkaListener(topics = "#{'${kafka.B2B.listener_topics}'.split(',')}",containerFactory = "kafkaListenerContainerFactoryB2B")
    public void B2Bconsume(ConsumerRecord<?, ?> record){
        recordDeal(record);
    }

    /**
     * 监听A2B的所有消息
     * @param record
     */
    @KafkaListener(topics = "#{'${kafka.A2B.listener_topics}'.split(',')}",containerFactory = "kafkaListenerContainerFactoryA2B")
    public void A2Bconsume(ConsumerRecord<?, ?> record) {
        recordDeal(record);
    }
    
//containerFactory = "kafkaListenerContainerFactoryA2B"  主要就是这个containerFactory参数，用它控制是哪个实例
```
#### 3.3.3 获取服务启动的ip和端口类
```
@Configuration
public class ServiceInfoUtil {
    public static String getIpAddress() throws UnknownHostException {
        InetAddress address = InetAddress.getLocalHost();
        return address.getHostAddress();
    }
    public static String getLocalPort() throws MalformedObjectNameException {
        MBeanServer beanServer = ManagementFactory.getPlatformMBeanServer();
        Set<ObjectName> objectNames = beanServer.queryNames(new ObjectName("*:type=Connector,*"),
                Query.match(Query.attr("protocol"), Query.value("HTTP/1.1")));
        String port = objectNames.iterator().next().getKeyProperty("port");
        return port;
    }
}    
```
#### 3.3.4 最后
&emsp;这样修改后启动时，通过配置文件中的kafka.A2B.listener_topics去判断这个消费者该监听哪个topic，通过containerFactory = "kafkaListenerContainerFactoryA2B"判断这个消费者在这个topic中属于哪个group。
然后发送消息测试，成了。

## 四、感谢大佬
这几个大佬的对于kafka的group的讲解比较好：  
[KAFKA 多个消费者同一个GROUPID，只有一个能收到消息的原因](http://www.mrslee.cn/archives/54)  
[Kafka消费组(consumer group)](http://www.cnblogs.com/huxi2b/p/6223228.html)  
[springboot 集成kafka 实现多个customer不同group](https://blog.csdn.net/caijiapeng0102/article/details/80765923)  
