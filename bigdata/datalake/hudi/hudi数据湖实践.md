# hudi解决的问题

## schema增量字段位置乱序对齐方案

* 基于hudi携带的schema evolution可以解决历史数据base schema从(id、age、name)变更为(id,sex,age,name)这种乱序schema的情况。

![](./img/schema演进.jpg)

## 一条数据进入Hudi的流程

![](./img/数据流.jpg)

![](./img/数据流2.jpg)

## 流量毛刺问题

### 集中毛刺更新

#### 离线架构存在的问题

1. 数据峰值任务处理周期长、引起时效性报警。
2. 数据量持续变大引起GC严重，任务OOM问题。
3. 峰值持续时间长、造成数据积压，失败的任务重启失败。

#### Hudi解决方式

![](./img/流量毛刺.jpg)

# hudi生产实践

## 参数设置

1. 0.8.0版本写Hudi重试失败导致数据丢失风险。解决办法：`hoodie.datasource.write.streaming.ignore.failed.batch`设置为false，不然Task会间隔`hoodie.datasource.write.streaming.retry.interval.ms`(默认2000)重试`hoodie.datasource.write.streaming.retry.count`(默认3)
2. 根据业务场景合理设置保留版本数对应参数为`hoodie.keep.min.commits`、`hoodie.keep.max.commits`调大cleanup retention版本数对应参数为`hoodie.cleaner.commits.retained`
3. 0.8.0版本Upsert模式下数据丢失问题。解决办法：`hoodie.datasource.write.insert.drop.duplicates`设置为false，这个参数会将已经存在index的record丢弃，如果存在update的record会被丢弃
4. Spark读取hudi可能会存在path not exists的问题，这个是由于cleanup导致的，解决办法：调整文件版本并进行重试读取

## 部分更新

* Hudi中默认只支持覆盖更新，采集端点上报的数据不可能包含完整的业务对象，如用户年龄的增长，在上报时只会包含一个字段的信息

```json
{
  "id": 123,
  "ts": 1435290195610,
  "data": {
    "age": 25
  }
}
```

* 这就需要先找出 rowkey=123 的数据内容，并与待更新内容进行合并后再写入。合并时如果待写入数据的字段不为空，那么进行归并。Hudi默认采用 `OverwriteWithLatestAvroPayload` 的 `combineAndGetUpdateValue` 方法,这样会导致如果写入的值为null的话会覆盖掉历史数据。
* 可以采用`OverwriteNonDefaultsWithLatestAvroPayload`这样空值不会进行覆盖，如果真的希望将某个字段更新为空值，那么使用 `OverwriteNonDefaultsWithLatestAvroPayload` 是无法实现的。

## Hudi结合数据缓存层Alluxio

### 数据入湖

* 将`Alluxio`与计算集群共同部署，在数据入湖之前将对应的数据存储Path挂载至`Alluxio`文件系统中，然后设置`Hudi`的`--target-base-path`参数为`alluxio://...`。在数据入湖时，使用Spark引擎拉起`Hudi`程序不断摄入数据，数据此时在`Alluxio`中流转，`Hudi`程序拉起后，设置每分钟将数据从`Allxuio`缓存中异步同步至远程OSS。这样Spark从之前的写远程OSS转变为写本地的Alluxio，缩短了数据入湖的时长。

### 湖上数据分析

* 使用`Presto`作为自助查询引擎，分析湖上的`Hudi`表。在每一个`Presto worker`节点共置`Alluxio`。当`Presto`与`Alluxio`服务共置运行时，`Alluxio`可能会将输入数据缓存到`Presto worker`的本地，并以内存速度提供下次检索。在这种情况下，`Presto`可以利用`Alluxio`从本地的`Alluxio worker`存储读取数据（称之为短路读取），无需任何额外的网络传输。

### 跨多个存储系统的并发访问

* 为了确保训练样本的准确性，我们的机器学习团队经常将生产中的脱敏数据同步到离线机器学习环境。在同步期间，数据跨多个文件系统流动，从生产OSS到线下数据湖集群HDFS，最后同步到机器学习集群的HDFS。对于数据建模人员来说，数据迁移过程不仅效率低下，而且会因错误配置而导致出错，因为其中涉及多个不同配置的文件系统。于是我们引入Alluxio,将多个文件系统都挂载到同一个Alluxio下，统一了命名空间。端到端对接时，使用各自的Alluxio路径，这保证了具有不同API的应用程序无缝访问和传输数据。这种数据访问布局还可以提高性能。

#### Alluxio优势

- Alluxio 支持层次化且透明的缓存机制；
- Alluxio 支持读取时缓存 promote 模式；
- Alluxio 支持异步写模式；
- Alluxio 支持 LRU 回收策略；
- Alluxio 拥有 pin 以及 TTL 特性；
