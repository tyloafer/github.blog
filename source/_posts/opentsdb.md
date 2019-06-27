---
title: OpenTSDB概述
tags:
  - OpenTSDB
categories:
  - OpenTSDB
date: '2019-05-19 20:45'
abbrlink: 44478
---
OpenTSDB 是一个时系列数据（库），它基于HBase存储数据，充分发挥了HBase的分布式列存储特性，支持数百万每秒的读写，它的特点就是容易扩展，灵活的tag机制。

<!--more-->

## 存储相关概念

例：假设我们采集1个服务器（hostname=qatest）的CPU使用率，发现该服务器在21:00的时候，CPU使用率达到99%

- Metric：即平时我们所说的监控项。譬如上面的CPU使用率

- Tags：就是一些标签，在OpenTSDB里面，Tags由tagk和tagv组成，即tagk=takv。标签是用来描述Metric的，譬如上面为了标记是服务器A的CpuUsage，tags可为hostname=qatest

- Value：一个Value表示一个metric的实际数值，譬如上面的99%

- Timestamp：即时间戳，用来描述Value是什么时候的，譬如上面的21:00

- Data Point：即某个Metric在某个时间点的数值。

  ```
  Data Point包括以下部分：Metric、Tags、Value、Timestamp
  ```

     上面描述的服务器在21:00时候的cpu使用率，就是1个DataPoint



## 查询组件

| 参数                 | 数据类型        | Required | 描述                                                         | 示例               |
| -------------------- | --------------- | -------- | ------------------------------------------------------------ | ------------------ |
| Start Time           | String或Integer | Required | 查询的开始时间。可以是绝对时间或相对时间                     | 24h-ago            |
| End Time             | String或Integer | Optional | 查询的结束时间。如果未提供结束时间，则当前时间即结束时间     | 1h-ago             |
| Metric               | String          | Required | 系统中的metric全名。必须是全名并且大小写敏感                 | sys.cpu.user       |
| Aggregation Function | String          | Required | 用于组合多个时间序列的数学函数（即如何合并一个组中的时间序列值） | sum                |
| Filter               | String          | Optional | 过滤标签值以减少查询或组中挑选出的时间序列的数量，并聚合各个标签 | host=*,dc=lax      |
| Downsampler          | String          | Optional | 可选的时间间隔和函数，用于减少随时间返回的数据点的数量       | 1h-avg             |
| Rate                 | String          | Optional | 用于计算结果的每秒变化率                                     | rate               |
| Functions            | String          | Optional | 数据处理函数，如附加过滤、时间切换等                         | highestMax(…)      |
| Expressions          | String          | Optional | 数据处理函数，例如将一个序列分化成另一个序列                 | (m2/(m1 + m2))*100 |



### Aggregation(聚合)

| 聚合器  | TSD版本 | 描述                                                     | 插值        |
| ------- | ------- | -------------------------------------------------------- | ----------- |
| avg     | 1.0     | 数据点平均值                                             | 线性插值    |
| count   | 2.2     | 集合中原始数据点的数量                                   | 0替换缺失值 |
| dev     | 1.0     | 计算标准差                                               | 线性插值    |
| Ep50r3  | 2.2     | 使用R-3方法计算估计的50%                                 | 线性插值    |
| Ep50r7  | 2.2     | 使用R-7方法计算估计的50%                                 | 线性插值    |
| Ep75r3  | 2.2     | 使用R-3方法计算估计的75%                                 | 线性插值    |
| Ep75r7  | 2.2     | 使用R-7方法计算估计的75%                                 | 线性插值    |
| Ep90r3  | 2.2     | 使用R-3方法计算估计的90%                                 | 线性插值    |
| Ep90r7  | 2.2     | 使用R-7方法计算估计的90%                                 | 线性插值    |
| Ep95r3  | 2.2     | 使用R-3方法计算估计的95%                                 | 线性插值    |
| Ep95r7  | 2.2     | 使用R-7方法计算估计的95%                                 | 线性插值    |
| Ep99r3  | 2.2     | 使用R-3方法计算估计的99%                                 | 线性插值    |
| Ep99r7  | 2.2     | 使用R-7方法计算估计的99%                                 | 线性插值    |
| Ep999r3 | 2.2     | 使用R-3方法计算估计的999%                                | 线性插值    |
| Ep999r7 | 2.2     | 使用R-7方法计算估计的999%                                | 线性插值    |
| first   | 2.3     | 返回集合中的第一个数据点。仅仅对降采样有用，对聚合无用   | 不定        |
| last    | 2.3     | 返回集合中的最后一个数据点。仅仅对降采样有用，对聚合无用 | 不定        |
| mimmin  | 2.0     | 筛选最小的数据点                                         | 线性插值    |
| mimmax  | 2.0     | 筛选最大的数据点                                         | 线性插值    |
| min     | 1.0     | 筛选最小的数据点                                         | 线性插值    |
| max     | 1.0     | 筛选最大的数据点                                         | 线性插值    |
| none    | 2.3     | 通过所有时间序列的聚合跳过组                             | 0替换缺失值 |
| p50     | 2.3     | 计算50%                                                  | 线性插值    |
| p75     | 2.3     | 计算75%                                                  | 线性插值    |
| p90     | 2.3     | 计算90%                                                  | 线性插值    |
| p95     | 2.3     | 计算95%                                                  | 线性插值    |
| p99     | 2.3     | 计算99%                                                  | 线性插值    |
| p999    | 2.3     | 计算999%                                                 | 线性插值    |
| sum     | 1.0     | 将数据点一起求和                                         | 线性插值    |
| zimsum  | 1.0     | 将数据点一起求和                                         | 0替换缺失值 |



