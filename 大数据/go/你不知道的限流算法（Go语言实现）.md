不依赖外部库的情况下，限流算法有什么实现的思路？本文介绍了3种实现限流的方式。



## **一、漏桶算法**

- **算法思想**

  与令牌桶是“反向”的算法，当有请求到来时先放到木桶中，worker以固定的速度从木桶中取出请求进行相应。如果木桶已经满了，直接返回请求频率超限的错误码或者页面

- **适用场景**
  流量最均匀的限流方式，一般用于流量“整形”，例如保护数据库的限流。先把对数据库的访问加入到木桶中，worker再以db能够承受的qps从木桶中取出请求，去访问数据库。不太适合电商抢购和微博出现热点事件等场景的限流，一是应对突发流量不是很灵活，二是为每个user_id/ip维护一个队列(木桶)，workder从这些队列中拉取任务，资源的消耗会比较大。

- **go语言实现**
  通常使用队列来实现，在go语言中可以通过buffered channel来快速实现，任务加入channel，开启一定数量的worker从channel中获取任务执行。

```go
package main

import (
 "fmt"
 "sync"
 "time"
)

// 每个请求来了，把需要执行的业务逻辑封装成Task，放入木桶，等待worker取出执行
type Task struct {
 handler func() Result // worker从木桶中取出请求对象后要执行的业务逻辑函数
 resChan chan Result   // 等待worker执行并返回结果的channel
 taskID  int
}

// 封装业务逻辑的执行结果
type Result struct {
}

// 模拟业务逻辑的函数
func handler() Result {
 time.Sleep(300 * time.Millisecond)
 return Result{}
}

func NewTask(id int) Task {
 return Task{
  handler: handler,
  resChan: make(chan Result),
  taskID:  id,
 }
}

// 漏桶
type LeakyBucket struct {
 BucketSize int       // 木桶的大小
 NumWorker  int       // 同时从木桶中获取任务执行的worker数量
 bucket     chan Task // 存方任务的木桶
}

func NewLeakyBucket(bucketSize int, numWorker int) *LeakyBucket {
 return &LeakyBucket{
  BucketSize: bucketSize,
  NumWorker:  numWorker,
  bucket:     make(chan Task, bucketSize),
 }
}

func (b *LeakyBucket) validate(task Task) bool {
 // 如果木桶已经满了，返回false
 select {
 case b.bucket <- task:
 default:
  fmt.Printf("request[id=%d] is refused\n", task.taskID)
  return false
 }

 // 等待worker执行
 <-task.resChan
 fmt.Printf("request[id=%d] is run\n", task.taskID)
 return true
}

func (b *LeakyBucket) Start() {
 // 开启worker从木桶拉取任务执行
 go func() {
  for i := 0; i < b.NumWorker; i++ {
   go func() {
    for {
     task := <-b.bucket
     result := task.handler()
     task.resChan <- result
    }
   }()
  }
 }()
}

func main() {
 bucket := NewLeakyBucket(10, 4)
 bucket.Start()

 var wg sync.WaitGroup
 for i := 0; i < 20; i++ {
  wg.Add(1)
  go func(id int) {
   defer wg.Done()
   task := NewTask(id)
   bucket.validate(task)
  }(i)
 }
 wg.Wait()
}

```

### **二、令牌桶算法**

- **算法思想**
  想象有一个木桶，以固定的速度往木桶里加入令牌，木桶满了则不再加入令牌。服务收到请求时尝试从木桶中取出一个令牌，如果能够得到令牌则继续执行后续的业务逻辑；如果没有得到令牌，直接返回反问频率超限的错误码或页面等，不继续执行后续的业务逻辑

- 特点：由于木桶内只要有令牌，请求就可以被处理，所以令牌桶算法可以支持突发流量。同时由于往木桶添加令牌的速度是固定的，且木桶的容量有上限，所以单位时间内处理的请求书也能够得到控制，起到限流的目的。假设加入令牌的速度为 1token/10ms，桶的容量为500，在请求比较的少的时候（小于每10毫秒1个请求）时，木桶可以先"攒"一些令牌（最多500个）。当有突发流量时，一下把木桶内的令牌取空，也就是有500个在并发执行的业务逻辑，之后要等每10ms补充一个新的令牌才能接收一个新的请求。

- 参数设置：木桶的容量 - 考虑业务逻辑的资源消耗和机器能承载并发处理多少业务逻辑。生成令牌的速度 - 太慢的话起不到“攒”令牌应对突发流量的效果。

