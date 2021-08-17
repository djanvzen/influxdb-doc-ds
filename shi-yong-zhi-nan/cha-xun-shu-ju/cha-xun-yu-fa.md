# 查询语法

InfluxQL是一种类似SQL的查询语言，用于与InfluxDB中的数据进行交互。 以下部分详细介绍了InfluxQL的`SELECT`语句有关查询语法。

### SELECT语句 <a id="&#x57FA;&#x672C;&#x7684;select&#x8BED;&#x53E5;"></a>

`SELECT`语句从特定的measurement中查询数据。

#### 语法 <a id="&#x8BED;&#x6CD5;"></a>

```text
SELECT <field_key>[,<field_key>,<tag_key>] FROM <measurement_name>[,<measurement_name>]
```

**语法描述**

`SELECT`语句需要一个`SELECT`和`FROM`子句。

**SELECT子句**

`SELECT`支持指定数据的几种格式：

`SLECT *`

返回所有的field和tag。

`SELECT "<field_key>"`

返回特定的field。

`SELECT "<field_key>","<field_key>"`

返回多个field。

`SELECT "<field_key>","<tag_key>"`

返回特定的field和tag，`SELECT`在包括一个tag时，必须至少指定一个field。

`SELECT "<field_key>"::field,"<tag_key>"::tag`

返回特定的field和tag，`::[field | tag]`语法指定标识符的类型。 使用此语法来区分具有相同名称的field key和tag key。

**FROM子句**

`FROM`子句支持几种用于指定measurement的格式：

`FROM <measurement_name>`

从单个measurement返回数据。如果使用CLI需要先用`USE`指定数据库，并且使用的`DEFAULT`存储策略。如果您使用HTTP API,需要用`db`参数来指定数据库，也是使用`DEFAULT`存储策略。

`FROM <measurement_name>,<measurement_name>`

从多个measurement中返回数据。

`FROM <database_name>.<retention_policy_name>.<measurement_name>`

从一个完全指定的measurement中返回数据，这个完全指定是指指定了数据库和存储策略。

`FROM <database_name>..<measurement_name>`

从一个用户指定的数据库中返回存储策略为`DEFAULT`的数据。

**引号**

如果标识符包含除\[A-z，0-9，\_\]之外的字符，如果它们以数字开头，或者如果它们是InfluxQL关键字，那么它们必须用双引号。虽然并不总是需要，我们建议您双引号标识符。

#### WHERE子句

`WHERE`子句用作field，tag和timestamp的过滤。

**语法**

```text
SELECT_clause FROM_clause WHERE <conditional_expression> [(AND|OR) <conditional_expression> [...]]
```

**语法描述**

`WHERE`子句在field，tag和timestamp上支持`conditional_expressions`.

**fields**

```text
field_key <operator> ['string' | boolean | float | integer]
```

`WHERE`子句支持field value是字符串，布尔型，浮点数和整数这些类型。

在`WHERE`子句中单引号来表示字符串字段值。具有无引号字符串字段值或双引号字符串字段值的查询将不会返回任何数据，并且在大多数情况下也不会返回错误。

支持的操作符：

`=` 等于  
`<>` 不等于  
`!=` 不等于  
`>` 大于  
`>=` 大于等于  
`<` 小于  
`<=` 小于等于

**tags**

```text
tag_key <operator> ['tag_value']
```

`WHERE`子句中的用单引号来把tag value引起来。具有未用单引号的tag或双引号的tag查询将不会返回任何数据，并且在大多数情况下不会返回错误。

支持的操作符：

`=` 等于  
`<>` 不等于  
`!=` 不等于

**timestamps**

对于大多数`SELECT`语句，默认时间范围为UTC的`1677-09-21 00：12：43.145224194`到`2262-04-11T23：47：16.854775806Z`。 对于只有`GROUP BY time()`子句的`SELECT`语句，默认时间范围在UTC的`1677-09-21 00：12：43.145224194`和`now()`之间。

### GROUP BY子句 <a id="group-by&#x5B50;&#x53E5;"></a>

`GROUP BY`子句后面可以跟用户指定的tags或者是一个时间间隔。

#### GROUP BY tags <a id="group-by-tags"></a>

`GROUP BY <tag>`后面跟用户指定的tags。

**语法**

