## 背景

最近上线了一个kafka的消费者，数据规模大概是低峰期单机每分钟消费88W条，QPS 14666。上线后看了下数据，进程CPU到了132%。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWn9uicyTibhAmz18P4zlQ6IjvPSUpj8kD7PgVa03OjiaMk8PCjQAAzu1HOA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

8核的机器，单进程CPU132倒也还好，但还是想看看，到底是咋回事。

## 过程

### 第一次排查&优化（协程池化->约为0优化）

于是就开始采集pprof的数据。golang pprof的采集是十分便捷的，在`main.go`引入`net/http/pprof`包，包里`pprof.go`文件的`init()`方法就会自动注册相关的http路由。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnxBHgyoYicQyUFUnMaQCZoAbMoO4J7RLgBafvM7v8UfoBuG4tiaibUMD2w/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

CPU的火焰图看着就有点不合理，**光是runtime的部分，居然耗费了1/3的CPU**。

首先怀疑是goroutine创建过多的问题，我们消费者框架如下图，服务从kafka消费到一条msg后，会分发给每一个plugin，为了plugin之间互不影响，所以都是异步调用plugin的。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnZUe77SPQV6Tia4qQv8AeIzQCtQS0DtokBLmVHUpAErWgb6oDicxmvqNQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

所以这里每条消息会有放大的问题，这个服务有3个plugin，每条消息就会创建3个goroutine，也就是每秒创建14666*3约45000 goroutine。

解决办法也简单，就是池化，以达到goroutine复用的目的，也就是老生常谈的协程池了。这里用了我司的一位go社区大牛的协程池库**ants**[1](可惜这位大牛已经江湖见了我哭死)，有协程池需求的可吃波安利。

#### 结果&问题

但上线后发现也就只有一点点效果，pprof再看了下goroutine的部分，采样到的goroutine总数其实不多，这一步优化的前后的采样也其实没太大区别。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWniao6qynOZvicUahqicnaBbXvR2bCiauyVDtgrl7A3BwcWBPljY9aFtVVBw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

而且想了下线上其他feed之类的服务，每个请求还并发拉多个数据源来拼数据，那种服务的goroutine创建可猛多了，但也不会像我这个服务，光是GMP就占了1/3。

这一步优化，最终的结果就是，强行把框架的TODO完成了。。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnV9UqPz93kicv17nqibNaf1qhAq2BIO4pSEBrgQtYOPWqP1RuUkPHPNiaw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

### 第二次排查&优化（定时器“泄漏” -> 初见成效）

pprof看不出问题的话，就得考虑更多的性能分析工具了，于是开始用go trace，trace的路由是和pprof一同注册的，直接使用就行。trace的用法要稍复杂点，用法可移步文末的参考文章，这里就不贴了。

在刚开始查看问题时，不建议直接陷入goroutine调度的细节，因此一般先看 “Scheduler latency profile(调度延迟概况)”，能看到整体的调用开销情况，如下：

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnnc3NezaicVYl4mT05YUJOrCJQemfzB0gu71JIkoKvCoabh3WEjbOr2g/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

只能看到大部分的延迟是由select带来的。。看不出个所以然，于是想把下面的几个统计都先看看，结果看到Goroutine analysis时，发现了一个很怪异的数据。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnkEbMOypicb1g3JWQEUkvaWrfpTwFp1MSbSolj6ciciczDezOib5I48Viavg/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

Context居然执行过24W的goroutine。这里有点要说明的，上文的图三也是goroutine采样数据, 路由是`/debug/pprof/goroutine`，个数是1000左右。而trace的Goroutine analysis，goroutine数 20W+了，数量级明显不对。

可以看下`pprof.go`对于前者的注释，`"goroutine": "Stack traces of all current goroutines",` 显然前者统计的是现有的所有goroutine；而后者则是采样期间所有执行过的goroutine。

回到context那24W goroutine，追踪代码看到是从这里引入的，而`time.AfterFunc()`内部会使用goroutine

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnlLRkwjnPBlkE9m1K81Hek1UfVgJKovgcEsXYL5VV9H3dKZuWmVqBvg/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

看到这里，结合框架的代码就看出问题了。`Context.WithDeadline()`这个方法，会创建定时器，上面的注释也给我们说了，当上下文完成时要立马调用`cancel`来释放资源。但框架里用到这个函数的地方，只在err的时候立马释放了，正常情况的定时器，全都等到了执行时间执行，然后才释放资源所以才有那么多的goroutine执行。

