# 前言

## go语言并发哲学

> Do not communicate by sharing memory; instead , sharing memory of communicating

**不要通过共享内存来通信，而要通过通信来实现内存共享。**

这就是Go语言的并发哲学，他依赖于CSP，基于channel实现。

## 什么是CSP

CSP全称是Communicating Sequential Processes，这是Tony Hoare在1978年发表的ACM中的一篇论文。论文中指出一门编程语言应该重视input和output的原语，尤其是并发变成的代码。

Go是第一个将CSP这些思想引入并发扬光大的语言 。 仅管理内存同步访问控制在某些情况下大有用处，Go也有相应的sync包进行支持。大多数的编程语言是基于线程和内存同步访问控制，`Go的并发的模型则是使用goroutine和channel代替`。

Go routine解放了程序员，让我们不用去考虑线程库、线程开销、线程调度等等繁琐的底层问题，goroutine天生就帮我们解决了。

本文重点是分析channel的源码层次的内容，gorutine相关的内容可以自行去翻阅资料。`本文所有源码的版本为go 1.17.5`。

## 其他说明

当前在很多公众号和博文中都能找到channel相关的源码解读，我这里重新写一篇文章来分析，为的只是自己更加深入的了解和学习，看别人的文章而不动手只能停留在概念层次，只有自己亲身经历了过程，印象才会深刻。

# channel的源码结构分析

channel的底层数据结构是由hchan来实现的。

## 代码位置

src/runtime/chan.go:hchan

## hchan的数据结构

```go
type hchan struct {
  qcount uint //当前队列中剩余元素的个数
  datasiz uint //环形链的长度
  buf unsafe.Pointer //环形队列指针
  elemsize uint16 //每个元素的大小
  closed uint32 //标识关闭状态
  elemtype *_type //元素类型
  sendx uint //已发送元素在循环数组中的索引
  recvx uint //已接收元素在循环数组中的索引
  recq waitq //等待接收的goroutine队列
  sendq waitq //等待发送的goroutine队列
  lock mutex //锁
}
```

相关的字段说明已经在定义中添加了注释说明，需要重点说的是如下字段

- buf ：指向底层循环数组，只有缓冲型channel才有。
- elemtype：代表元素类型，用于数据传递过程的中赋值
- elemsize：元素大小，用于在buf中定位元素出现的位置
- sendx、recvx：均指向底层循环数组，表示当前可以发送和接收的元素位置索引。
- sendq、recvq：分别表示被阻塞的goroutine，这些goroutine由于尝试读取channel中的数据或者向channel发送数据而被阻塞
- lock 用于保证每个读channel和写channel的操作是原子的。

waitq是sudog类型的双向链表，而sudog实际是对goroutine的封装，其结构如下

```go
type waitq struct {
	first *sudog
	last *sudog
}
```

## channel的操作

channel的操作分为创建、发送、接收、关闭等4个操作，下面就从源码和流程方面分开展示

### 创建chan

通道一般有两个方向，发送和接收。理论上，我们可以创建一个只发送或者只接收。在go中通过make来创建channel，代码如下

```go
//无缓冲的通道
ch1 := make(chan int)
//有缓冲的通道
ch2 := make(chan int , 1)
```

在底层，channel的创建是由makechan函数来执行的，函数定义如下

```go
func makechan(t *chantype, size int64) *hchan
```

入参包含两个

- t  ：类型，对应chantype类型的数据结构

- size：大小，对应是否为有缓冲还是无缓冲的channel，值大于零表示是有缓冲的channel，等于0表示无缓冲的channel

  

#### 创建chan的过程

1. 检查元素的大小，hchan的size以及align等相关信息

2. 判断内存大小是否为0，若为0，则为chan分配内存空间，并将chan的buf地址进行初始化。

3. 若元素中不包含指针，则调用mallocgc函数进行内存分配，并将chan的buf指向新的地址，新的地址是根据hchan的当前地址加上hchansize的值最后得到的。

4. 若元素中包含指针，则调用new函数来创建hchan指针，并为chan的buf分配内存。

   

#### makechan函数代码

`````go
//相关的常量定义
const (
	maxAlign  = 8
	hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
	debugChan = false
)

func makechan(t *chantype, size uint64) *hchan {
  elem := t.elem

	// 检测channel的大小是否已经超过限制，否则抛出异常
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
  //检测元素的align是否超出最大的align，否则抛出异常
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}
  //检测溢出情况，否则抛出异常
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

  // 当存储在buf中的元素不包含指针时候，hchan不包含指针。
  //buf指向相同的内存空间，elemtype是持久的
  // Sudog是从从他们自己的线程中引用的，所以不能被收集。
	var c *hchan
	switch {
	case mem == 0:
		// 如果队列或者chan size大小为0，则调用mallocgc函数为chan分配内存空间
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
    // 元素中不包含指针，在一次调用中为buf和和hchan分配内存
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 元素中包含指针，则使用new关键值来分配内存
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}



//chan的raceaddr方法代码如下
func (c *hchan) raceaddr() unsafe.Pointer() {
  return unsafe.Pointer(&c.buf)
}
`````