```text
SELECT_clause FROM_clause [WHERE_clause] GROUP BY [* | <tag_key>[,<tag_key]]
```

**语法描述**

`GROUP BY *`  
对结果中的所有tag作group by。

`GROUP BY <tag_key>`  
对结果按指定的tag作group by。

`GROUP BY <tag_key>,<tag_key>`  
对结果数据按多个tag作group by，其中tag key的顺序没所谓。

#### GROUP BY 时间间隔 <a id="group-by&#x65F6;&#x95F4;&#x95F4;&#x9694;"></a>

`GROUP BY time()`返回结果按指定的时间间隔group by。

**基本的GROUP BY time\(\)语法**

**语法**

```text
SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(<time_interval>),[tag_key] [fill(<fill_option>)]
```

**基本语法描述**

基本`GROUP BY time()`查询需要`SELECT`子句中的InfluxQL函数和`WHERE`子句中的时间范围。请注意，`GROUP BY`子句必须在`WHERE`子句之后。

`time(time_interval)`  
`GROUP BY time()`语句中的`time_interval`是一个时间duration。决定了InfluxDB按什么时间间隔group by。例如：`time_interval`为`5m`则在`WHERE`子句中指定的时间范围内将查询结果分到五分钟时间组里。

`fill(<fill_option>)`  
`fill（<fill_option>）`是可选的。它会更改不含数据的时间间隔的返回值。

覆盖范围：基本`GROUP BY time()`查询依赖于`time_interval`和InfluxDB的预设时间边界来确定每个时间间隔中包含的原始数据以及查询返回的时间戳。

**高级GROUP BY time\(\)语法**

**语法**

```text
SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(<time_interval>,<offset_interval>),[tag_key] [fill(<fill_option>)]
```

**高级语法描述**

高级`GROUP BY time()`查询需要`SELECT`子句中的InfluxQL函数和`WHERE`子句中的时间范围。 请注意，`GROUP BY`子句必须在`WHERE`子句之后。

`time(time_interval,offset_interval)`

`offset_interval`是一个持续时间。它向前或向后移动InfluxDB的预设时间界限。`offset_interval`可以为正或负。

`fill(<fill_option>)`

`fill(<fill_option>)`是可选的。它会更改不含数据的时间间隔的返回值。

范围

高级`GROUP BY time()`查询依赖于`time_interval`，`offset_interval`和InfluxDB的预设时间边界，以确定每个时间间隔中包含的原始数据以及查询返回的时间戳。

**GROUP BY time\(\)加fill\(\)**

`fill()`更改不含数据的时间间隔的返回值。

**语法**

```text
SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(time_interval,[<offset_interval])[,tag_key] [fill(<fill_option>)]
```

**语法描述**

默认情况下，没有数据的`GROUP BY time()`间隔返回为null作为输出列中的值。`fill()`更改不含数据的时间间隔返回的值。请注意，如果`GROUP(ing)BY`多个对象（例如，tag和时间间隔），那么`fill()`必须位于`GROUP BY`子句的末尾。

fill的参数

* 任一数值：用这个数字返回没有数据点的时间间隔
* linear：返回没有数据的时间间隔的线性插值结果。
* none: 不返回在时间间隔里没有点的数据
* previous：返回时间隔间的前一个间隔的数据

### INTO子句 <a id="into&#x5B50;&#x53E5;"></a>

`INTO`子句将查询的结果写入到用户自定义的measurement中。

#### 语法 <a id="&#x8BED;&#x6CD5;"></a>

```text
SELECT_clause INTO <measurement_name> FROM_clause [WHERE_clause] [GROUP_BY_clause]
```

#### 语法描述 <a id="&#x8BED;&#x6CD5;&#x63CF;&#x8FF0;"></a>

`INTO`支持多种格式的measurement。

`INTO <measurement_name>`

写入到特定measurement中，用CLI时，写入到用`USE`指定的数据库，保留策略为`DEFAULT`，用HTTP API时，写入到`db`参数指定的数据库，保留策略为`DEFAULT`。

`INTO <database_name>.<retention_policy_name>.<measurement_name>`

写入到完整指定的measurement中。

`INTO <database_name>..<measurement_name>`

写入到指定数据库保留策略为`DEFAULT`。

`INTO <database_name>.<retention_policy_name>.:MEASUREMENT FROM /<regular_expression>/`

