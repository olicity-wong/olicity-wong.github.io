---
title: crontab使用时间参数
tags:
  - linux
  - shell
abbrlink: 78cb4de
date: 2018-12-25 16:02:00
---
&emsp;最近写了一个脚本，需要定时执行，决定使用Crontab。
<!--more-->
## 一、问题描述
&emsp;由于脚本需要传入时间参数，传入时间时发生了不能执行的问题，如下。  
刚开始写的调用脚本为
```
20 0 * * * source ~/.bash_profile;cd */shell;sh count.sh $(date +%Y%m%d)> count.log 2>&1
```
定时执行不能执行，查看系统日志后，发现错误位置在$(date。
## 二、问题解决
&emsp;看了网上发现问题所在。原因应该是，任务调度中，%是个特殊字符，表示特殊含义，有换行的意思。所以不能直接使用%，而需要添加反斜杠来进行转义。
修改后的调用脚本为
```
20 0 * * * source ~/.bash_profile;cd */shell;sh count.sh $(date +"\%Y\%m\%d")> count.log 2>&1^C
```
然后问题解决了。
## 三、拓展
&emsp;既然用到了crontab，就简单的学习一下吧。
### 1.时间参数说明
```
20 0 * * *
从左到右依次为：
[分钟] [小时] [每月的某一天] [每年的某一月] [每周的某一天] [执行的命令]
该参数的意义为：每天的0点20分执行脚本
星号（*）：代表所有可能的值，例如month字段如果是星号，则表示在满足其它字段的制约条件后每月都执行该命令操作
逗号（,）：可以用逗号隔开的值指定一个列表范围，例如，“1,2,5,7,8,9”
中杠（-）：可以用整数之间的中杠表示一个整数范围，例如“2-6”表示“2,3,4,5,6”
正斜线（/）：可以用正斜线指定时间的间隔频率，例如“0-23/2”表示每两小时执行一次。同时正斜线可以和星号一起使用，例如*/10
@reboot 系统重启时执行
```
### 2.添加/编辑 Crontab
```
crontab [-u username] -e
默认情况下，系统会编辑当前用户的crontab命令集合
```
### 3.查看Crontab
```
crontab [-u username] -l
```
### 4.删除Crontab
```
crontab [-u username] -r
慎用。可以直接crontab -e 进行编辑
```
### 5.载入
```
crontab [-u user] file
将file做为crontab的任务列表文件并载入crontab
如果在命令行中没有指定这个文件，crontab命令将接受标准输入（键盘）上键入的命令，并将它们载入crontab。
```
### 6.Crontab服务
```
service crond start    //启动服务
service crond stop     //关闭服务
service crond restart  //重启服务
service crond reload   //重新载入配置
service crond status   //查看服务状态
```
### 7.目录
```
/etc/cron.d/
这个目录用来存放任何要执行的crontab文件或脚本。
```
### 8.注意事项
```
1)脚本中涉及文件路径时写全局路径
2)脚本执行要用到java或其他环境变量时，通过source命令引入环境变量
3)新创建的cron job，不会马上执行，至少要过2分钟才执行。如果重启cron则马上执行。
```