> chan中所有分配内存的函数都是有mallcogc函数来完成的，其源代码在src/runtime/malloc.go:mallocgc中，感兴趣的可以去查看下。



### 往channel中发送数据

往channel的发送数据由chansend1和chansend两个方法完成，chansend1底层也是直接调用chansend方法完成发送的

#### chansend函数源代码

```go
// entry point for c <- x from compiled code
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
  //如果channel为nil
	if c == nil {
    //不阻塞，直接返回false，表示没有发送成功
		if !block {
			return false
		}
    //将当前goroutine挂起，并抛出unreachable异常
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

  //debug模式下打印chan相关的信息
	if debugChan {
		print("chansend: chan=", c, "\n")
	}
  
  //race相关的信息
	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}
	
  //对于非阻塞的 并且chan的closed标记为0 并且chan的队列已经满了，直接返回false
	if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

  //锁住chan，实现并发安全
	lock(&c.lock)

  //如果chan已经被关闭，则释放锁，并进行panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

  //如果接受队列中有goroutine，直接将要发送的数据拷贝到接收的goroutine中
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

  //对于缓冲型的channel，如果还有缓冲空间
	if c.qcount < c.dataqsiz {
    //知道发送的元素的在buf中的位置
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
    //将数据从ep拷贝到qp
		typedmemmove(c.elemtype, qp, ep)
    //发送索引加一
		c.sendx++
    //如果发送索引等于唤醒链表的长度，则将游标贵0
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
    //缓冲区元素加1，并释放锁
		c.qcount++
		unlock(&c.lock)
		return true
	}

  //如果不需要阻塞，则直接返回 ，并释放锁
	if !block {
		unlock(&c.lock)
		return false
	}

	// 获取当前goroutine的指针，
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
  //当前goroutine进入的发送等待队列
	c.sendq.enqueue(mysg)
  //通知其他goroutine，并将goroutine挂起
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	
  //在没有其他的接收队列将数据复制到队列中时候，需要保证当前需要被发送的的值一直是可用状态
  // sudog 有一个指向堆栈对象的指针，但是sudog不是堆栈追踪器的根
	KeepAlive(ep)


	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
  //更新goroutine相关的对象信息
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
  //释放sudog对象
	releaseSudog(mysg)
  //如果channel已经关闭
	if closed { 
    // close标志位为0，则抛出假性唤醒异常
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
    //直接panic
		panic(plainError("send on closed channel"))
	}
	return true
}
```

#### 发送操作简单流程

1、如果channel是一个空值，在非阻塞模式下，会直接返回。在阻塞模式下，会调用gopark函数挂起goroutine；在非阻塞模式下快速检测到失败并进行返回

2、当channel没准备好接收时（非缓冲型，等待队列中没有goroutine在等待；缓冲型，buf中没有元素），如果channel已经被关闭，直接返回false；否则，加锁，如果channel已经关闭，并且循环数组buf中没有元素，对应非缓冲型关闭和缓冲型关闭但是buf无元素时，返回对应类型的零值，但是received标志为false，告诉调用者此channel已经关闭，取出来的值并不是正常发送者发送来的数据。

3、如果等待接收队列recvq不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从recvq取出G,并把数据写入，最后把该G唤醒，结束发送过程；

4、如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程；

5、如果缓冲区中没有空余位置，将待发送数据写入G，将当前G加入sendq，进入睡眠，等待被读goroutine唤醒；

#### 简单发送的流程图

![image-20220912131308100](/Users/admin/Library/Application Support/typora-user-images/image-20220912131308100.png)

#### channel发送数据示例

````go
//线程1接收数据
func goroutineA(vChanA <- chan int) {
  val := <- vChanA
  fmt.Println("G1 received data :", val)
  return
}

//线程2接收数据
func goroutineB(vChanB <- chan int) {
  val := <- vChanA
  fmt.Println("G2 received data :", val)
  return
}

func main() {
  sendChan := make(chan int)
  go goroutineA()
  go goroutineB()
  //发送数据
  sendChan <- 1
  time.Sleep(time.Second)
}
````

### 从channel中接收数据

channel接收数据是有chanrecv1和chanrecv2两个函数完成，底层调用的是chanrecv函数。

#### chanrecv函数源代码