将数据写入与`FROM`子句中正则表达式匹配的用户指定数据库和保留策略的所有measurement。 `:MEASUREMENT`是对`FROM`子句中匹配的每个measurement的反向引用。

### ORDER BY TIME DESC <a id="order-by-time-desc"></a>

默认情况下，InfluxDB以升序的顺序返回结果; 返回的第一个点具有最早的时间戳，返回的最后一个点具有最新的时间戳。 `ORDER BY time DESC`反转该顺序，使得InfluxDB首先返回具有最新时间戳的点。

**语法**

```text
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] ORDER BY time DESC
```

**语法描述**

如果查询包含`GROUP BY`子句,`ORDER by time DESC`必须出现在`GROUP BY`子句之后。如果查询包含一个`WHERE`子句并没有`GROUP BY`子句，`ORDER by time DESC`必须出现在`WHERE`子句之后。

### LIMIT和SLIMIT子句 <a id="limit&#x548C;slimit&#x5B50;&#x53E5;"></a>

`LIMIT <N>`从指定的measurement中返回前`N`个数据点。

#### 语法 <a id="&#x8BED;&#x6CD5;"></a>

```text
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] LIMIT <N>
```

#### 语法描述 <a id="&#x8BED;&#x6CD5;&#x63CF;&#x8FF0;"></a>

`N`指定从指定measurement返回的点数。如果`N`大于measurement的点总数，InfluxDB返回该measurement中的所有点。请注意，`LIMIT`子句必须以上述语法中列出的顺序显示。

### SLIMIT子句 <a id="slimit&#x5B50;&#x53E5;"></a>

`SLIMIT <N>`返回指定measurement的前个series中的每一个点。

#### 语法 <a id="&#x8BED;&#x6CD5;"></a>

```text
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] GROUP BY *[,time(<time_interval>)] [ORDER_BY_clause] SLIMIT <N>
```

#### 语法描述 <a id="&#x8BED;&#x6CD5;&#x63CF;&#x8FF0;"></a>

`N`表示从指定measurement返回的序列数。如果`N`大于measurement中的series数，InfluxDB将从该measurement中返回所有series。

#### LIMIT和SLIMIT一起使用

`SLIMIT <N>`后面跟着`LIMIT <N>`返回指定measurement的个series中的个数据点。

#### 语法 <a id="&#x8BED;&#x6CD5;"></a>

```text
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] GROUP BY *[,time(<time_interval>)] [ORDER_BY_clause] LIMIT <N1> SLIMIT <N2>
```

#### 语法描述 <a id="&#x8BED;&#x6CD5;&#x63CF;&#x8FF0;"></a>

`N1`指定每次measurement返回的点数。如果`N1`大于measurement的点数，InfluxDB将从该测量中返回所有点。

`N2`指定从指定measurement返回的series数。如果`N2`大于measurement中series联数，InfluxDB将从该measurement中返回所有series。

### OFFSET和SOFFSET子句 <a id="offset&#x548C;soffset&#x5B50;&#x53E5;"></a>

`OFFSET`和`SOFFSET`分页和series返回。

#### OFFSET子句 <a id="offset&#x5B50;&#x53E5;"></a>

`OFFSET <N>`从查询结果中返回分页的N个数据点

**语法**

```text
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] LIMIT_clause OFFSET <N> [SLIMIT_clause]
```

**语法描述**

`N`指定分页数。`OFFSET`子句需要一个`LIMIT`子句。使用没有`LIMIT`子句的`OFFSET`子句可能会导致不一致的查询结果。

#### SOFFSET子句 <a id="soffset&#x5B50;&#x53E5;"></a>

`SOFFSET <N>`从查询结果中返回分页的N个series

**语法**

```text
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] GROUP BY *[,time(time_interval)] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] SLIMIT_clause SOFFSET <N>
```

**语法描述**

`N`指定series的分页数。`SOFFSET`子句需要一个`SLIMIT`子句。使用没有`SLIMIT`子句的`SOFFSET`子句可能会导致不一致的查询结果。

{% hint style="warning" %}
如果SOFFSET指定的大于series的数目，则InfluxDB返回空值。
{% endhint %}

### Time Zone子句 <a id="time-zone&#x5B50;&#x53E5;"></a>

`tz()`子句返回指定时区的UTC偏移量。

