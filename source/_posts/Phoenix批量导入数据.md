---
title: Phoenix批量导入数据
date: 2021-11-02 12:41:53
tags:
- Phoenix
categories:
- Phoenix
---

# Phoenix批量导入数据

Phoenix有两种方式供批量写数据。一种是单线程psql方式，另一种是mr分布式。单线程适合一次写入十来兆的文件，mr方式更加适合写大批量数据。

## 单线程psql模式

* 1、创建phoenix 表

```sql
CREATE TABLE example (
    my_pk bigint not null,
    m.first_name varchar(50),
    m.last_name varchar(50) 
    CONSTRAINT pk PRIMARY KEY (my_pk)
);
```

* 2、创建二级索引

```sql
 create index example_first_name_index on example(m.first_name);
```

* 3、创建data.csv文件，并上传一份至hdfs中

```sql
12345,Joddhn,Dois
67890,Maryddd,Poppssssins
123452,Joddhn,Dois
678902,Maryddd,Poppssssins2
```

* 4、上传数据

```shell
/usr/hdp/2.5.3.0-37/phoenix/bin/psql.py -t EXAMPLE hdp14:2181 /root/Templates/data.csv
```

批量导入数据正常以及批量导入数据是会自动更新索引表的



## mr批量写数据方式

```shell
HADOOP_CLASSPATH=/usr/hdp/3.1.4.0-315/hbase/lib/hbase-protocol.jar:/usr/hdp/3.1.4.0-315/hbase/conf hadoop jar /usr/hdp/3.1.4.0-315/phoenix/phoenix-5.0.0.3.1.4.0-315-client.jar org.apache.phoenix.mapreduce.CsvBulkLoadTool --table PEMS.TB_TAGDAYVALUE  --input /user/root/test.csv
```

>  /tmp/YCB/data.csv为hdfs上对应的文件路径
>
> 该命令可随意挑集群中一台机器，当然也可以通过指定具体机器执行，如添加-z 机器:端口便可

* 注：官网指出如果为phoenix4.0及以上要用如下方式

```shell
HADOOP_CLASSPATH=/path/to/hbase-protocol.jar:/path/to/hbase/conf hadoop jar phoenix-<version>-client.jar org.apache.phoenix.mapreduce.CsvBulkLoadTool --table EXAMPLE --input /data/example.csv
```

## 总结

1、速度：
		CSV data can be bulk loaded with built in utility named psql. Typical upsert rates are 20K - 50K rows per second (depends on how wide are the rows).
解释上述意思是：通过bulk loaded 的方式批量写数据速度大概能达到20K-50K每秒。官网出处：https://phoenix.apache.org/faq.html

2、 通过测试确认批量导入会自动更新phoenix二级索引（这个结果不受是否先有hbase表的影响）。

3、导入文件编码默认是utf-8格式。

4、mr方式支持的参数还有其他的具体如下：

5、mr方式导入数据默认会自动更新指定表的所有索引表，如果只需要更新指定的索引表可用-it 参数指定更新的索引表。对文件默认支持的分割符是逗号，参数为-d.

6、如果是想通过代码方式批量导入数据，可以通过代码先将数据写到hdfs中，将mr批量导入方式写到shell脚本中，再通过代码调用shell脚本（写批量执行命令的脚本）执行便可