```go
ctx, cancelFunc := context.WithTimeout(context.Background(), 5*time.Second)
err := s.Limiter.Wait(ctx)
if err != nil {
   log.Errorlnf("等待限流器错误，err:%v", err)
   cancelFunc()
   continue
}

ctxReadMsg, cancelFunc2 := context.WithTimeout(context.Background(), s.opt.FetchTimeout)
msg, err := s.reader.ReadMessage(ctxReadMsg)
if err != nil {
   if !errors.Is(err, context.DeadlineExceeded) {
      s.ErrorLogger.Printf("read message err:%v", err)
   }
   cancelFunc2()
}
```

大量的定时器调度，导致了GMP的调度需要很高的CPU，我是这么理解的。解决问题的办法更简单了，调用完成后直接`cancel()`，如

```go
ctx, cancelFunc := context.WithTimeout(context.Background(), 5*time.Second)
err := s.Limiter.Wait(ctx)
if err != nil {
   log.Errorlnf("等待限流器错误，err:%v", err)
   cancelFunc()
   continue
}
cancelFunc()
```

#### 结果&问题

上线后效果还是挺显著的，CPU成功从132 下降到 100，优化了1/4。看新的trace，goroutine也没了24W的大头

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnsNNEJqCPnwR1UVppdnUayakhVjj1DIm01pibIz2ZbAvYgNU1Iy3dWPQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnNQzohSDk0zBp7SyFSZdiaQwuaQqbiclx9TOAeoRJoriaB5ADnsOM77cZQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

但咋讲呢。runtime的CPU 还是占了很大一部分。。问题还是没有彻底解决。调度的部分，还是有25%的CPU调用，加上sysmon的已经30%了。golang这么优秀的语言，光是调度部分就这么耗CPU也太不讲道理了吧，肯定还有哪里不对。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWn1PbVFqKZosHNNg14ibqicyz0u7FhxGpDJGydD77qc57toyDLWOjH2Kzw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

### 第三次排查&优化（GMP的自旋 -> 更进一层）

这次把pprof和trace里所有的概况数据，以及具体的trace细节都看了，发现了有几个疑惑点。

1. 调度延迟里，大头都是有阻塞的

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnONPr29AiacbjiaPicGLa13bwhgMrA39W0nxaH428D2AmGgLmHpufQTRMQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

1. View Trace的细节，每50ms，总会核心数的利用率大概只有50%的情况，8核，只用了4核（更贴切的说法是8个P，只有4个在处理G）

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWn7PMSDppEs45D1xClePOxzNgGoWD7zRKKCtbMWP6At3shDrhFmjicjAg/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

1. 继续细化View trace，发现即便在工作看着很密集的时候，大多数时间其实也只有1-2个P在同时Work。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnGwE7DDWtxtJCaibmS3U5IP8fBp1KqiaH4hJcTN7YYNJfooQjHRu4nKcA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

这几个点概括起来就是，`1. 出现了较多的channel阻塞 2. P的使用率不高`。

于是到这里就得引入我们的GMP模型了。首先是P，go为了让新的G能尽快运行，所以会有一批P在不停自旋执行`findrunnable`，但自旋会耗费CPU啊，所以自旋的P也不能太多，而这个数是由`GOMAXPROCS`决定的，默认是CPU的核心数，我这里是8核的机器，所以P数量是8。

然后是阻塞，M关联P运行后若遇到channel阻塞，P会和M解绑，然后P继续找runnable的G。但我的服务是IO密集型，同一时间内大部分的G都在阻塞，所以能找到的也不多，同时有任务处理的P也不多。

这两个原因加起来就是，同时可运行的G不多，当前的P已经完全足够了，导致剩下的P都在白白自旋。在网上的博客中，也看到了类似的例子。

解决方法就更简单了，无非就是调低程序启动的时候，把`GOMAXPROCS`调低。

```
func main() {
    runtime.GOMAXPROCS(4)
    // ...
}
```

#### 结果

终于cpu从100 下降到 73，火焰图中，runtime调度的CPU占比也降低了8。这里其实还有一个点，对比图10，`runtime.sysmon`的在火焰图上看不到了，这里后面再细化下原因。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2KqcB5HVMK3c1J4a2If3XWnrBCHT8eV75kUL1f9axibdDJKhvjavjShgyfUa8rgcmvhZcgGsd1gxSw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

## 总结

1. golang 原生的pprof和trace支持，作为go开发者要熟练地用来做性能分析。
2. 带Deadline的Context，使用完记得及时回收资源。
3. golang 的 GMP模型，P的数量，不是越多越好。

## 参考文章

**Go 大杀器之跟踪剖析 trace**[2]

**通过实例理解Go Execution Tracer**[3]

**[****Golang三关-典藏版] Golang 调度器 GMP 原理与调度全分析**[4]

**Go gomaxprocs 调高引起调度性能损耗**[5]