#### 语法 <a id="&#x8BED;&#x6CD5;"></a>

```text
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause] tz('<time_zone>')
```

#### 语法描述 <a id="&#x8BED;&#x6CD5;&#x63CF;&#x8FF0;"></a>

默认情况下，InfluxDB以UTC为单位存储并返回时间戳。 `tz()`子句包含UTC偏移量，或UTC夏令时（DST）偏移量到查询返回的时间戳中。 返回的时间戳必须是RFC3339格式，用于UTC偏移量或UTC DST才能显示。`time_zone`参数遵循[Internet Assigned Numbers Authority时区数据库](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List)中的TZ语法，它需要单引号。

### 时间语法 <a id="&#x65F6;&#x95F4;&#x8BED;&#x6CD5;"></a>

对于大多数`SELECT`语句，默认时间范围为UTC的`1677-09-21 00：12：43.145224194`到`2262-04-11T23：47：16.854775806Z`。 对于具有`GROUP BY time()`子句的`SELECT`语句，默认时间范围在UTC的`1677-09-21 00：12：43.145224194`和now\(\)之间。以下部分详细说明了如何在`SELECT`语句的`WHERE`子句中指定替代时间范围。

#### 绝对时间 <a id="&#x7EDD;&#x5BF9;&#x65F6;&#x95F4;"></a>

用时间字符串或是epoch时间来指定绝对时间

**语法**

```text
SELECT_clause FROM_clause WHERE time <operator> ['<rfc3339_date_time_string>' | '<rfc3339_like_date_time_string>' | <epoch_time>] [AND ['<rfc3339_date_time_string>' | '<rfc3339_like_date_time_string>' | <epoch_time>] [...]]
```

**语法描述**

**支持的操作符**

`=` 等于  
`<>` 不等于  
`!=` 不等于  
`>` 大于  
`>=` 大于等于  
`<` 小于  
`<=` 小于等于

最近，InfluxDB不再支持在`WHERE`的绝对时间里面使用`OR`了。

**rfc3399时间字符串**

```text
'YYYY-MM-DDTHH:MM:SS.nnnnnnnnnZ'
```

`.nnnnnnnnn`是可选的，如果没有的话，默认是`.00000000`,rfc3399格式的时间字符串要用单引号引起来。

**epoch\_time**

Epoch时间是1970年1月1日星期四00:00:00（UTC）以来所经过的时间。默认情况下，InfluxDB假定所有epoch时间戳都是纳秒。也可以在epoch时间戳的末尾包括一个表示时间精度的字符，以表示除纳秒以外的精度。

**基本算术**

所有时间戳格式都支持基本算术。用表示时间精度的字符添加（+）或减去（-）一个时间。请注意，InfluxQL需要+或-和表示时间精度的字符之间用空格隔开。

#### 相对时间 <a id="&#x76F8;&#x5BF9;&#x65F6;&#x95F4;"></a>

使用`now()`查询时间戳相对于服务器当前时间戳的的数据。

**语法**

```text
SELECT_clause FROM_clause WHERE time <operator> now() [[ - | + ] <duration_literal>] [(AND|OR) now() [...]]
```

**语法描述**

`now()`是在该服务器上执行查询时服务器的Unix时间。`-`或`+`和时间字符串之间需要空格。

**支持的操作符**

`=` 等于  
`<>` 不等于  
`!=` 不等于  
`>` 大于  
`>=` 大于等于  
`<` 小于  
`<=` 小于等于

**时间字符串**

`u`或`µ` 微秒  
`ms` 毫秒  
`s` 秒  
`m` 分钟  
`h` 小时  
`d` 天  
`w` 星期

### 正则表达式 <a id="&#x6B63;&#x5219;&#x8868;&#x8FBE;&#x5F0F;"></a>

InluxDB支持在以下场景使用正则表达式：

* 在`SELECT`中的field key和tag key
* 在`FROM`中的measurement
* 在`WHERE`中的tag value和字符串类型的field value
* 在`GROUP BY`中的tag key

目前，InfluxQL不支持在`WHERE`中使用正则表达式去匹配不是字符串的field value，以及数据库名和retention policy。

> 注意：正则表达式比精确的字符串更加耗费计算资源; 具有正则表达式的查询比那些没有的性能要低一些。

#### 语法 <a id="&#x8BED;&#x6CD5;"></a>