```go

// entry points for <- c from compiled code
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

//chanrecv从channel c中接收数据，并将接收到的数据写入到ep中
//当ep为空时，接收的数据将会被忽略
//如果goroutine是非阻塞，切元素为空，直接return false
// 否则如果c已经关闭，ep被复制为0，并返回false
// 否则将数据填充导ep指针中，并返回true
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// 如果是非阻塞操作，并且c为空
	if !block && empty(c) {
    //观察到channel没有做好接收的准备，则观察channel是否已经报备关闭
		//原子加载通道如果关闭则返回，因为通道不能重新打开，后来观察到的通道没有被关闭，就意味着第一次观察的
    //时候也没有被关闭，则表示接收操作不能继续往下，直接return
		if atomic.Load(&c.closed) == 0 {
			return
		}
    //通道是不可逆转的关闭，重新检查是否有待接收的数据，这些数据数据可能是在上面的空和关闭检查之前到达的
    // 在这种发送场景下，我们需要保持发送的顺序一致。
		if empty(c) {
			// 通过通道是不可逆的且是关闭的
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

  //加锁
	lock(&c.lock)
	//channel已经关闭，并且循环数组buf中没有元素，
  //这里可以处理非缓冲型关闭和缓冲型关闭但是buf无元素的情况
  //也就是说即使是关闭状态，但在缓冲型channel，buf有元素的情况下还能接收元素
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
    //解锁
		unlock(&c.lock)
		if ep != nil {
      //从一个已经关闭的channel中执行接收操作，且未忽略返回值
      //那么接收的值是该类型的一个零值
      //typedmemclr 根据类型清理相应地址的内存。
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

  //等待发送的队列有goroutine存在，说明buf是满的
  //这可能是
  // 1、非缓冲型的channel
  // 2、缓冲型的channel，但是buf已经满了
  //针对1，直接进行内存拷贝（从sender goroutine -> receiver goroutine）
  //针对2，接收到循环数组的头部元素，并将发送者的元素放到循环数组尾部
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

  //缓冲型，buf中有元素，可以正常接收数据
	if c.qcount > 0 {
		// 直接从buf中寻找导对应的元素
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
    // 清理掉循环数组里相应位置的值
		typedmemclr(c.elemtype, qp)
    //接收游标向前移动
		c.recvx++
    //如果接收游标 等于环形链表的值，则接收游标清零。
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
    //循环数组buf数量减一
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
    // 非阻塞接收，解锁。selected 返回 false，因为没有接收到值
		unlock(&c.lock)
		return false, false
	}

  //被阻塞的情况，直接构造一个sudog
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// 待接收的地址被保存下来
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
  //进入channel的等待接收队列中
	c.recvq.enqueue(mysg)
  //向任何试图缩减堆栈的对象发送信号，表明过程即将停在一个通道上。
	atomic.Store8(&gp.parkingOnChan, 1)
  //将goroutine挂起
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// 被唤醒了，接着从这里继续执行一些扫尾工作
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```

#### 数据读取的简单过程

1. 如果等待发送的队列sendq不为空，且没有缓冲区，直接从sendq中取出G，把G中数据读出，最后把G唤醒，结束读取过程
2. 如果等待发送队列sendq不为空，此时说明缓冲区已满，从缓冲区首部读出数据，把G中数据写入缓冲区尾部，把G唤醒，结束读取过程
3. 如果缓冲区中有数据，则从缓冲区中取出数据，结束读取过程
4. 将当前goroutine加入recvq，进入睡眠，等待被写goroutine唤醒。

#### 数据读取简单过程的流程图

![image-20220912140041707](/Users/admin/Library/Application Support/typora-user-images/image-20220912140041707.png)



### 关闭channel

channel的关闭由closechan函数完成

#### closechan源码

````go

func closechan(c *hchan) {
  //关闭一个nil的channel直接panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}
	
  //加锁
	lock(&c.lock)
  //如果channel已经被关闭，直接panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

 	//修改为已经关闭
	c.closed = 1

	var glist gList

	// 将 channel 所有等待接收队列的里 sudog 释放
	for {
    //接收队列中出一个sudog
		sg := c.recvq.dequeue()
    //出队完成，跳出循环
		if sg == nil {
			break
		}
    //如果元素不为空，说明此receiver未忽略接收数据
    //给它赋一个相应类型的0值
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
    //取出goroutine
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
    //推入导链表中
		glist.push(gp)
	}

	// 将channel中等待接收队列里的sudog释放
  //如果存在这些goroutine将会panic
	for {
    //从发送队列中出一个sudog
		sg := c.sendq.dequeue()
    //出队完毕，直接跳出循环
		if sg == nil {
			break
		}
    //发送者panic
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
    //推入链表中
		glist.push(gp)
	}
  //释放锁
	unlock(&c.lock)

  //遍历链表
	for !glist.empty() {
    //取最后一个
		gp := glist.pop()
		gp.schedlink = 0
    //唤醒goroutine
		goready(gp, 3)
	}
}

