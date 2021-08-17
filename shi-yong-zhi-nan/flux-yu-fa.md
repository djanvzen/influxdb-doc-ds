# Flux语法

### 启用Flux查询

修改 `influxdb.conf`的`[http]`段落，将`flux-enabled` 选项p设置为 `true` 。并重启 InfluxDB 服务

```text
# ...

[http]

  # ...

  flux-enabled = true

  # ...
```

### 通过 influx CLI 使用Flux查询InfluxDB

在1.8版本下的 InfluxDB中，可以使用 `influx`  命令加参数 `-type=flux` 启用 Flux 查询。

```text
influx -type=flux
```

### 关键概念

#### Buckets <a id="buckets"></a>

Flux 为 InfluxDB 引入了一种新的数据存储概念 -- `Bucket`。一个`Bucket`是存储数据具有保留策略具名的位置。它类似于 InfluxDB v1.x `DataBase`，但它是数据库和保留策略的组合。使用多个保留策略时，每个数据保留策略都是一个独立的`Bucket`。

Flux 的`from()`函数，为`bucket`定义了一个 InfluxDB 数据源。将 Flux 与 InfluxDB v1.x 一起使用时，请使用数据库名称和保留策略组合为单个`bucket`命名：

**InfluxDB v1.x** `bucket`**命名约定**

```text
// Pattern
from(bucket:"<database>/<retention-policy>")

// Example
from(bucket:"telegraf/autogen")
```

#### 管道转发操作符 <a id="pipe-forward-operator"></a>

Flux`|>`广泛使用管道转发运算符 \( \) 将操作链接在一起。在每个函数或操作之后，Flux 返回一个包含数据的表或表的集合。管道转发运算符将这些表通过管道输送到下一个函数或操作中，在那里它们将被进一步处理或操作。

#### 表 <a id="tables"></a>

Flux 构造表格中的所有数据。当数据从数据源流式传输时，Flux 将其格式化为带注释的逗号分隔值 \(CSV\)，表示表格。然后函数操作或处理它们并输出新表。这使得将函数链接在一起以构建复杂查询变得容易。

### 使用 Flux 查询

每个 Flux 查询都需要以下3步：

#### 1.定义数据源

Flux 的`from()`函数定义了一个 InfluxDB 数据源。它需要一个`bucket`参数。对于此示例，请使用`telegraf/autogen`TICK 堆栈提供的默认数据库和保留策略的组合。

```text
from(bucket:"telegraf/autogen")
```

#### 2.指定时间范围

Flux 在查询时间序列数据时需要一个时间范围。无限制的查询非常耗费资源，作为一种保护措施，Flux 需要指定时间范围的情况下才会查询数据库。

使用管道转发运算符 \( `|>`\) 将数据从数据源通过管道传输到`range()` 函数，该函数指定查询的时间范围。它接受两个属性：`start`和`stop`。范围可以使用到当前时间的**相对**时间，也可以**绝对**使用时间戳。

**相对时间范围**

```text
// Relative time range with start only. Stop defaults to now.
from(bucket:"telegraf/autogen")
  |> range(start: -1h)

// Relative time range with start and stop
from(bucket:"telegraf/autogen")
  |> range(start: -1h, stop: -10m)
```

{% hint style="info" %}
相对时间范围指的是相对于当前时间
{% endhint %}

**绝对时间范围**

```text
from(bucket:"telegraf/autogen")
  |> range(start: 2018-11-05T23:30:00Z, stop: 2018-11-06T00:00:00Z)
```

**使用以下内容：**

对于本指南，使用相对时间范围 ,`-15m`将查询结果限制为过去 15 分钟的数据：

```text
from(bucket:"telegraf/autogen")
  |> range(start: -15m)
```

#### 3.过滤数据

将范围数据传递到`filter()`函数中，可以根据数据属性或列缩小结果范围。`filter()`函数有一个参数 ，`fn`它需要一个匿名函数，该函数具有基于列或属性过滤数据的逻辑。

Flux 的匿名函数语法与 Javascript 非常相似。记录或行`filter()`作为记录 \( `r`\)传递到函数中。匿名函数获取记录并对其进行评估以查看它是否与定义的过滤器匹配。使用`AND`关系运算符链接多个过滤器。

```text
// Pattern
(r) => (r.recordProperty comparisonOperator comparisonExpression)

// Example with single filter
(r) => (r._measurement == "cpu")

// Example with multiple filters
(r) => (r._measurement == "cpu") and (r._field != "usage_system" )
```

**使用以下内容：**

对于此示例，按`cpu`measurement、`usage_system`字段和`cpu-total`tag过滤：

```text
from(bucket:"telegraf/autogen")
  |> range(start: -15m)
  |> filter(fn: (r) =>
    r._measurement == "cpu" and
    r._field == "usage_system" and
    r.cpu == "cpu-total"
  )
```



