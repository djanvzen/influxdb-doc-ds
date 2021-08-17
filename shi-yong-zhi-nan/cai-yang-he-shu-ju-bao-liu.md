# 采样和数据保留

InfluxDB每秒可以处理数十万的数据点。如果要长时间地存储大量的数据，对于存储会是很大的压力。一个很自然的方式就是对数据进行采样，对于高精度的裸数据存储较短的时间，而对于低精度的的数据可以保存得久一些甚至永久保存。

InfluxDB提供了两个特性，即连续查询\(Continuous Queries简称CQ\)和保留策略\(Retention Policies简称RP\)，分别用来处理数据采样和管理老数据的。

### 定义 <a id="&#x5B9A;&#x4E49;"></a>

**Continuous Query \(CQ\)** 是指InfluxDB提供了自动聚合数据，并将聚合数据存储至measurement的方法 ，CQ需要在`SELECT`语句中使用一个函数，并且一定包括一个`GROUP BY time()`语句。

**Retention Policy \(RP\)**表示数据保留策略，策略包含数据保留时长，备份个数等信息。InfluxDB为每个database默认创建了一个默认的RP，名称为autogen，默认数据保留时间为永久。DB会比较服务器本地的时间戳和请求数据里的时间戳，并删除比你在RP里面用`DURATION`设置的更老的数据Influx。

### 使用CQ与RP的目的 <a id="&#x4F7F;&#x7528;CQ&#x4E0E;RP&#x7684;&#x76EE;&#x7684;"></a>

1. 利用CQ可以达到数据降准的目的，即将细粒度的数据转换为粗粒度的数据，例如1m维度的数据可以聚合为5m数据
2. 不同粒度的数据可以设置不同的RP，节省存储空间。例如1m维度的数据可以保存7d，而5m维度的数据则可以保存30d，细粒度的数据主要用于及时排查问题，粗粒度的数据在于查看变化趋势。

### CQ介绍

#### 基础用法 <a id="&#x57FA;&#x7840;&#x7528;&#x6CD5;"></a>

```text
CREATE CONTINUOUS QUERY <cq_name> ON <database_name>
BEGIN
  <cq_query>
END            
```

其中cq\_query表示聚合语句，

```text
SELECT <function[s]> INTO <destination_measurement> FROM <measurement> [WHERE <stuff>] GROUP BY time(<interval>)[,<tag_key[s]>]
```

cq\_query的where条件中不需要设置时间区间，inflxuDB会自动生成时间区间

InfluxDB根据cq\_query中的Group By time\(interval\)的interval，每隔interval执行一次cq\_query，而查询的时间区间也为interval。例如当前时间为17：00，interval为1h，则cq\_query会查询16:00至16：59分的数据。

生成一条CQ如下，则每隔1h执行一次query

```text
CREATE CONTINUOUS QUERY "cq_basic" ON "iot"
BEGIN
  SELECT mean("monitor_val") INTO "avg_tem" FROM "aircondition" GROUP BY time(1h)
END
```

在8：00，时间区间为\[7:00,8:00\)

在9：00，时间区间为\[8:00,9:00\)

根据CQ的特性，进行可以数据降准，例如将维度为1m的数据聚合为5m,measurement\_1m存储维度为1m的数据，cq每隔5m对数据进行聚合，并将聚合的数据存储到rp\_5m.measurement\_5m中，此时维度变成了5m

```text
CREATE CONTINUOUS QUERY "cq_basic" ON "iot"
BEGIN
  SELECT mean(*) INTO "rp_5m"."measurement_5m" 
  FROM "rp_1m"."measurement_1m"
  GROUP BY time(5m),*
END    
```

**时间偏移**

```text
CREATE CONTINUOUS QUERY "cq_basic_offset" ON "iot"
BEGIN
  SELECT mean("monitor_val") INTO "avg_tem" FROM "aircondition" GROUP BY time(1h,15m)
END
```

group by time\(1h\) 与 group by time\(1h,15m\)的区别

group by time\(1h\)的执行时间点和时间区间

| time | range |
| :--- | :--- |
| 8:00 | \[7:00,8:00\) |
| 9:00 | \[8:00,9:00\) |

group by time\(1h, 15m\)的执行时间点和时间区间

| time | range |
| :--- | :--- |
| 8:15 | \[7:15,8:15\) |
| 9:15 | \[8:15,9:15\) |

#### 高级用法 <a id="&#x9AD8;&#x7EA7;&#x7528;&#x6CD5;"></a>

