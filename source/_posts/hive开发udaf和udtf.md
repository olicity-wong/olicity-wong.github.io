---
title: hive开发udaf和udtf
tags:
  - hive
  - udf
abbrlink: 37d63f90
date: 2019-03-27 10:58:54
---
&emsp;之前开发过udf，但是udf只能处理一对一的情况，也就是一个输入对应一个输出。而日常开发中却会遇到多种情况，普通的udf不能满足，这时候就需要引入udtf和udaf了。
<!--more-->
## 一、简介
### 1.1 UDAF
&emsp;UDAF(User- Defined Aggregation Funcation)用户定义聚合函数，可对多行数据产生作用；等同与SQL中常用的SUM()，AVG()，也是聚合函数；  
简单说就是多行输入一行输出。
### 1.2 UDTF
&emsp;UDTF(User-Defined Table-Generating Functions)用户定义表生成函数，用来解决输入一行输出多行；  
简单说就是一行输入多行输出。
## 二、UDAF编写例子
### 2.1 说明
```
1.引入如下两下类
    import org.apache.hadoop.hive.ql.exec.UDAF  
    import org.apache.hadoop.hive.ql.exec.UDAFEvaluator  
2.函数类需要继承UDAF类，计算类Evaluator实现UDAFEvaluator接口
3.Evaluator需要实现UDAFEvaluator的init、iterate、terminatePartial、merge、terminate这几个函数。
    a）init函数实现接口UDAFEvaluator的init函数。
    b）iterate接收传入的参数，并进行内部的迭代。其返回类型为boolean。
    c）terminatePartial无参数，其为iterate函数遍历结束后，返回遍历得到的数据，terminatePartial类似于 hadoop的Combiner。
    d）merge接收terminatePartial的返回结果，进行数据merge操作，其返回类型为boolean。
    e）terminate返回最终的聚集函数结果。
```
### 2.2 实例
```
package hive.udaf;  
  
import org.apache.hadoop.hive.ql.exec.UDAF;  
import org.apache.hadoop.hive.ql.exec.UDAFEvaluator;  
//create temporary function udaf_avg 'hive.udaf.Avg'
public class Avg extends UDAF {  
    public static class AvgState {  
        private long mCount;  
        private double mSum;  
  
    }  
  
    public static class AvgEvaluator implements UDAFEvaluator {  
        AvgState state;  
  
        public AvgEvaluator() {  
            super();  
            state = new AvgState();  
            init();  
        }  
  
        /** 
         * init函数类似于构造函数，用于UDAF的初始化 
         */  
        public void init() {  
            state.mSum = 0;  
            state.mCount = 0;  
        }  
  
        /** 
         * iterate接收传入的参数，并进行内部的轮转。其返回类型为boolean * * @param o * @return 
         */  
  
        public boolean iterate(Double o) {  
            if (o != null) {  
                state.mSum += o;  
                state.mCount++;  
            }  
            return true;  
        }  
  
        /** 
         * terminatePartial无参数，其为iterate函数遍历结束后，返回轮转数据， * terminatePartial类似于hadoop的Combiner * * @return 
         */  
  
        public AvgState terminatePartial() {  
            // combiner  
            return state.mCount == 0 ? null : state;  
        }  
  
        /** 
         * merge接收terminatePartial的返回结果，进行数据merge操作，其返回类型为boolean * * @param o * @return 
         */  
  
        public boolean merge(AvgState avgState) {  
            if (avgState != null) {  
                state.mCount += avgState.mCount;  
                state.mSum += avgState.mSum;  
            }  
            return true;  
        }  
  
        /** 
         * terminate返回最终的聚集函数结果 * * @return 
         */  
        public Double terminate() {  
            return state.mCount == 0 ? null : Double.valueOf(state.mSum / state.mCount);  
        }  
    }  
}
```
## 三、UDTF编写例子
### 3.1 说明
```
1.引入import org.apache.hadoop.hive.ql.udf.generic.GenericUDTF;
2.继承GenericUDTF类
3.重写initialize（返回输出行信息：列个数，类型）, process, close三方法
```
### 3.2 实例
```
//create temporary function DaylistUDTF as 'com.ai.hive.udf.uc.DaylistUDTF';
public class DaylistUDTF extends GenericUDTF{

    @Override
    public void close() throws HiveException {
        // TODO Auto-generated method stub

    }
    @Override
    public void process(Object[] args) throws HiveException {
        int argsLength = args.length;
        int resultLength = argsLength+1;
        String daylist = args[4].toString();
        String dayId = args[11].toString();
        String[] result = new String[resultLength];
        for(int j = 0; j < argsLength; j++){
            result[j] = args[j].toString();
        }
        try{
            for (int i = daylist.length() - 1; i >= 0; i--) {
                char c = daylist.charAt(i);
                if (c == 49) {
                    int diff = 0-(daylist.length() - i);
                    SimpleDateFormat sdf=new SimpleDateFormat("yyyyMMdd");
                    Date dt=sdf.parse(dayId);
                    Calendar rightNow = Calendar.getInstance();
                    rightNow.setTime(dt);
                    rightNow.add(Calendar.DAY_OF_YEAR,diff);
                    Date dt1=rightNow.getTime();
                    String reStr = sdf.format(dt1);
                    result[resultLength-1] = reStr;
                    forward(result);
                }
            }
        }catch (Exception e){
            System.out.println("error:"+e.toString());
        }
    }
    @Override
    public StructObjectInspector initialize(ObjectInspector[] args)
            throws UDFArgumentException {
        ArrayList<String> fieldNames = new ArrayList<String>();
        ArrayList<ObjectInspector> fieldOIs = new ArrayList<ObjectInspector>();
        for (int i = 1; i <= 14; i++)
        {
            fieldNames.add("col" + i);
            fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
        }
        return ObjectInspectorFactory.getStandardStructObjectInspector(fieldNames,fieldOIs);
    }
}
```
## 四、感谢大佬
&emsp;[Hive 自定义函数 UDF UDAF UDTF](https://www.cnblogs.com/mzzcy/p/7119423.html)  
&emsp;[hive udaf开发入门和运行过程详解](http://www.cnblogs.com/ggjucheng/archive/2013/02/01/2888051.html)  
&emsp;[hive中udtf编写及使用](https://blog.csdn.net/u012485099/article/details/80790908)