```text
SELECT /<regular_expression_field_key>/ FROM /<regular_expression_measurement>/ WHERE [<tag_key> <operator> /<regular_expression_tag_value>/ | <field_key> <operator> /<regular_expression_field_value>/] GROUP BY /<regular_expression_tag_key>/
```

#### 语法描述 <a id="&#x8BED;&#x6CD5;&#x63CF;&#x8FF0;"></a>

正则表达式前后使用斜杠`/`，并且使用[Golang的正则表达式语法](http://golang.org/pkg/regexp/syntax/)。

支持的操作符：

`=~` 匹配  
`!~` 不匹配

### 数据类型和转换 <a id="&#x6570;&#x636E;&#x7C7B;&#x578B;&#x548C;&#x8F6C;&#x6362;"></a>

在`SELECT`中支持指定field的类型，以及使用`::`完成基本的类型转换。

#### 数据类型 <a id="&#x6570;&#x636E;&#x7C7B;&#x578B;"></a>

field的value支持浮点，整数，字符串和布尔型。`::`语法允许用户在查询中指定field的类型。

> 注意：一般来说，没有必要在SELECT子句中指定字段值类型。 在大多数情况下，InfluxDB拒绝尝试将字段值写入以前接受的不同类型的字段值的字段的任何数据。字段值类型可能在分片组之间不同。在这些情况下，可能需要在SELECT子句中指定字段值类型。

**语法**

```text
SELECT_clause <field_key>::<type> FROM_clause
```

**语法描述**

`type`可以是`float`，`integer`，`string`和`boolean`。在大多数情况下，如果`field_key`没有存储指定`type`的数据，那么InfluxDB将不会返回数据。

**例子**

```text
> SELECT "water_level"::float FROM "h2o_feet" LIMIT 4

name: h2o_feet
--------------
time                   water_level
2015-08-18T00:00:00Z   8.12
2015-08-18T00:00:00Z   2.064
2015-08-18T00:06:00Z   8.005
2015-08-18T00:06:00Z   2.116
```

该查询返回field key`water_level`为浮点型的数据。

#### 类型转换 <a id="&#x7C7B;&#x578B;&#x8F6C;&#x6362;"></a>

`::`语法允许用户在查询中做基本的数据类型转换。目前，InfluxDB支持冲整数转到浮点，或者从浮点转到整数。

**语法**

```text
SELECT_clause <field_key>::<type> FROM_clause
```

**语法描述**

`type`可以是`float`或者`integer`。

如果查询试图把整数或者浮点数转换成字符串或者布尔型，InfluxDB将不会返回数据。

### 多语句 <a id="&#x591A;&#x8BED;&#x53E5;"></a>

用分号`;`分割多个`SELECT`语句。

#### 例子 <a id="&#x4F8B;&#x5B50;"></a>

**CLI:**

```text
> SELECT MEAN("water_level") FROM "h2o_feet"; SELECT "water_level" FROM "h2o_feet" LIMIT 2

name: h2o_feet
time                   mean
----                   ----
1970-01-01T00:00:00Z   4.442107025822522

name: h2o_feet
time                   water_level
----                   -----------
2015-08-18T00:00:00Z   8.12
2015-08-18T00:00:00Z   2.064
```

### 子查询 <a id="&#x5B50;&#x67E5;&#x8BE2;"></a>

子查询是嵌套在另一个查询的`FROM`子句中的查询。使用子查询将查询作为条件应用于其他查询。子查询提供与嵌套函数和SQL`HAVING`子句类似的功能。

#### 语法 <a id="&#x8BED;&#x6CD5;"></a>

```text
SELECT_clause FROM ( SELECT_statement ) [...]
```

#### 语法描述 <a id="&#x8BED;&#x6CD5;&#x63CF;&#x8FF0;"></a>

InfluxDB首先执行子查询，再次执行主查询。

主查询围绕子查询，至少需要`SELECT`和`FROM`子句。主查询支持本文档中列出的所有子句。

子查询显示在主查询的`FROM`子句中，它需要附加的括号。 子查询支持本文档中列出的所有子句。

InfluxQL每个主要查询支持多个嵌套子查询。 多个子查询的示例语法：

```text
SELECT_clause FROM ( SELECT_clause FROM ( SELECT_statement ) [...] ) [...]
```

