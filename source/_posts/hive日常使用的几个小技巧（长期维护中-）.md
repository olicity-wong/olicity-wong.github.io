---
title: hive日常使用的几个小技巧（长期维护中...）
tags:
  - hive
abbrlink: d4f0dd5b
date: 2019-03-27 16:57:03
---
&emsp;长期维护中。。。。主要记录日常使用hive中会用到的小技巧
<!--more-->
### 1.简单查询不跑MapReduce
&emsp;如果你想直接查询（select * from table），却不想执行MapReduce，可以使用FetchTask，FetchTask不同于MapReduce任务，它不会启动mapreduce，而是直接读取文件，输出结果。  
```
<property>
  <name>hive.fetch.task.conversion</name>
  <value>minimal</value>
  <description>
    Some select queries can be converted to single FETCH task 
    minimizing latency.Currently the query should be single 
    sourced not having any subquery and should not have
    any aggregations or distincts (which incurrs RS), 
    lateral views and joins.
    1. minimal : SELECT STAR, FILTER on partition columns, LIMIT only
    2. more    : SELECT, FILTER, LIMIT only (+TABLESAMPLE, virtual columns)
  </description>
</property>
```
&emsp;该参数默认值为minimal，表示运行“select * ”并带有limit查询时候，会将其转换为FetchTask；如果参数值为more，则select某一些列并带有limit条件时，也会将其转换为FetchTask任务。  
&emsp;使用前提:单一数据源，即输入来源一个表或者分区；没有子查询；没有聚合运算和distinct；不能用于视图和join
```
实现：
set hive.fetch.task.conversion=more 
```
### 2.小文件合并
&emsp;Hive中存在过多的小文件会给namecode带来巨大的性能压力,同时小文件过多会影响JOB的执行，hadoop会将一个job转换成多个task，即使对于每个小文件也需要一个task去单独处理，task作为一个独立的jvm实例，其开启和停止的开销可能会大大超过实际的任务处理时间。hive输出最终是mr的输出，即reducer（或mapper）的输出，有多少个reducer（mapper）输出就会生成多少个输出文件，根据shuffle/sort的原理，每个文件按照某个值进行shuffle后的结果。为了防止生成过多小文件，hive可以通过配置参数在mr过程中合并小文件。而且在执行sql之前将小文件都进行Merge，也会提高程序的性能。我们可以从两个方面进行优化，其一是map执行之前将小文件进行合并会提高性能，其二是输出的时候进行合并压缩，减少IO压力。
```
Map操作之前合并小文件：
    set mapred.max.split.size=2048000000
    #每个Map最大输入大小设置为2GB（单位：字节）
    
    set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat
    #执行Map前进行小文件合并


输出时进行合并：
    sethive.merge.mapfiles = true
    #在Map-only的任务结束时合并小文件

    set hive.merge.mapredfiles= true
    #在Map-Reduce的任务结束时合并小文件

    set hive.merge.size.per.task = 1024000000
    #合并后文件的大小为1GB左右

    set hive.merge.smallfiles.avgsize=1024000000
    #当输出文件的平均大小小于1GB时，启动一个独立的map-reduce任务进行文件merge


如果需要压缩输出文件，就需要增加一个压缩编解码器，同时还有两个压缩方式和多种压缩编码器，压缩方式一个是压缩输出结果，一个是压缩中间结果，按照自己的需求选择，我需要的是gzip就选择的GzipCodec，同时也可以选择使用BZip2Codec、SnappyCodec、LzopCodec进行压缩。
    压缩文件：
    set hive.exec.compress.output=true;
    #默认false，是否对输出结果压缩

    set mapred.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec;
    #压缩格式设置

    set mapred.output.compression.type=BLOCK;
    #一共三种压缩方式（NONE, RECORD,BLOCK），BLOCK压缩率最高，一般用BLOCK。
```
```
老哥关于map和reduce个数配置的说明：
https://irwenqiang.iteye.com/blog/1535809
```
### 3.动态分区
&emsp;有时候需要根据数据去动态生成分区，这时候就需要用到动态分区
```
set hive.exec.dynamic.partition=true; 
#开启动态分区功能
set hive.exec.dynamic.partition.mode=nonstrict;  
#表示允许所有分区都是动态的，否则必须有静态分区字段
set hive.exec.max.dynamic.partitions=100000;
#表示一个动态分区语句可以创建的最大动态分区个数，超出报错
set hive.exec.max.dynamic.partitions.pernode=100000;
#表示每个maper或reducer可以允许创建的最大动态分区个数，默认是100，超出则会报错。
set hive.exec.max.created.files =10000
#全局可以创建的最大文件个数，超出报错。
```
注意：动态分区不允许主分区采用动态列而副分区采用静态列，这样将导致所有的主分区都要创建副分区静态列所定义的分区
### 4.容错
&emsp;有时候因为各种原因难免会有hive执行时出错的问题，例如个别数据不规范等。这时候需要允许部分错误发生。
```
set mapreduce.map.failures.maxpercent=10; 
#设置map任务失败的比率，可以容许10%的任务失败
set mapreduce.reduce.failures.maxpercent = 10; 
#设置reduce任务失败的比率，可以容许10%的任务失败
```
### 5.交集并集差集
&emsp;交集（join），并集（union all）。这俩简单没啥说的。差集（left outer join、not in、not exists）
```
具体参考：
https://blog.csdn.net/u010003835/article/details/80928732
```
这里涉及到left outer join和left semi join
```
#left semi join解决的是IN/EXISTS的问题
select a.id from a left semi join b on (a.id = b.id);

#left outer join解决的是a差b的问题
select a.id from a left outer join b on (a.id = b.id) where b.id is null;
```
### 6.reduce资源申请时间
&emsp;为了节省时间，map未执行完时就申请reduce资源。mapreduce.job.reduce.slowstart.completedmaps，这个参数可以控制当map任务执行到哪个比例的时候就可以开始为reduce task申请资源。  
配置在mapred-site.xml，如下
```
<property>
    <name>
         mapreduce.job.reduce.slowstart.completedmaps
    </name>
    <value>
        0.05
    </value>
    <description>
        Fraction of the number of maps in the job which should be complete before reduces are scheduled for the job.
     </description>
</property>
```
&emsp;默认map5%时申请reduce资源，开始执行reduce操作，reduce可以开始进行拷贝map结果数据和做reduce shuffle操作。    
&emsp;mapreduce.job.reduce.slowstart.completedmaps这个参数如果设置的过低，那么reduce就会过早地申请资源，造成资源浪费；如果这个参数设置的过高，比如为1，那么只有当map全部完成后，才为reduce申请资源，开始进行reduce操作，实际上是串行执行，不能采用并行方式充分利用资源。  
&emsp;如果map数量比较多，一般建议提前开始为reduce申请资源。
### 7.三种常用的判断空后赋值的方法
```
1)if(boolean testCondition, T valueTrue, T valueFalseOrNull)
说明：当条件testCondition为TRUE时，返回valueTrue；否则返回valueFalseOrNull
应用：select if(a is null,0,1) from table;

2)COALESCE(T v1, T v2, …) 
说明：返回参数中的第一个非空值；如果所有值都为NULL，那么返回NULL
应用：select coalesce(a,0) from table;

3）case a when b then c [else f] end
说明：如果a等于b，那么返回c；如果a等于d，那么返回e；否则返回f
应用：select case a
            when a is null
            then a=0
            else a
            end
      from table;
```

### 8.hive transform
&emsp;hive自定义函数除了支持udf还支持transform,可以引入脚本
```
首先添加脚本文件
add file /data/users/olicity/a.py;
select transform(a, b, c, d, e) using 'python a.py' as (f , g)
from table;
```
自己没有比较过速度，不过看大佬们说是要比udf慢很多。