### Functions(函数)

#### Bosun语法

- q(query string, startDuration string, endDuration string) seriesSet *

- band(query string, duration string, period string, num scalar) seriesSet *

- over(query string, duration string, period string, num scalar) seriesSet *

- change(query string, startDuration string, endDuration string) numberSet *

- count(query string, startDuration string, endDuration string) scalar *

- window(query string, duration string, period string, num scalar, funcName string) seriesSet *

- avg(seriesSet) numberSet
  取数据集的平均值

- sum(seriesSet) numberSet
  求和

- cCount(seriesSet) numberSet
  数据集的变化程度，根据每个数据与它前一个数据的比较得出。例如，数据集cCount([0,1,0,1])返回3

- dev(seriesSet) numberSet
  标准差

- diff(seriesSet) numberSet
  最后一个数减去第一个数的差值

- first(seriesSet) numberSet
  数据集的第一个数据

- last(seriesSet) numberSet
  数据集的最后一个数据

- len(seriesSet) numberSet
  数据集的长度

- max(seriesSet) numberSet
  最大值

- median(seriesSet) numberSet
  中间值

- min(seriesSet) numberSet
  最小值

- percentile(seriesSet, p numberSet|scalar) numberSet
  返回数据集前百分比为p的数据，例如最大值max相当于p>=1, 中间值相当于p==0.5

- since(seriesSet) numberSet

- Returns the number of seconds since the most recent data point in each series.

- streak(seriesSet) numberSet
  连续非零数据的最大长度

- t(numberSet, group string) seriesSet

  Transposes N series of length 1 to 1 series of length N. If the group parameter is not the empty string, the number of series returned is equal to the number of tagks passed. This is useful for performing scalar aggregation across multiple results from a query. For example, to get the total memory used on the web tier: sum(t(avg(q("avg:os.mem.used{host=-web}", "5m", "")), "")).

- alert(name string, key string) numberSet *

- abs(numberSet) numberSet
  返回数据集中每个元素的绝对值

- d(string) scalar
  返回一个时间段的秒数，例如d("1h")返回60

- tod(scalar) string
  Returns an OpenTSDB duration string that represents the given number of seconds. This lets you do math on durations and then pass it to the duration arguments in functions like q()
  des(series, alpha scalar, beta scalar) series
  Returns series smoothed using Holt-Winters double exponential smoothing. Alpha (scalar) is the data smoothing factor. Beta (scalar) is the trend smoothing factor.

- dropg(seriesSet, threshold numberSet|scalar) seriesSet
  移除数据集中大于指定阀值的元素

- dropge(seriesSet, threshold numberSet|scalar) seriesSet
  移除数据集中大于等于指定阀值的元素

- dropl(seriesSet, threshold numberSet|scalar) seriesSet
  移除数据集中小于等于指定阀值的元素

- drople(seriesSet, threshold numberSet|scalar) seriesSet
  移除数据集中小于等于指定阀值的元素

- dropna(seriesSet) seriesSet
  移除数据集中类型是NaN是Inf的元素

- dropbool(seriesSet, seriesSet) seriesSet

- epoch() scalar
  返回当前的Unix时间戳

- filter(seriesSet, numberSet) seriesSet

- limit(numberSet, count scalar) numberSet

- lookup(table string, key string) numberSet

- shift(seriesSet, dur string) seriesSet

- merge(SeriesSet…) seriesSet

- nv(numberSet, scalar) numberSet

## 示例

### 简单聚合测试

| 序号 | metric      | tag                   | value |
| ---- | ----------- | --------------------- | ----- |
| 1    | test.host.2 | dc=dal host=web01     | 3     |
| 2    | test.host.2 | dc=dal host=web02     | 2     |
| 3    | test.host.2 | dc=dal host=web03     | 10    |
| 4    | test.host.2 | host=web01            | 1     |
| 5    | test.host.2 | host=web01 owner=jdoe | 4     |
| 6    | test.host.2 | dc=lax host=web01     | 8     |
| 7    | test.host.2 | dc=lax host=web02     | 4     |