- 适用场景：
  适合电商抢购或者微博出现热点事件这种场景，因为在限流的同时可以应对一定的突发流量。如果采用均匀速度处理请求的算法，在发生热点时间的时候，会造成大量的用户无法访问，对用户体验的损害比较大。

- go语言实现：
  假设每100ms生产一个令牌，按user_id/IP记录访问最近一次访问的时间戳 t_last 和令牌数，每次请求时如果 now - last > 100ms, 增加 (now - last) / 100ms个令牌。然后，如果令牌数 > 0，令牌数 -1 继续执行后续的业务逻辑，否则返回请求频率超限的错误码或页面。

```go
package main

import (
 "fmt"
 "sync"
 "time"
)

// 并发访问同一个user_id/ip的记录需要上锁
var recordMu map[string]*sync.RWMutex

func init() {
 recordMu = make(map[string]*sync.RWMutex)
}

func max(a, b int) int {
 if a > b {
  return a
 }
 return b
}

type TokenBucket struct {
 BucketSize int // 木桶内的容量：最多可以存放多少个令牌
 TokenRate time.Duration // 多长时间生成一个令牌
 records map[string]*record // 报错user_id/ip的访问记录
}

// 上次访问时的时间戳和令牌数
type record struct {
 last time.Time
 token int
}

func NewTokenBucket(bucketSize int, tokenRate time.Duration) *TokenBucket {
 return &TokenBucket{
  BucketSize: bucketSize,
  TokenRate:  tokenRate,
  records:    make(map[string]*record),
 }
}

func (t *TokenBucket) getUidOrIp() string {
 // 获取请求用户的user_id或者ip地址
 return "127.0.0.1"
}

// 获取这个user_id/ip上次访问时的时间戳和令牌数
func (t *TokenBucket) getRecord(uidOrIp string) *record {
 if r, ok := t.records[uidOrIp]; ok {
  return r
 }
 return &record{}
}

// 保存user_id/ip最近一次请求时的时间戳和令牌数量
func (t *TokenBucket) storeRecord(uidOrIp string, r *record) {
 t.records[uidOrIp] = r
}

// 验证是否能获取一个令牌
func (t *TokenBucket) validate(uidOrIp string) bool {
 // 并发修改同一个用户的记录上写锁
 rl, ok := recordMu[uidOrIp]
 if !ok {
  var mu sync.RWMutex
  rl = &mu
  recordMu[uidOrIp] = rl
 }
 rl.Lock()
 defer rl.Unlock()

 r := t.getRecord(uidOrIp)
 now := time.Now()
 if r.last.IsZero() {
  // 第一次访问初始化为最大令牌数
  r.last, r.token = now, t.BucketSize
 } else {
  if r.last.Add(t.TokenRate).Before(now) {
   // 如果与上次请求的间隔超过了token rate
   // 则增加令牌，更新last
   r.token += max(int(now.Sub(r.last) / t.TokenRate), t.BucketSize)
   r.last = now
  }
 }
 var result bool
 if r.token > 0 {
  // 如果令牌数大于1，取走一个令牌，validate结果为true
  r.token--
  result = true
 }

 // 保存最新的record
 t.storeRecord(uidOrIp, r)
 return result
}

// 返回是否被限流
func (t *TokenBucket) IsLimited() bool {
 return !t.validate(t.getUidOrIp())
}

func main() {
 tokenBucket := NewTokenBucket(5, 100*time.Millisecond)
 for i := 0; i< 6; i++ {
  fmt.Println(tokenBucket.IsLimited())
 }
 time.Sleep(100 * time.Millisecond)
 fmt.Println(tokenBucket.IsLimited())
}
```

### **三、滑动时间窗口算法**



- **算法思想**
  滑动时间窗口算法，是从对普通时间窗口计数的优化。
  使用普通时间窗口时，我们会为每个user_id/ip维护一个KV: uidOrIp: timestamp_requestCount。假设限制1秒1000个请求，那么第100ms有一个请求，这个KV变成 uidOrIp: timestamp_1，递200ms有1个请求，我们先比较距离记录的timestamp有没有超过1s，如果没有只更新count，此时KV变成 uidOrIp: timestamp_2。当第1100ms来一个请求时，更新记录中的timestamp并重置计数，KV变成 uidOrIp: newtimestamp_1
  普通时间窗口有一个问题，假设有500个请求集中在前1s的后100ms，500个请求集中在后1s的前100ms，其实在这200ms没就已经请求超限了，但是由于时间窗每经过1s就会重置计数，就无法识别到此时的请求超限。

  
  对于滑动时间窗口，我们可以把1ms的时间窗口划分成10个time slot, 每个time slot统计某个100ms的请求数量。每经过100ms，有一个新的time slot加入窗口，早于当前时间100ms的time slot出窗口。窗口内最多维护10个time slot，储存空间的消耗同样是比较低的。