```text
CREATE CONTINUOUS QUERY <cq_name> ON <database_name>
RESAMPLE EVERY <interval> FOR <interval>
BEGIN
  <cq_query>
END    
```

其中EVERY表示执行间隔，即每隔多久执行一次，FOR表示时间区间，即每次执行查询多久的数据，区间为\[now - for\_interval, now\)

**EVERY=30m,GROUP BY time\(1h\)**

```text
CREATE CONTINUOUS QUERY "cq_advanced_every" ON "iot"
RESAMPLE EVERY 30m
BEGIN
  SELECT mean("monitor_val") INTO "avg_tem" FROM "aircondition" GROUP BY time(1h)
END
```

| time | range | 返回time |
| :--- | :--- | :--- |
| 8:00 | \[7:00,8:00\) | 7:00 |
| 8:30 | \[7:00,8:00\) | 7:00 |
| 9:00 | \[8:00,9:00\) | 8:00 |

可以看到区间\[7:00,8:00\)的数据被查询了两次，后一次的数据会覆盖前一次的查询

**FOR=1h,GROUP BY time\(30m\)**

```text
CREATE CONTINUOUS QUERY "cq_advanced_for" ON "iot"
RESAMPLE FOR 1h
BEGIN
  SELECT mean("monitor_val") INTO "avg_tem" FROM "aircondition" GROUP BY time(30m)
END
```

| time | range | 返回time |
| :--- | :--- | :--- |
| 8:00 | \[7:00,8:00\) | 7:00 7:30 |
| 8:30 | \[7:30,8:30\) | 7:30 8:00 |
| 9:00 | \[8:00,9:00\) | 8:00 8:30 |

每次查询都会返回两条数据，每个时间点的数据都被计算了两次，这样在一定程度可以避免数据延迟而导致CQ数据丢失的情况。

**EVERY=1h，FOR=90m，GROUP BY time\(30m\)**

```text
CREATE CONTINUOUS QUERY "cq_advanced_every_for" ON "iot"
RESAMPLE EVERY 1h FOR 90m
BEGIN
  SELECT mean("monitor_val") INTO "avg_tem" FROM "aircondition" GROUP BY time(30m)
END
```

| time | range | 返回time |
| :--- | :--- | :--- |
| 8:00 | \[6:30,8:00\) | 6:30 7:00 7:30 |
| 9:00 | \[7:30,9:00\) | 7:30 8:00 8:30 |

FOR interval必须大于GROUP by time\(interval\)以及EVERY interval，否则InfluxDB会返回如下错误，因为这样会造成数据丢失

```text
error parsing query: FOR duration must be >= GROUP BY time duration: must be a minimum of <minimum-allowable-interval> got <user-specified-interval>
```

#### CQ管理 <a id="CQ&#x7BA1;&#x7406;"></a>

**查看CQ**

```text
> SHOW CONTINUOUS QUERIES
```

**删除CQ**

```text
> DROP CONTINUOUS QUERY <cq_name> ON <database_name>
```

**修改CQ**

CQ不能被修改，如果需要需改，只能先删除CQ，在重新创建CQ

### RP介绍

#### 查看RP <a id="&#x65B0;&#x5EFA;RP"></a>

可以直接在指定的数据库中直接查看当前数据库正在使用的RP。

```text
> show retention policies
name    duration   shardGroupDuration replicaN default
----    --------   ------------------ -------- -------
autogen 0s         168h0m0s           1        false
iotrule 48000h0m0s 168h0m0s           1        true

```

#### 新建RP <a id="&#x65B0;&#x5EFA;RP"></a>

新建RP需要使用如下的语法

```text
> CREATE RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> [SHARD DURATION <duration>] [DEFAULT]
```

| name | example |
| :--- | :--- |
| duration | 表示数据保留时间，最长为INF，最短为1h，0表示不限制 |
| replication | 备份数，在集群模式中可用 |
| shard duration | 一个分片包含的时间，InfluxDB会根据duration设置默认的shard duration |
| default | 表示该RP为默认的RP |

新建一个RP，保留时间为1d

```text
> CREATE RETENTION POLICY "iotrule " ON "iot" DURATION 1d REPLICATION 1    
```

#### 修改RP <a id="&#x4FEE;&#x6539;RP"></a>

使用如下语法修改RP

```text
> ALTER RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> SHARD DURATION <duration> DEFAULT
```

#### 删除RP <a id="&#x5220;&#x9664;RP"></a>

使用如下语法删除RP

```text
> DROP RETENTION POLICY <retention_policy_name> ON <database_name>
```

####  <a id="RP&#x4F7F;&#x7528;"></a>

