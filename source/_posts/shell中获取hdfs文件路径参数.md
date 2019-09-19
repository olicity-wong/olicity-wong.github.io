---
title: shell中获取hdfs文件路径参数
tags:
  - linux
  - shell
abbrlink: ad3ef435
date: 2019-03-28 20:55:57
---

&emsp;算是个简单的工具吧。需求是这样的，有套脚本是不定期跑的累积表，所以需要知道上次跑到了哪天。累积表有个day_id分区，所以直接看表分区是最后的day_id就行。
<!--more-->
不多比比直接上代码
```
#!/bin/bash
hdfs_path=$1
#获取hdfs最后一个时间分区时间参数脚本
#注意分区在第几层改第二个print的参数
last_data_date=`hadoop fs -ls  $hdfs_path | awk '{print $8}' |  awk -F'/' '{print $8}' | tail -n 1`
#echo $last_data_date 
last_date=${last_data_date##*=}
echo $last_date 
```
获取其他信息同理。需要加别的条件就加grep。