- 适用场景
  与令牌桶一样，有应对突发流量的能力



- go语言实现
  主要就是实现sliding window算法。可以参考Bilibili开源的kratos框架里circuit breaker用循环列表保存time slot对象的实现，他们这个实现的好处是不用频繁的创建和销毁time slot对象。下面给出一个简单的基本实现：

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var winMu map[string]*sync.RWMutex

func init() {
	winMu = make(map[string]*sync.RWMutex)
}

type timeSlot struct {
	timestamp time.Time // 这个timeSlot的时间起点
	count     int       // 落在这个timeSlot内的请求数
}

func countReq(win []*timeSlot) int {
	var count int
	for _, ts := range win {
		count += ts.count
	}
	return count
}

type SlidingWindowLimiter struct {
	SlotDuration time.Duration // time slot的长度
	WinDuration  time.Duration // sliding window的长度
	numSlots     int           // window内最多有多少个slot
	windows      map[string][]*timeSlot
	maxReq       int // win duration内允许的最大请求数
}

func NewSliding(slotDuration time.Duration, winDuration time.Duration, maxReq int) *SlidingWindowLimiter {
	return &SlidingWindowLimiter{
		SlotDuration: slotDuration,
		WinDuration:  winDuration,
		numSlots:     int(winDuration / slotDuration),
		windows:      make(map[string][]*timeSlot),
		maxReq:       maxReq,
	}
}

// 获取user_id/ip的时间窗口
func (l *SlidingWindowLimiter) getWindow(uidOrIp string) []*timeSlot {
	win, ok := l.windows[uidOrIp]
	if !ok {
		win = make([]*timeSlot, 0, l.numSlots)
	}
	return win
}

func (l *SlidingWindowLimiter) storeWindow(uidOrIp string, win []*timeSlot) {
	l.windows[uidOrIp] = win
}

func (l *SlidingWindowLimiter) validate(uidOrIp string) bool {
	// 同一user_id/ip并发安全
	mu, ok := winMu[uidOrIp]
	if !ok {
		var m sync.RWMutex
		mu = &m
		winMu[uidOrIp] = mu
	}
	mu.Lock()
	defer mu.Unlock()

	win := l.getWindow(uidOrIp)
	now := time.Now()
	// 已经过期的time slot移出时间窗
	timeoutOffset := -1
	for i, ts := range win {
		if ts.timestamp.Add(l.WinDuration).After(now) {
			break
		}
		timeoutOffset = i
	}
	if timeoutOffset > -1 {
		win = win[timeoutOffset+1:]
	}

	// 判断请求是否超限
	var result bool
	if countReq(win) < l.maxReq {
		result = true
	}

	// 记录这次的请求数
	var lastSlot *timeSlot
	if len(win) > 0 {
		lastSlot = win[len(win)-1]
		if lastSlot.timestamp.Add(l.SlotDuration).Before(now) {
			lastSlot = &timeSlot{timestamp: now, count: 1}
			win = append(win, lastSlot)
		} else {
			lastSlot.count++
		}
	} else {
		lastSlot = &timeSlot{timestamp: now, count: 1}
		win = append(win, lastSlot)
	}

	l.storeWindow(uidOrIp, win)

	return result
}

func (l *SlidingWindowLimiter) getUidOrIp() string {
	return "127.0.0.1"
}

func (l *SlidingWindowLimiter) IsLimited() bool {
	return !l.validate(l.getUidOrIp())
}

func main() {
	limiter := NewSliding(100*time.Millisecond, time.Second, 10)
	for i := 0; i < 5; i++ {
		fmt.Println(limiter.IsLimited())
	}
	time.Sleep(100 * time.Millisecond)
	for i := 0; i < 5; i++ {
		fmt.Println(limiter.IsLimited())
	}
	fmt.Println(limiter.IsLimited())
	for _, v := range limiter.windows[limiter.getUidOrIp()] {
		fmt.Println(v.timestamp, v.count)
	}

	fmt.Println("a thousand years later...")
	time.Sleep(time.Second)
	for i := 0; i < 7; i++ {
		fmt.Println(limiter.IsLimited())
	}
	for _, v := range limiter.windows[limiter.getUidOrIp()] {
		fmt.Println(v.timestamp, v.count)
	}
}

```