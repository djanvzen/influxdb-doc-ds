# 入门



### 连接数据库 <a id="&#x521B;&#x5EFA;&#x6570;&#x636E;&#x5E93;"></a>

如果你已经在本地安装运行了InfluxDB，你就可以直接使用`influx`命令行，执行`influx`连接到本地的InfluxDB实例上。

```text
$ influx -port 9086
Connected to http://localhost:9086 version 1.8.6
InfluxDB shell version: 1.8.6
> 

```

{% hint style="info" %}
InfluxDB 的 HTTP 接口默认起在8086上，如果修改过则需要使用`-port`参数指定修改后的端口号。
{% endhint %}

这样这个命令行已经准备好接收influx的查询语句了\(简称InfluxQL\)，用`exit`可以退出命令行。

### 基本概念

我们在使用 influxDB 之前需要理解`database`、`measurement`、`tag`、`field`这4个专业术语的概念，下一章节中将详细描述这些术语，在本章节中会使用关系型数据库的概念帮助理解。

首先是`database`的概念，这与关系型数据库中`database`是一致的，而`measurement`类比于SQL里面的`table`，其主键索引总是时间戳，与`table`不同的是，`measurement`无需预先定义表结构。`tag`和`field`是在`table`里的其他列，`tag`是被索引起来的，`field`没有。此外null值不会被存储。

### 创建Database

为了进行数据存储，我们需要通过`CREATE DATABASE <db_name>`创建一个数据库

```text
> CREATE DATABASE iot
>
```

创建之后，可以通过`SHOW DATABASES`查看已创建的数据库

```text
> SHOW DATABASES
name: databases
---------------
name
_internal
iot

>
```

使用`USE <db_name>`，指定需要操作的数据库，如果不指定数据库会报错。

```text
> USE iot
Using database iot
>
```

### 读写数据 <a id="&#x8BFB;&#x5199;&#x6570;&#x636E;"></a>

将数据点写入 InfluxDB，基本语法如下

```text
INSERT INTO <retention policy> measurement,tagKey=tagValue fieldKey=fieldValue,fieldKey1=fieldValue1,fieldKey2=fieldValue2 timestamp
```

{% hint style="warning" %}
需要注意的是插入语句的最后一列是时间戳，在 influxDB 中所有的时间戳都是以纳秒计的。
{% endhint %}

{% hint style="info" %}
若`tag`或`field`有多组键值对，则每个键值对间用`，`进行分割的。
{% endhint %}

示例分析

```text
> INSERT aircondition,poiont_code='320500000003000101048' monitor_val=0.64,last_val=0.12 1609430401000000000
>
```

这样一个measurement为`aircondition`，如果这个measurement是不存在的，则会被自动创建。tag是`ponit_code`，filed是`monitor_value`与`last_valu`。

现在我们可以查出写入的这条数据：

```text
> select * from aircondition
name: aircondition
time                last_val monitor_val poiont_code
----                -------- ----------- -----------
1609430401000000000 0.12     0.64        '320500000003000101048'

>
```

{% hint style="info" %}
我们在写入的时候没有包含时间戳，当没有带时间戳的时候，InfluxDB会自动添加本地的当前时间作为它的时间戳
{% endhint %}

