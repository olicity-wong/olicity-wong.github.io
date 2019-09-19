---
title: log4j与hadoop的简单结合
tags:
  - 日志处理
  - log4j
abbrlink: b3d37a88
date: 2018-12-25 15:00:48
---
&emsp;最近使用了一种数据存储的方法，就是使用log4j的logback将数据进行保存，然后将数据上传到hive表中，进行相关的数据分析操作。
<!--more-->
## 一、配置说明
&emsp;不多比比，感谢大佬。[logback的使用和logback.xml详解](http://www.cnblogs.com/warking/p/5710303.html)。这篇博客写的比较详细，关于logbak的相关配置文件说明。
## 二、提取需要的信息
先在业务逻辑层中提取关键信息。
这里我是简单定义一个字符串数组，将信息保存。如果有别的需求，可以自行更改提取方法。
```
public static String[] getLogMessage(String a,String b,String c,String d)
    {
        return new String[]{a,b,c,d};
    }
```
## 三、编写单独的日志打印类
```
public class TestBhLogger
{
    private static final Logger log = LoggerFactory.getLogger(TestBhLogger.class);
    public static void log(String[] array)
    {
        if ((array == null) || (array.length == 0)) {
            return;
        }
        StringBuilder sb = new StringBuilder();
        for (String str : array) {
            sb.append(str).append("\t");
        }
        if (sb.length() >= 1)
        {
            String l = sb.substring(0, sb.length() - 1);
            System.out.println(l);
            log.info(l);
        }
    }
}
```
## 四、logback-spring.xml文件配置
### 1.将日志信息输出到控制台标准输出流
```
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <pattern>%d{MM/dd HH:mm:ss.SSS} %-5level %logger{5} [%thread] - %msg%n</pattern>
    </encoder>
</appender>
```
### 2.配置编写打印日志类的日志回滚策略
1)指定要打印日志的类及日志级别  
2)将日志输出到定义目录的日志文件中  
3)定义日志回滚策略及日志文件格式  
```
<appender name="test-log" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <File>目录/logs/test/event.txt</File>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>目录/logs/test/event-%d{yyyyMMddHH}.txt</fileNamePattern>
    </rollingPolicy>
    <layout class="ch.qos.logback.classic.PatternLayout">
        <Pattern>%msg%n</Pattern>
    </layout>
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
        <level>INFO</level>
        <onMatch>ACCEPT</onMatch>
        <onMismatch>DENY</onMismatch>
    </filter>
</appender>
<logger name="com.test.log.TestBhLogger" level="INFO" >
    <appender-ref ref="test-log"/>
</logger>
```
### 3.完整的配置文件
```
<?xml version="1.0" encoding="utf-8" ?>
<configuration scan="false" scanPeriod="60 seconds" debug="false">
    <timestamp key="day" datePattern="yyyyMMdd"/>
    <timestamp key="hour" datePattern="yyyyMMddHH"/>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{MM/dd HH:mm:ss.SSS} %-5level %logger{5} [%thread] - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="operatorLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File>目录/logs/operator.log</File>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>目录/logs/operator-%d{yyyyMMddHH}.log</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>%d{MM/dd HH:mm:ss.SSS} %-5level %logger{5} [%thread] - %msg%n</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
	
	<appender name="test-log" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<File>目录/logs/test/event.txt</File>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<fileNamePattern>目录/logs/test/event-%d{yyyyMMddHH}.txt</fileNamePattern>
		</rollingPolicy>
		<layout class="ch.qos.logback.classic.PatternLayout">
			<Pattern>%msg%n</Pattern>
		</layout>
		<filter class="ch.qos.logback.classic.filter.LevelFilter">
			<level>INFO</level>
			<onMatch>ACCEPT</onMatch>
			<onMismatch>DENY</onMismatch>
		</filter>
	</appender>
	<logger name="com.test.log.TestBhLogger" level="INFO" >
		<appender-ref ref="test-log"/>
	</logger>


    <root level="info">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="operatorLog"/>
    </root>

</configuration>
```
## 五、编写激活程序
由于日志回滚需要打印日志去激活，故编写一个根据需要日志回滚的时间间隔的定时激活程序。  
这里我直接采用了Sping自带的定时任务注解@EnableScheduling
```
@Configuration
@EnableScheduling
public class AliveTask
{
    private static final Logger log = LoggerFactory.getLogger(AliveTask.class);
    SimpleDateFormat dataFormat_yyyyMMddHH = new SimpleDateFormat("yyyyMMddHH");

    @Scheduled(cron="0 10 0-23 * * ?")
    public void scheduler()
    {
        //大概就是这么个意思，具体代码根据不同需求与逻辑更改
        String[] arr = getLogMessage(a,b,c,d);
		TestBhLogger.log(arr);
    }
}
```
## 六、加载日志文件到hive表
根据不同需求获取需要的时间格式，加载到相应的hive表的相应分区中  
HiveUtil工具类网上一堆，就不细写了
```
public void loadData(){
    try
    {
        Calendar cal = Calendar.getInstance();
        cal.add(11, -1);
        String hourId = this.dataFormat_yyyyMMddHH.format(cal.getTime());
        String dayId = hourId.substring(0, 8);
        String hivesql = "LOAD DATA LOCAL INPATH '" + Main.BASE_PATH + "/logs/test/*" + hourId + "*' INTO TABLE " + Config.getString("hive.database") + ".test_bh PARTITION(day_id='" + dayId + "',hour_id='" + hourId + "')";
        HiveUtil.exec(hivesql);
    }
    catch (Exception e)
    {
        log.error("上传数据失败" + e.getMessage(), e);
    }
}
```
---
&emsp;至此就将需要的数据存储到hive表中了，接下来就是根据需求进行数据分析了。log4j真的强大。