1. 单tag查询

   > q("sum:test.host.2{host=web01}", "24h", "")

   > 匹配结果：1，4，5，6

   | group          | result                | computations |
   | :------------- | :-------------------- | :----------- |
   | { host=web01 } | `{"1557327756": 16 }` |              |

   > q("avg:test.host.2{host=web01}", "24h", "")

   | group          | result                 | computations |
   | :------------- | :--------------------- | :----------- |
   | { host=web01 } | `{   "1557327756": 4}` |              |

   > q("count:test.host.2{host=web01}", "24h", "")

   |                |                         |              |
   | :------------- | :---------------------- | :----------- |
   | group          | result                  | computations |
   | { host=web01 } | `{   "1557327756": 4 }` |              |

2. 多tag查询

   > q("sum:test.host.2{dc=dal,host=web01}", "24h", "")

   > 匹配结果：1

   | group                  | result                  | computations |
   | :--------------------- | :---------------------- | :----------- |
   | { dc=dal, host=web01 } | `{   "1557327756": 3 }` |              |

   > q("avg:test.host.2{dc=dal,host=web01}", "24h", "")

   | group                  | result                  | computations |
   | :--------------------- | :---------------------- | :----------- |
   | { dc=dal, host=web01 } | `{   "1557327756": 3 }` |              |

   > q("count:test.host.2{dc=dal,host=web01}", "24h", "")

   | group                  | result                  | computations |
   | :--------------------- | :---------------------- | :----------- |
   | { dc=dal, host=web01 } | `{   "1557327756": 1 }` |              |

3. 正则*查询

   > q("sum:test.host.2{dc=dal,host=*}", "24h", "")

   > 匹配结果：1，2，3
   >
   > 匹配结果会以 查询中的唯一tag为一个聚类

   | group                  | result                   | computations |
   | :--------------------- | :----------------------- | :----------- |
   | { dc=dal, host=web01 } | `{   "1557327756": 3 }`  |              |
   | { dc=dal, host=web02 } | `{   "1557327756": 2 }`  |              |
   | { dc=dal, host=web03 } | `{   "1557327756": 10 }` |              |

   > q("avg:test.host.2{dc=dal,host=*}", "24h", "")

   | group                  | result                   | computations |
   | :--------------------- | :----------------------- | :----------- |
   | { dc=dal, host=web01 } | `{   "1557327756": 3 }`  |              |
   | { dc=dal, host=web02 } | `{   "1557327756": 2 }`  |              |
   | { dc=dal, host=web03 } | `{   "1557327756": 10 }` |              |

   > q("count:test.host.2{dc=dal,host=*}", "24h", "")

   | group                  | result                  | computations |
   | :--------------------- | :---------------------- | :----------- |
   | { dc=dal, host=web01 } | `{   "1557327756": 1 }` |              |
   | { dc=dal, host=web02 } | `{   "1557327756": 1 }` |              |
   | { dc=dal, host=web03 } | `{   "1557327756": 1 }` |              |

4. 正则|查询

   > q("sum:test.host.2{dc=dal|lax}", "24h", "")

   > 匹配结果：1，2，3，6，7

   | group      | result                   | computations |
   | :--------- | :----------------------- | :----------- |
   | { dc=dal } | `{   "1557327756": 15 }` |              |
   | { dc=lax } | `{   "1557327756": 12 }` |              |

   > q("avg:test.host.2{dc=dal|lax}", "24h", "")

   | group      | result                  | computations |
   | :--------- | :---------------------- | :----------- |
   | { dc=dal } | `{   "1557327756": 5 }` |              |
   | { dc=lax } | `{   "1557327756": 6 }` |              |

   > q("count:test.host.2{dc=dal|lax}", "24h", "")

   | group      | result                  | computations |
   | :--------- | :---------------------- | :----------- |
   | { dc=dal } | `{   "1557327756": 3 }` |              |
   | { dc=lax } | `{   "1557327756": 2 }` |              |



### rate测试

| 序号 | metric      | tag        | value | Time       |
| ---- | ----------- | ---------- | ----- | ---------- |
| 1    | test.host.3 | host=host1 | 4     | 1557327757 |
| 2    | test.host.3 | host=host1 | 8     | 1557327758 |
| 3    | test.host.3 | host=host1 | 12    | 1557327759 |
| 4    | test.host.3 | host=host1 | 16    | 1557327760 |
| 5    | test.host.3 | host=host1 | 16    | 1557327761 |
| 6    | test.host.3 | host=host1 | 17    | 1557327763 |

> q("avg:rate:test.host.3{host=host1}", "24h", "")

| group          | result                                                       | computations |
| :------------- | :----------------------------------------------------------- | :----------- |
| { host=host1 } | `{   "1557327757": 4,   "1557327758": 4,   "1557327759": 4,   "1557327760": 4,   "1557327761": 0,   "1557327763": 0.5 }` |              |

