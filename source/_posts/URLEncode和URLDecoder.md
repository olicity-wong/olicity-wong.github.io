---
title: URLEncode和URLDecoder
tags:
  - java
abbrlink: 686dbd19
date: 2018-10-17 16:22:49
---
## 一、背景
&emsp;使用http请求的在服务之间传递消息时，会出现字符串乱码现象。  
&emsp;使用POST方法提交时，会对其中的有些字符进行编码,数据内容的类型是 application/x-www-form-urlencoded
<!--more-->
```
1.字符"a"-"z"，"A"-"Z"，"0"-"9"，"."，"-"，"*"，和"_" 都不会被编码;
2.将空格转换为加号 (+) ;
3.将非文本内容转换成"%xy"的形式,xy是两位16进制的数值;
4.在每个 name=value 对之间放置 & 符号。
```
&emsp;URLDecoder 和 URLEncoder 用于完成普通字符串 和 application/x-www-form-urlencoded MIME 字符串之间的相互转换。
## 二、使用
```
正常的字符串：破晓
URL中的字符串：%e7%a0%b4%e6%99%93
```
### 1.URLEncode
&emsp;URLEncode是对URL中的特殊字符部分进行编码
```
String url = "破晓";
try{
    url = URLEncoder.encode(url,"utf-8");
    System.out.println("-----"+url);
}catch (Exception e){
    System.out.println("---");
}
``` 
### 2.URLDecode
&emsp;URLEncode是对URL中的特殊字符部分进行解码
```
String url = "%e7%a0%b4%e6%99%93";
try{
    url = URLDecoder.decode(url);
    System.out.println("-----"+url);
}catch (Exception e){
    System.out.println("---");
}
```
## 三、应用
&emsp;上次发现个问题，经过我们使用BASE64加密，通过http请求后，发现原来的“+”全部变成了“ ”。最后发现问题出在这里。  
&emsp;上述方法即可解决http请求后出现乱码现象。