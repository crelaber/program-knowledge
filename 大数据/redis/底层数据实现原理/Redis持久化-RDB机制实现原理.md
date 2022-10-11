### 概览

上一篇文章主要讲述了AOF持久化机制的实现原理，我们知道AOF日志在保证性能的同时，也能最大程度保证日志的完整性，即使出现服务器宕机等异常情况，也最多只会丢失1s的日志数据。

但因为其记录的是原始的写入指令，因此需要通过指令重放的方式恢复数据，在数据量非常大的情况下，会导致整体恢复耗时较长。

而Redis同时提供了另外一种基于内存快照的持久化技术，可以将数据库某一时刻的状态记录下来存入日志，后续数据恢复时只需要将日志中的数据载入内存即可，因此数据恢复效率更高。

这一持久化技术就是指Redis的RDB持久化机制，本文主要讲述RDB的实现原理。

### 实现原理

RDB持久化时，会将该时刻的数据库快照写入日志，涉及数据库的全量数据。![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLgHIe5WDtxfHEOniaqFOatx2JqmRiaI1EOIcX6QTUdCzfM7zFBr9DIMHenBfXsy4Ig9SDGxJGl41bg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

如果涉及的数据量较大，那么这个操作是比较耗时的。因此持久化操作是否会阻塞主线程，影响Redis性能，这一点至关重要。

Redis提供了两个指令显示触发RDB持久化操作：

- SAVE：在主线程中执行，会导致主线程阻塞
- BGSAVE：创建一个子进程，专门用于RDB持久化操作，避免了主线程阻塞，这也是Redis的默认配置

BGSAVE 子进程在进行快照期间，允许数据发生修改，其同样也是通过写时复制技术实现的：

![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLgHIe5WDtxfHEOniaqFOatxtHchUqxV4uCUZXKJEpYGfz9iaQeZSspkGAQD0Xyetoo4pFjMMI3OrrA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从图示的例子中可以看到，在 RDB 进行快照的过程中，Key-3发生了修改，此时会额外为 Key-3开辟一块内存空间，在新开辟的空间上进行修改操作，而不影响RDB快照所依赖的旧数据。

除了可以显示触发RDB持久化，Redis还提供自动触发机制，在 redis.conf 可以进行配置：

![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLgHIe5WDtxfHEOniaqFOatxFlUMuw44EMocU2nx9xgGEQuiajCF4mCicb7LSRDIx1PsTq3DCRBjqYCw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

需要注意的是：

- 同一时间，只能有一个BGSAVE子进程执行
- BGREWRITEAOF和BGSAVE两个指令不能同时执行

这么做的原因主要是BGSAVE，BGREWRITEAOF指令都比较占用系统资源，会产生大量的磁盘写入操作，如果允许同时执行则可能影响服务器整体性能。

### 混合持久化

通过上述内容，我们了解到AOF与RDB两种持久化机制各有优缺点：

- AOF日志可以保证日志完整性，但数据恢复效率较差
- RDB日志数据恢复效率很高，但日志完整性较差

因此在Redis 4.0之后，推出了混合持久化机制，通过redis.conf可以配置是否开启：

```
aof-use-rdb-preamble yes
```

开启混合持久化之后，在AOF重写时，会先以RBD持久化的方式将快照数据记录在日志开头；重写完成后，再将重写期间发生的写指令以AOF格式追加到日志结尾。

重写之后的AOF日志格式如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLgHIe5WDtxfHEOniaqFOatxALuasr0j5QEhbKn8jiaicY3dEJTIGXIPHcWm7HVG7b3X9XTqF6YiaatwQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Redis 在启动时会优先根据 AOF 文件进行数据恢复，流程如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLgHIe5WDtxfHEOniaqFOatxcRDZUUtmJtOYPrWtZ5ia3jictcbwbqeG0knuqjM9aOswZKSxDLH6ImmQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

因此，混合持久化机制结合了RDB和AOF的优点，既能够保证日志的完整性，且数据恢复效率也高，在Redis 5.0是默认开启的。