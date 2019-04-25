## 问题描述
- sqoop抽取mysql业务数据导入到对应hive表，时间戳字段比mysql中多处一个.0字段

## 分析过程
```
//测试mysql中表结构时间字段类型如下：
//mysql中主要有datatime和timestamp两种时间类型

CREATE TABLE `test_all_time_type` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
   `create_date` date,
   `create_time` time,
   `create_datetime` datetime ,
  `create_timestamp` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3649 DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='测试各种数据库时间类型';

```
- 1. 业务mysql中时间类型主要有datetime和timestamp
- 2. 自测java程序获取到mysql中数据，java中bean对象，只有设置为java.sql.Timestamp类型时，在java中输出可以看到确实带有.0后缀。据说这个Timestamp精度导致的。
```
//测试使用java
package com.wonhigh.module;

import java.sql.Timestamp;
import java.util.Date;


public class TestAllTimeType {
    private Integer id;

    private Date createDate;

    private Date createTime;

    private Date createDatetime;

    private Timestamp createTimestamp;
    private java.sql.Date h;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public Date getCreateDate() {
        return createDate;
    }

    public void setCreateDate(Date createDate) {
        this.createDate = createDate;
    }

    public Date getCreateTime() {
        return createTime;
    }

    public void setCreateTime(Date createTime) {
        this.createTime = createTime;
    }

    public Date getCreateDatetime() {
        return createDatetime;
    }

    public void setCreateDatetime(Date createDatetime) {
        this.createDatetime = createDatetime;
    }

    public Timestamp getCreateTimestamp() {
        return createTimestamp;
    }

    public void setCreateTimestamp(Timestamp createTimestamp) {
        this.createTimestamp = createTimestamp;
    }

    @Override
    public String toString() {
        return "TestAllTimeType{" +
                "id=" + id +
                ", createDate=" + createDate +
                ", createTime=" + createTime +
                ", createDatetime=" + createDatetime +
                ", createTimestamp=" + createTimestamp +
                '}';
    }
}
```
- 3. 第一次尝试，直接使用sqoop命令将数据导出到hdfs文件上。观察数据情况：
```
sqoop import -D mapred.job.name=testImport \
--connect jdbc:mysql://172.17.209.6:3306/test \
--username root \
--password root \
--table test_all_time_type \
--fields-terminated-by "\t" \
--lines-terminated-by "\n" \
--target-dir /tmp/hive/testTime \
-m 1


查看目录： hdfs dfs -ls /tmp/hive/testTime 
查看具体文件数据：hdfs dfs -cat /tmp/hive/testTime/xxxx
结果发现文件中的时间戳是存在.0结尾数据

当时认为是sqoop在自动生成java的bean对象时候使用的类型不对导致。如果可以通过java去控制也可以解决此问题，但是设计到改源码，难度大
```
- 4. 第二次尝试直接将数据导入到hive，模拟业务问题
```
sqoop import \
--connect jdbc:mysql://172.17.209.6:3306/test \
--username root \
--password root \
--table test_all_time_type \
--fields-terminated-by "\t" \
--lines-terminated-by "\n" \
--target-dir /tmp/hive/testTime \
--hive-import \
--hive-overwrite \
--create-hive-table \
--hive-table  test_all_time_type \
--delete-target-dir \
-m 1 

结果：
     hive :登录hive
     use default :使用default数据库
     select * from test_all_time_type;
     果然在hive中间的时间戳多了一个.0的后缀，复现了业务问题
```
- 6. 尝试使用sqoop中的字段类型转换参数。（map-column-java ：可以转换java的bean对象中的字段类型）
```
sqoop import \
--connect jdbc:mysql://172.17.209.6:3306/test \
--username root \
--password root \
--table test_all_time_type \
--map-column-java create_datetime=java.sql.Timestamp,create_timestamp=java.sql.Timestamp \
--fields-terminated-by "\t" \
--lines-terminated-by "\n" \
--target-dir /tmp/hive/testTime \
--hive-import \
--hive-table  test_all_time_type \
-m 1 

结果：
    其实根据之前在java中的测试可以推测，此转化最后肯定会有一个.0的后缀结尾。
    
    结果在hive中是跟预测一样的，时间戳还是存在.0后缀，最后还是通过网上查找资料：
    发现一个博客，还是有相同类型问题：
    原文：https://www.cnblogs.com/wrencai/p/3935877.html
    
```
- 7. 最后尝试加入两个参数控制（--map-column-hive）和（--map-column-java）同时配合使用，结果问题完美解决
```
sqoop import \
--connect jdbc:mysql://172.17.209.6:3306/test \
--username root \
--password root \
--table test_all_time_type \
--map-column-hive create_datetime=TIMESTAMP,create_timestamp=TIMESTAMP \
--map-column-java create_datetime=java.sql.Timestamp,create_timestamp=java.sql.Timestamp \
--fields-terminated-by "\t" \
--lines-terminated-by "\n" \
--target-dir /tmp/hive/testTime \
--hive-import \
--hive-table  test_all_time_type \
-m 1 

结果：
    在hive中查询，时间戳已经没有了.0后缀，完美解决这个sqoop同步mysql数据到hive的时间戳问题
```

## 注意
```
解决办法： 
在import 时使用 –map-column-java 和 –map-column-hive 两个参数， 多个字段用逗号隔开

 --map-column-java  create_time=java.sql.Timestamp, update_time=java.sql.Timestamp 
 --map-column-hive  create_time=TIMESTAMP, update_time=TIMESTAMP 

在export 时 使用

--map-column-java create_time=java.sql.Timestamp,update_time=java.sql.Timestamp
--------------------- 
原文：https://blog.csdn.net/jobschen/article/details/810
```
