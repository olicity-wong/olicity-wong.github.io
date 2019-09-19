---
title: hadoop删库跑路？
abbrlink: bc878bda
date: 2019-01-03 21:00:01
tags:
 - hadoop
---
&emsp;emmmm...昨天hadoop删库了。当时很慌，想半天好像记起来hadoop有个类似于回收站的东西，找了一下果然有，记录下，下次删库不要急着跑路。  
<!--more-->
## 一、“回收站”
hadoop有个类似于回收站的机制，通常我们删除hdfs文件时
```
hadoop fs -rm -r olicity/tableName
```
执行命令后，并非将文件直接删除，而是将文件移动到设置的".Trash"目录下。  
## 二、配置“回收站”
默认情况下，.Trash为关闭状态，如果需要恢复误删文件，需要进行配置core-site.xml
```
<property>
    <name>fs.trash.interval</name>
    <value>100</value>
</property>
```
说明：fs.trash.interval代表删除的文件保留的时间，时间单位为分钟，默认为0代表不保存删除的文件。我们只需要设置该时间即可打开.Trash。
## 三、使用“回收站”
配置完了，就试试吧。先删库。（一定要确保配置完了，要不然还是随便删点不重要的东西吧）  
删除之后，会提示
```
19/01/04 16:24:10 INFO fs.TrashPolicyDefault: Moved: 'hdfs://路径' to trash at: hdfs://路径/.Trash/Current/路径
```
恢复文件时，只需要
```
hadoop fs -mv 
```
嗯，移动出来就可以了。
## 四、彻底删除
如果说这东西，你真的确定不想要了&&还觉得他占空间，可以彻底删除。
```
hadoop fs -rm -r 路径/.Trash/路径
```
不过轻易不要这么干，万一出事就真的得跑路了。