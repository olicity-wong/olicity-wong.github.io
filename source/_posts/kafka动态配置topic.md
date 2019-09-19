---
title: kafka动态配置topic
tags:
  - kafka
abbrlink: bf5e9970
date: 2019-01-11 03:31:59
---

&emsp;之前使用@org.springframework.kafka.annotation.KafkaListener这个注解的时候，是在yml文件中配置，然后使用@KafkaListener(topics = {"${kafka.topic.a2b.name}"})，这样去单独监听某一个topic，生产者也固定在代码里定义变量读取配置文件。昨天改了个需求，希望以后通过配置文件去动态配置生产者和消费者的topic（不知道个数和topic名字），而不需要改代码。   
<!--more-->
## 一、踩坑
&emsp;刚开始的时候，由于考虑不充分（没有考虑到topic个数未知），想到@KafkaListener注解中的topics本身就是个字符串数组，于是想通过传入变量的形式。产生了以下两种方法：  
### 1.传入变量方法一
&emsp; 使用@Value注解提取配置文件中相关配置，@KafkaListener中传入变量
```
    public static String[] topicArr;
    @Value("${kafka.bootstrap.servers}")
    public void setTopicArr(String[] value){
        String topicArr = value;
    }
    @KafkaListener(topics= topicArr)
```
emmmm。。。结果可想而知，不行。
### 2.传入变量方法二
&emsp;还是传入变量，不过这次写了个动态配置的代码
```
    注解里这么写
    @KafkaListener(topics = "${topicName1}","${topicName2}","${topicName3}")
    提前将yml文件里添加
    topics: topicName1,topicName2,topicName3
    然后加载进来
    @Value("${kafka.topics}")
    public void setTopics(String value){
        topics = value;
    }
    动态配置代码：
    @Configuration
    public class KafkaTopicConfiguration implements InitializingBean {
        @Autowired
        private KafkaConfig kafkaconfig;
        @Override
        public void afterPropertiesSet() throws Exception {
            String[] topicArr = kafkaconfig.split(",");
            int i = 1;
            for(String topic : topicArr){
                String topicName = "topicName"+i;
                System.setProperty(topicName, topic);
            }
        }
    }
```
相比方法一，可行。但是未知topic数量呢。GG。
### 3.不用注解
&emsp;百度找到几个老哥的动态获取并创建topic的方法
```
https://www.cnblogs.com/gaoyawei/p/7723974.html
https://www.cnblogs.com/huxi2b/p/7040617.html
https://blog.csdn.net/qq_27232757/article/details/78970830
```
写了几版，各种各样的问题，还是我太菜。就想再看看有没有别的简单点的解决办法，没有了再回来搞这个。
### 4.正则匹配topic
&emsp;这期间又找到一个使用正则匹配topic的。直接贴[链接](https://www.jianshu.com/p/4c422a6a6c7a)。
```
@KafkaListener(topicPattern = "showcase.*")
这里使用正则匹配topic，其中【*】之前得加上【.】才能匹配到。
```
中间模仿写了一版使用正则匹配的，其实也可以糊弄实现需求，除了topic取名的时候一定得规范以外，还得考虑到如果不想用某个topic了又得想怎么去避免他。  
这种方法不太严谨，继续踩坑吧。

## 二、问题解决
&emsp;用蹩脚的英语google了一下，嗯?好多老哥们也是用的以上差不多的方法。然而最后在某个老哥github的[issues](https://github.com/spring-projects/spring-kafka/issues/361)中看到了解决办法。老哥的需求跟我差不多，感谢大佬,贴上最终问题解决方案。  
### 1.kafka消费者监听配置
```
还是注解的形式
@KafkaListener(topics = "#{'${kafka.listener_topics}'.split(',')}")
```
读取yml文件中kafka.listener_topics的参数，然后根据“,”去split,得到一个topics数组。  
这么做就可以根据配置文件动态的去监听topic。

### 2.yml配置文件
```
只列出topic相关部分（mqTypes是我用来判断使用哪个topic发送的）
    kafka:
      listener_topics: kafka-topic-a2b,kafka-topic-c2b
      consume:
        topic:
          - name: kafka-topic-a2b
            partitions: 12
            replication_factor: 2
          - name: kafka-topic-c2b
            partitions: 12
            replication_factor: 2
      product:
        topic:
          - name: kafka-topic-b2a
            partitions: 12
            replication_factor: 2
            mqTypes: type1
          - name: kafka-topic-b2c
            partitions: 12
            replication_factor: 2
            mqTypes: type1
```
### 3.yml参数解析
这里我将kafka的topic相关加载到bean中处理。  
创建KafkaConsumerBean和KafkaProducerBean分别用来存储yml中生产者和消费者的topic相关参数
```
//KafkaConsumerBean
@Component
@ConfigurationProperties(prefix = "kafka.consume")
public class KafkaConsumerBean {
    private List<Map<String,String>> topic;
    public void setTopic(List<Map<String, String>> topic) {
        this.topic = topic;
    }
    public List<Map<String, String>> getTopic() {
        return topic;
    }
}

//KafkaProducerBean
@Component
@ConfigurationProperties(prefix = "kafka.product")
public class KafkaProducerBean {
    private List<Map<String,String>> topic;
    public void setTopic(List<Map<String, String>> topic) {
        this.topic = topic;
    }

    private Map<String,String> mqType2NameMap = new HashMap<String,String>();
    public List<Map<String, String>> getTopic() {
        return topic;
    }

    public String getTopic(String mqType){
        String name = mqType2NameMap.get(mqType);
        if(name != null){
            return name;
        }else{
            for(Map<String,String> topicProperty : topic){
                if (topicProperty.get("mqTypes").indexOf(mqType) >= 0){
                    name = topicProperty.get("name");
                    mqType2NameMap.put(mqType,name);
                    return name;
                }
            }
            return null;
        }

    }
}

```
### 4.创建topic
```
    List<Map<String,String>> producerTopicList = kafkaProducerBean.getTopic();
    for (Map<String,String> topicProperty : producerTopicList){
        KafkaClient.createTopic(topicProperty.get("name"),Integer.parseInt(topicProperty.get("partitions")),Integer.parseInt(topicProperty.get("replication_factor")));
    }
    List<Map<String,String>> consumerTopicList = kafkaConsumerBean.getTopic();
    for (Map<String,String> topicProperty : consumerTopicList){
        KafkaClient.createTopic(topicProperty.get("name"),Integer.parseInt(topicProperty.get("partitions")),Integer.parseInt(topicProperty.get("replication_factor")));
    }
```
## 三、总结
&emsp;上面解决问题的方法关键在于
```
@KafkaListener(topics = "#{'${kafka.listener_topics}'.split(',')}")
```
@KafkaListener这个注解会去读取spring的yml配置文件中
```
kafka:
      listener_topics: kafka-topic-a2b,kafka-topic-c2b
```
这块listener_topics配置信息，然后通过','分割成topic数组，KafkaListener注解中的 topics 参数，本身就是个数组，如下
```
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package org.springframework.kafka.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Repeatable;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import org.springframework.messaging.handler.annotation.MessageMapping;

@Target({ElementType.TYPE, ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@MessageMapping
@Documented
@Repeatable(KafkaListeners.class)
public @interface KafkaListener {
    String id() default "";

    String containerFactory() default "";

    String[] topics() default {};

    String topicPattern() default "";

    TopicPartition[] topicPartitions() default {};

    String group() default "";
}
```
&emsp;结合我之前的kafka文章，应该是可以拼出一套成型的。