---
title: hive开发UDF及使用
tags:
  - hive
  - udf
abbrlink: 96910ad9
date: 2018-10-26 11:03:04
---
&emsp;最近有个数据挖掘的需求，要求统计所给经纬度附近n公里某些事物的数量。涉及到地球两点间的距离计算，需要写UDF进行计算。
<!--more-->
## 一、UDF编写
&emsp;根据经纬度计算两点间的距离，网上有很多计算方法，试了几个，发现这篇[博客](https://blog.csdn.net/u011001084/article/details/52980834)的方法计算的精度差比较小，他的分析方法也很详细，最终采用此方法。
```
import com.ai.hive.udf.topdomain.StringUtil;
import org.apache.hadoop.hive.ql.exec.UDF;
import org.apache.hadoop.io.Text;
import org.apache.log4j.Logger;

/**
 * 功能：根据两地经纬度计算两点之间的距离
 * create temporary function LocDistanceCalUDF as 'com.ai.hive.udf.util.LocDistanceCalUDF';
 * @author olicity
 */
public class LocDistanceCalUDF extends UDF{
    private static Logger log = Logger.getLogger(LocDistanceCalUDF.class);

    private Text nullText = new Text("");
    /**
    *根据经纬度计算地球两点间的距离
    */
    private static double distanceCal(double lng1, double lat1,double lng2,double lat2){
        double dx = lng1 - lng2;// 经度差值
        double dy= lat1 - lat2;// 纬度差值
        double b = (lat1 + lat2) / 2.0;// 平均纬度
        double Lx = Math.toRadians(dx)*6367000.0*Math.cos(Math.toRadians(b));// 东西距离
        double Ly = 6367000.0*Math.toRadians(dy);// 南北距离
        return Math.sqrt(Lx*Lx+Ly*Ly);// 用平面的矩形对角距离公式计算总距离(米)
    }
    /**
    *重写evaluate方法
    */
    public Text evaluate(Text longitudeText1, Text latitudeText1,Text longitudeText2, Text latitudeText2){
        try{
            if(longitudeText1==null || latitudeText1==null || longitudeText2==null || latitudeText2==null){
                return nullText;
            }
            if(StringUtil.isEmpty(longitudeText1.toString()) || StringUtil.isEmpty(latitudeText1.toString()) || StringUtil.isEmpty(longitudeText2.toString()) || StringUtil.isEmpty(latitudeText2.toString())){
                return nullText;
            }
            double lng1 = Double.valueOf(longitudeText1.toString());
            double lat1 = Double.valueOf(latitudeText1.toString());
            double lng2 = Double.valueOf(longitudeText2.toString());
            double lat2 = Double.valueOf(latitudeText2.toString());

            double dis = distanceCal(lng1,lat1,lng2,lat2);
            return new Text(String.valueOf(dis));
        }catch (Exception e){
            return nullText;
        }

    }
    /**
    *重写evaluate方法
    */
    public Text evaluate(Text locationA,Text locationB){
        try{
            if (locationA==null||locationB==null){
                return nullText;
            }
            if(StringUtil.isEmpty(locationA.toString()) || StringUtil.isEmpty(locationB.toString())){
                return nullText;
            }
            String locationA2String  = locationA.toString();
            String locationB2String  = locationB.toString();
            double lng1 = Double.valueOf(locationA2String.split(",")[0]);
            double lat1 = Double.valueOf(locationA2String.split(",")[1]);
            double lng2 = Double.valueOf(locationB2String.split(",")[0]);
            double lat2 = Double.valueOf(locationB2String.split(",")[1]);

            double dis = distanceCal(lng1,lat1,lng2,lat2);
            return new Text(String.valueOf(dis));
        }catch(Exception e){
            return nullText;
        }
    }

}
```
&emsp;UDF类要继承org.apache.hadoop.hive.ql.exec.UDF类，类中要实现evaluate。 当我们在hive中使用自定义的UDF的时候，hive会调用类中的evaluate方法来实现特定的功能。  

## 二、UDF导入
### 1.jar包上传
&emsp;右键类名，Copy reference，复制此类全路径得到：com.ai.hive.udf.util.LocDistanceCalUDF。将写完的类打成jar包上传到服务器。路径如：/user/olicity/hive/UDF
### 2.jar包引入classpath变量中
进入hive，引入jar包，执行命令
```
add jar /user/olicity/hive/UDF/udf.jar;
```
查看导入的jar包
```
list jars;
```
### 3.创建函数
创建一个名为LocDistanceCalUDF的临时函数，关联该jar包
```
create temporary function LocDistanceCalUDF as 'com.ai.hive.udf.util.LocDistanceCalUDF';
```
查看创建的函数
```
show functions;
```
### 4.注意
&emsp;上述方法仅限于当前会话生效，如需要添加一个永久的函数对应的永久的路径，则
```
create function locUDF.LocDistanceCalUDF 
  as 'com.ai.hive.udf.util.LocDistanceCalUDF' 
  using jar 'hdfs://hdfs路径/udf.jar';
use LocDistanceCalUDF;
```
需要将jar包放到hdfs上，然后创建函数关联路径即可。  
另外还看到过另一种方法，配置hive-site.xml文件中的hive.aux.jars.path
```
配置参考如下：
   <property>
       <name>hive.aux.jars.path</name>
       <value>file:///home/hdfs/fangjs/DefTextInputFormat.jar,file:///jarpath/test.jar</value>
   </property>
```

## 三、UDF使用
&emsp;准备工作已经就绪，可以开始查表了。emmmmm，就简单的俩表查吧，表结构和表数据就不展示了，示例表也就不建了，所给经纬度表叫A表，需要查询的表叫B表，临时中间表叫c表，经纬度的表中字段定义是loc，距离就算2公里吧。
```
create table C
as
select B.* from B join A where (LocDistanceCalUDF(A.loc,B.loc)<=2000);
```
OK.
## 四、总结
&emsp;关于sql语句还是需要再多加练习，尤其是多表联查。Hadoop之路任重而道远。