````

#### 关闭channel总结

关闭channel会把recvq中的G全部唤醒，本该写入G的数据位置为nil。把sendq中的G全部唤醒，但这些G会panic

除此之外，panic出现的场景还有

1. 关闭值为nil的channel
2. 关闭已经被关闭的channel
3. 想已经关闭的channel写数据

#### 使用channel的几点不方便的地方

1、在不改变channel自身的情况下，无法获知一个channel是否关闭

2、关闭一个close channel会导致panic。所以，如果关闭channel的一方在不知道channel是否处于关闭状态是去默然关闭channel是很危险的事情。

3、向一个close的channel发送数据会panic。所以，如果向channel发送数据的一方不知道channel是否处于关闭状态就贸然向channel发送数据是很危险的事情。

#### 关闭channel示例

```go
func IsChanClosed(ch <- chan T) bool {
  select {
    case <- ch :
      return true
    default:
    
  }
  return false
}

func main() {
  c := make(chan T)
  //打印得到false
  fmt.Println(IsChanClosed(c))
  close(c)
  //打印得到true
  fmt.Println(IsChanClosed(c))
}
```

本示例只是一个粗糙的示例，返回的结果仅仅代表调用的那个瞬间，并不能保证后续调用会不会有其他goroutine改变其状态。

而如何优雅的进行关闭呢 ？答案是只需要增加一个传递关闭信号的channel，receiver通过信号channel下达关闭channel指令。sender监听到关闭信号后，停止发送数据。代码如下

```go
func main() {
  rand.Seed(time.Now().UnixNano())
  const Max = 10000
  const senderNums = 1000
  dataCh := make(chan int, 100)
  stopChn := make(chan struct{})
  
  //发送数据
  for i := 0;i<senderNums;i++  {
    go func(){
      for {
        select {
          case <- stopChan:
          	return
          case dataCh <- rand.Intn(Max)
        }
      }
    }
  }
  
  //接收数据
  go func() {
    for value := range dataCh {
      
      if value == Max - 1 {
        fmt.Println("send stop signal to senders.")
        close(stopCh)
        return
      }
      fmt.Println(value)
    }
  }()
  
  //执行等待一个小时
  select {
    case <- time.After(time.Hour)
  }
  
}
```







### 操作channel总结

| 操作     | nil channel | closed channel   | 正常的channel                                                |
| -------- | ----------- | ---------------- | ------------------------------------------------------------ |
| close    | panic       | panic            | 正常关闭                                                     |
| 读 <-ch  | 阻塞        | 读对应类型的零值 | 阻塞或者正常读数据。缓冲型为空或非缓冲型没有等待发送者时候为空 |
| 写 ch <- | 阻塞        | panic            | 阻塞或者正常写入。非缓冲型没有等待接受者或者缓冲型buf满时会阻塞。 |

## channel的使用场景

### 1、停止信号

channel 用于停止信号的场景还是挺多的，经常是关闭某个 channel 或者向 channel 发送一个 元素，使得接收 channel 的那一方获知道此信息，进而做一些其他的操作。

### 2、任务定时

与 timer 结合，一般有两种玩法:实现超时控制，实现定期执行某个任务。有时候，需要执行某项操作，但又不想它耗费太长时间，上一个定时器就可以搞定:

```go
select {
  case <- time.After(100 * time.Millisecond):
	case <- s.stopc :
  	return fase
}
```

定时执行某个任务也简单，如下是每隔1s执行任务示例

````go
func worker() {
	ticker := time.Tick(1 * time.Second)
	for {
		select {
			case <- ticker :
				fmt.Println("执行1s定时任务")
		}
	}
}
````

### 3、解耦生产方和消费方

服务启动时，启动n个worker作为工作协程池，这些协程池工作在一个for循环里，从某个channel消费并执行。如下例子所示，10个协程不断的从工作队列中取任务，生产方只管往channel中发送任务，解耦了生产方和消费方。

```go
func main() {
  taskCh := make(chan int, 100)
  go worker(taskCh)
  
  //阻塞任务
  for i := 0;i<100;i++ {
    taskCh <- i
  }
  
  //等待一个小时
  select {
    case <- time.After(time.Hour):
  }
}

func worker(taskCh <- chan int) {
  const N = 10
  for i:=0;i<N;i++ {
    go func(index int){
      for {
        task := <-taskCh
        fmt.Println("task : %d is done by worker :%d \n", task, index)
      }
    }(i)
  }
}
```

