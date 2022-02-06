### Rocketmq 消息文件删除策略

**对于过期文件**
 1)通过设置删除过期文件的时间，会在这个小时内去删除文件，每次删除10个。
 相关配置参数：

```json
deleteWhen=04 删除文件时间点，默认是凌晨4点，24小时制，可以通过;分隔配置多个
fileReservedTime=72          文件保留时间，默认48小时    
```

2)通过设置磁盘存储空间，达到了阈值就会删除过期的文件。
 相关配置参数：

```
diskMaxUsedSpaceRatio=75% 默认75%
fileReservedTime=72          文件保留时间，默认48小时
```

**对于没有过期的文件**
 1)磁盘存储空间达到强制清理阈值,(通过启动命令设置)

```
-Drocketmq.broker.diskSpaceCleanForciblyRatio=0.85   强制清理,默认85%
destroyMapedFileIntervalForcibly= 1000 * 120  ms 删除的文件被引用时，不会马上被删除，最大的存活时间
```

2)磁盘存储空间达到预警线，(通过启动命令设置)

```
-Drocketmq.broker.diskSpaceWarningLevelRatio=0.90    禁止写入,并清理，默认90%
destroyMapedFileIntervalForcibly= 1000 * 120  ms 删除的文件被引用时，不会马上被删除，最大的存活时间
```

**3.结论**    
 对于过期的文件，不会存在堆积的问题。出现此种情况一般是短时间大流量产生的。
 为了避免生产环境流量暴增引起的短时间消息堆积。
 建议业务系统加强消费能力，不要让消息堆积，消息文件不被占用就可以更安全的被删除。
 fileReservedTime设置合理的时间，保持可用磁盘空间在一定程度，可以挡住短时间的流量冲击。
 发生堆积时优先删除同一个磁盘空间的其它无用日志
 找到消息量大的topic，动态扩容broker，update这些topic的broker。