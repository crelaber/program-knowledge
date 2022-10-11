

### 概览

Redis 作为内存数据库，通常情况下数据存储在内存当中，而读写操作都是基于内存中的数据进行的，但这样也意味着如果服务器出现异常情况宕机了，存储在内存中的数据也随之丢失。

如果依赖于外力进行数据恢复，成本较高且效率较低，而 Redis 本身提供了数据持久化技术，在服务器出现异常宕机之后，能快速恢复宕机前的数据。

Redis 提供了两种持久化机制：

•AOF(Append Only File)，记录数据变化涉及的写入指令•RDB(Redis Database)，记录某个时刻数据库的数据快照信息

两种持久化机制有各自的优缺点，应对不同的使用场景。本文主要讲述 AOF 持久化机制的实现原理以及优缺点。

### AOF实战

我本地使用的 Redis 版本是 5.0.8，默认 AOF 持久化是关闭的，修改 redis.conf 配置将其打开：

```
appendonly yes
```

设置 AOF 日志文件名称：

```
appendfilename "appendonly.aof"
```

设置日志的写回策略，默认是 everysec（下文会说明几种写回策略的区别）：

```
# 每秒写回策略appendfsync everysec
```

设置日志自动重写的触发条件：

```
# 当日志比上次重写时大小增长了一倍auto-aof-rewrite-percentage 100# 当日志大小大于 64mbauto-aof-rewrite-min-size 64mb
```

配置完成后，重启 Redis 服务器，通过客户端执行写入命令：

```
127.0.0.1:6379> lpush testlist 1(integer) 1127.0.0.1:6379> lpush testlist 2(integer) 2127.0.0.1:6379> lpush testlist 2(integer) 3
```

发现对应的目录（默认是当前用户的主目录）下并没有生成 appendonly.aof 文件，通过显示执行日志重写命令，才触发了日志文件生成；

```
127.0.0.1:6379> BGREWRITEAOFBackground append only file rewriting started
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLv8Rl8s8vOrG3n1pkFUjiayyTX50sa7LuWZWr91ojdhf60VK4qv0cqMWqj1t2l8RsjkUeevhUaFaQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这个问题可能与 Redis 的持久化机制有关（先埋个坑）。

### 实现原理

AOF 日志的实现包含两个核心点：

- 写回策略，性能与可靠性的权衡
- 日志重写，控制日志文件大小

### 写回策略

Redis 记录 AOF 日志时，何时把日志更新同步到磁盘中，取决于写回策略。

Redis 提供了三种写回策略，应对不同场景的需求，在 redis.conf 中可以配置写回策略：

![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLv8Rl8s8vOrG3n1pkFUjiaymfRkmMYcZZnTtBCAbSIn4bPib7WmvqzXVDDzEiam8mDTOxib9qDBQ54xQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- always 同步写回：每处理完一个写事件，都会同步将日志更新同步刷到磁盘，在日志完整性方面是最可靠的策略，当然对性能影响也最大；
- everysec 每秒写回：通过后台线程，每隔一秒将日志更新同步刷到磁盘，是性能和可靠性权衡之下的最佳策略；
- no 操作系统控制写回：由操作系统决定何时将日志更新同步刷到磁盘，可靠性最差，但性能最好；

### 日志重写

如果 AOF 日志体量不断膨胀，则会出现以下问题：

- 文件系统本身对文件大小有限制，无法保存过大的文件
- 当文件过大，后续往里面追加内容，效率也会变低
- 当文件过大，如果服务器发生宕机，后续基于 AOF 文件恢复的过程也会非常缓慢

因此 Redis 提供了 AOF 日志重写机制，用于控制日志文件的大小，它需要解决以下问题：

- **如何有效控制日志大小**
- *非阻塞的重写机制，在不影响客户端写入操作的情况下，如何保证日志数据完整性**

日志重写是基于 Redis 数据库某一时刻的快照，将数据反向转换成指令的过程， 下面是重写机制的实现原理：![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLv8Rl8s8vOrG3n1pkFUjiayibN9IehJT4XY7c5NVac9lTHicRMMsxw0Maa3SGbyjjicn9zc3pYtJXXKg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- Redis 通过 fork 子进程的方式，在后台进行日志重写工作
- 日志重写时会根据数据库当时的状态，生成一份 Redis 数据拷贝，需要注意的是 Redis 采用的是写时复制技术，正常情况下子进程与父进程共享同一片内存空间，只有当父进程中数据发生变更时，才会开辟新的内存空间，产生实际的数据复制
- 当日志重写完成之后，会将旧日志文件替换为重写后的新日志文件。在重写期间，旧日志文件也会正常写入日志，这样即使重写发生了异常，也不会影响整体日志的完整性

首先为什么日志重写，可以有效控制日志文件大小？

举个例子：![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLv8Rl8s8vOrG3n1pkFUjiay9OZbqaeNQNZCD67ILKICWtWOOqwXz5MgSoGNbghnppVXhHnWfuDZmQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**由于 AOF 日志重写时，依据的是数据库当前的状态，因此可以省略很多过程中产生的指令，只需生成达到当前数据状态需要的写入指令即可。**

Redis 可以在不阻塞主线程正常工作的情况下，进行日志的重写工作。这样随之而来有另外一个问题：在子进程进行日志重写的期间，主进程发生了数据变更，而重写完成的日志没办法感知到这期间的发生变更，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLv8Rl8s8vOrG3n1pkFUjiayExkfDP9zXAln8Ar2UtOIO0HuuibOmyfohF6E05v1ItrMoOjAcyicFe1A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

数据在 Time-2 时刻发生修改，而日志重写依赖的还是 Time-1 时刻的快照数据。

Redis 通过写两份缓冲数据的方式解决该问题：![图片](https://mmbiz.qpic.cn/mmbiz_png/6SS7gx5ZuxLv8Rl8s8vOrG3n1pkFUjiaysicQK9aEOu3qkVlUAW6XEKyXpSCDruT8w9DutRmSZuTlKYPSsiaHvJhQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**在重写期间，会将数据变化同时写入到 AOF重写缓冲区，用以记录重写期间发生的数据变更，当重写完成后，再将重写缓冲区中记录的变更追加到重写日志尾部，通过这种方式解决日志数据完整性的问题。**

### 优缺点

优点如下：

- AOF更可靠，不同的写回策略可以让用户根据使用场景灵活选择，默认的 everysec 写回策略在保证良好性能的同时，即使发生了异常宕机，最多也只会丢失 1s 的日志数据
- AOF日志中内容以Redis协议格式保存，因此日志具有较高的可读性，更易于对日志进行分析

缺点如下：

- 相同数据集的情况下，AOF文件通常比RDB文件要大，需要有额外的重写机制控制日志文件的大小
- 在重写期间如果发生了数据变更，会比平时更占用内存，因为需要同时往两个日志缓冲区写入数据
- 服务器重启后，通过逐一重放 AOF 日志中的写入指令进行数据恢复，如果 AOF 文件过大，则会导致整体恢复耗时较长