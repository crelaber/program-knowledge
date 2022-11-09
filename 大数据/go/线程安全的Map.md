我记得在面试 Java 岗位的时候，经常被问到的一个问题就是：HashMap 是否线程安全。如何实现一个线程安全的哈希表。

在 Go 语言中我想应该也逃不过这个问题，毕竟 Go 也提供了哈希表的实现 -- map。当然先说结论。map 非线程安全，那么如何实现一个线程安全的哈希表就是这篇学习笔记要讨论的一个问题。

## 一、map 的基本使用

map 的类型是 map[K]V 其中 K 和 V 都是散列表的键和值对应的数据类型。一个给定类型的 map 的 K 和 V 都是确定且统一的。

当然 K 和 V 可以不是同一种类型，在 Golang 中 V 的类型并没有限制，但是只有可以通过操作符 == 进行比较的数据类型才可以作为 K。

关于 map 的详细使用，可以参看我的这篇文章[Golang 学习笔记 06 -- 映射 map](http://mp.weixin.qq.com/s?__biz=MzU3MTg1MjYzNQ==&mid=2247484447&idx=1&sn=aa99f47778422a5a268b0b04e0045e92&chksm=fcd8962bcbaf1f3dadbe9383649f7e769fff9130f8e17d269e7f09dcd901fc7e27b8cb4cff2f&scene=21#wechat_redirect)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Rw0vaSVgVvJfJPLcOPlLEChwrYODc6LNGgnWl0iciaxw5dn8Ef4ViaIZDBicaaBKkYK5icvfVFpEJuZ5jtf8gPrTpAg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



##   二、map 的错误使用场景

在日常工作中，我遇到过两种关于 map 使用的错误：未初始化 和 并发使用。好在有写单元测试的习惯，所以这两种错误往往都在单元测试的时候爆出来的。

### 未初始化

未初始化就使用 map 的错误往往是发生在使用 var 关键字进行 map 变量声明的时候，如果一开始没有进行初始化，那么在函数里就很容易忘了对其进行初始化。而对一个未初始化的 map 的取值操作是不会产生 panic 的，只会返回对应类型的零值，但是插入元素的操作则会产生空指针的 panic

忘记初始化比较常见使用场景有：

> 1. map 是 struct 的一个成员，在初始化 struct 对象时忘记对 map 成员进行初始化
> 2. 使用 var 关键字 声明了一个全局或者局部的 map，但只是声明了变量，未进行赋值

```go
type MapDemo struct {
  dict   map[string]int
  keys   []string
  values []int
  count  int
}

func InitMapDemo() *MapDemo {
  var dict map[string]int
  return &MapDemo{
    keys:   make([]string, 0),
    values: make([]int, 0),
    dict: dict,
  }
}

func (md *MapDemo) Insert(key string, value int) bool {
  if oldValue, ok := md.dict[key]; ok { // 不会抛出异常，进入到 else 分支打印 key 和 int 的零值
    fmt.Printf("key: %s exists, old value is: %d\n", key, oldValue)
    if oldValue == value {
      return false
    }
    md.values[md.dict[key]] = value
    return true
  } else {
    fmt.Printf("key: %s not exists, value is: %d\n", key, oldValue)
  }
  md.keys = append(md.keys, key)
  md.values = append(md.values, value)
  md.dict[key] = len(md.values) //panic：assignment to entry in nil map
  return true
}
```



### 并发读写

map 非线程安全，读写操作在运行时会进行检查，类似 Java 中的 fail fast 机制。所以不加任何并发控制的读写操作是会导致 panic。

```go
func CurrentRWDemo() {
  var wg sync.WaitGroup
  wg.Add(2)
  var dict = make(map[string]int, 0)
  go func(wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 100; i++ {
      key := fmt.Sprintf("key_%d", i)
      dict[key] = i // fatal error: concurrent map read and map write
    }
  }(&wg)

  go func(wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 100; i++ {
      key := fmt.Sprintf("key_%d", i)
      fmt.Printf("key: %s, value: %d\n", key, dict[key]) // fatal error: concurrent map read and map write
    }
  }(&wg)

  wg.Wait()
}
```

###   

## 三、实现线程安全的 map

### 1、使用读写锁

在 Go 1.18 版本之后提供了泛型的支持，所以可以使用 Go1.18 版本实现一个通用的支持泛型的加锁 map。而 1.18 版本之前，则可以使用 interface{} 来模拟泛型。一个加锁的 map 其实很简单，将 map 和 读写锁放到同一个 struct 中组合起来即可

```go
type RWMap[K comparable, V any] struct {
  sync.RWMutex
  m map[K]V
}

func NewRWMap[K comparable, V any](n int) *RWMap[K, V] {
  return &RWMap[K, V]{
    m: make(map[K]V, n),
  }
}

func (rm *RWMap[K, V]) Get(key K) (V, bool) {
  rm.RLock()
  defer rm.RUnlock()
  value, ok := rm.m[key]
  return value, ok
}

func (rm *RWMap[K, V]) Set(key K, value V) {
  rm.Lock()
  defer rm.Unlock()
  rm.m[key] = value
}

func (rm *RWMap[K, V]) Delete(key K) {
  rm.Lock()
  defer rm.Unlock()
  delete(rm.m, key)
}
```

### 2、分段锁

在方案一中，所有的读写操作都使用的是一个读写锁，但是在高并发下的大量读写操作，性能就会因为读写锁而下降，所以要尽量减少锁的使用。这就有两种策略：

> 1. 优化业务逻辑，减少锁的持有时间
> 2. 分片，使用多个锁，每个锁控制一个分片。

在 Go 中已经有了开源的分片并发 map 的轮子：orcaman/concurrent-map ( https://github.com/orcaman/concurrent-map )，所以在工作中其实没必要再去实现一个。我按照这个 concurrent-map 的设计，对方案一中的 map 进行改造

#### 定义

由于 ConcurrentMap 的核心在于对 Key 做 Hash，再对桶数取模得到对应的桶，而支持 comparable 参数类型的 hash 函数还没有找到，所以这里我暂时使用了 Hasher 这个接口，接口只有一个方法 GetHash，入参为 comparable 类型，也就是说要使用泛型的 ConcurrentMap，需要自己定义来控制 key 应该属于哪个分桶。

GetShard 则是根据对 key 调用 GetHash 方法后获得的 hash，对 SHARD_COUNT 取模确定 key 应该处于的分桶编号，然后根据分桶编号获取对应的 RWMap

```go

var SHARD_COUNT = 32

type Hasher[K comparable] interface {
  GetHash(key K) uint32
}

type RWMap[K comparable, V any] struct {
  sync.RWMutex
  m map[K]V
}

type ConcurrentMap[K comparable, V any] struct {
  shards []*RWMap[K, V]
  Hasher[K]
}

func NewConcurrentMap[K comparable, V any](hash Hasher[K]) ConcurrentMap[K, V]{
  cm := ConcurrentMap[K, V] {
    shards: make([]*RWMap[K, V], SHARD_COUNT),
    Hasher: hash,
  }
  for i := 0; i < SHARD_COUNT; i++ {
    cm.shards[i] = NewRWMap[K, V](8)
  }
  return cm
}

// GetShard get key's hash by calling GetHash(key), get shards index by modulo operation.
func (cm *ConcurrentMap[K, V]) GetShard(key K) *RWMap[K, V] {
  return cm.shards[uint(cm.GetHash(key)) % uint(SHARD_COUNT)]
}
```

#### 增删改查

增删改查的操作就比较简单了，通过调用 GetShard 方法获得 key 对应的分桶，对该桶进行增删改查即可

```go

func (cm *ConcurrentMap[K, V]) Set(key K, value V) {
  shard := cm.GetShard(key)
  shard.Lock()
  defer shard.Unlock()
  shard.m[key] = value
}

func (cm *ConcurrentMap[K, V]) Get(key K) (V, bool){
  shard := cm.GetShard(key)
  shard.RLock()
  defer shard.RUnlock()
  value, ok := shard.m[key]
  return value, ok
}

func (cm *ConcurrentMap[K, V]) Delete(key K) {
  shard := cm.GetShard(key)
  shard.Lock()
  defer shard.Unlock()
  delete(shard.m, key)
}
```

### 3、sync.Map

Go 1.9 版本之后增加了一个线程安全的 map，即 sync.Map，不过需要强调的是，sync.Map 只能用在一些特殊的场景。在 Go 的官方文档中给出了两种使用 sync.Map 的场景。原文如下

```
The Map type is optimized for two common use cases: (1) when the entry for a given key is only ever written once but read many times, as in caches that only grow, or (2) when multiple goroutines read, write, and overwrite entries for disjoint sets of keys. In these two cases, use of a Map may significantly reduce lock contention compared to a Go map paired with a separate Mutex or RWMutex.
```

翻译一下: 推荐 sync.Map 在以下两种场景下使用：

> 1. 用于只会增长的缓存系统中，一个 key 只会写入一次而被读取多次
> 2. 多个 G 会对不相交的键集合进行读、写、重写操作

可以看到 sync.Map 的使用场景确实比较少，其实官方也并不是特别推荐使用 sync.Map。在官方文档中也指出了是否使用 sync.Map 取决于性能测试，也就是对 sync.Map 和 加锁的 Map 进行性能测试，以此来决定使用哪种。尽管如此，了解 sync.Map 的内部结构还是挺有必要的。

#### 基础结构

```go
type Map struct {
  mu Mutex

  // 可以将 read 看成一个支持并发访问的 map，其包含的元素都是通过原子操作更新的
  read atomic.Value // readOnly

  // 包含需要加锁才可以访问的元素
  dirty map[any]*entry

  // 用来统计read 被穿透的次数（也就是需要查询dirty 次数）一旦 misses 的数值等于 dirty 的长度，则将 dirty 提升为 read，并将 misses 重置为0
  misses int
}

// readOnly 是原子操作存储的结构体，不可变
type readOnly struct {
  m       map[any]*entry
  amended bool // true if the dirty map contains some key not in m.
}

// expunged 是一个指针，用于标识从 dirty 中删除的键值对。Map中的删除是软删除，会标记为 expunged，等到特定的时候才会真的删除
var expunged = unsafe.Pointer(new(any))

// 代表 map 中的一个值
type entry struct {
  p unsafe.Pointer
}
```



如果 dirty 不为 nil，那么 Map 中的 read 和 dirty 会包含相同的非 expunged 的项，由于 read 和 dirty 中值保存的是指针，所以即使 read 中修改了值，dirty 中也能读取到修改后的内容

使用两个 map 的好处在于一旦 misses 的数值等于 dirty 的长度，可以立刻将 dirty 设置为 read 对象，缺点就在于创建新的 dirty 后，需要逐条遍历 read，将非 expunged 的项复制到 dirty 中

#### 增删改查

接下去我们重点看一下 sync.Map 的增删改查的内部实现，也就是 Store、Load、Delete 三个方法。这三个核心方法第一阶段都是先读取 read，因为读取 read 不需要加锁。

#### Store 源码解析

> 1. 第一阶段 Map 会先根据 key 查询 read，是否存在这个键值对，如果存在，则直接调用 tryStore 更新值，如果 key 对应的值对已经被标记为删除，那么就执行后续操作，否则利用 CAS 操作更新 entry 的指针

```go

func (m *Map) Store(key, value any) {
  read, _ := m.read.Load().(readOnly)
  if e, ok := read.m[key]; ok && e.tryStore(&value) {
    return
  }
}
func (e *entry) tryStore(i *any) bool {
  for {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
      return false
    }
    if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
      return true
    }
  }
}
```



1. 第二阶段：先上锁。分为三种情况：

2. 1. read 中已经存在，但是 value 被标记为删除。调用 unexpungeLocked 将标记去除，然后添加到 dirty 中，最后更新 read
   2. dirty 中存在的，直接更新 read（因为保存的是指针，所以更新 read，dirty 也能读到更新后的值）
   3. 都不存在，完全新加的键值对。此时如果 read 和 dirty 中的键值对完全一致，dirty 中不存在 read 中没有的 key，那么会判断 dirty 是否为 nil，如果是 nil，则会创建 dirty，并遍历 read，将非 expunge 的键值对复制到 dirty 中。最后将当前新加的键值对添加到 dirty 中

```go
func (m *Map) Store(key, value any) {
  ...

  m.mu.Lock()
  read, _ = m.read.Load().(readOnly)
  if e, ok := read.m[key]; ok {
    if e.unexpungeLocked() {
      m.dirty[key] = e
    }
    e.storeLocked(&value)
  } else if e, ok := m.dirty[key]; ok {
    e.storeLocked(&value)
  } else {
    if !read.amended {
      m.dirtyLocked()
      m.read.Store(readOnly{m: read.m, amended: true})
    }
    m.dirty[key] = newEntry(value)
  }
  m.mu.Unlock()
}
  

func (m *Map) dirtyLocked() {
  if m.dirty != nil {
    return
  }

  read, _ := m.read.Load().(readOnly)
  m.dirty = make(map[any]*entry, len(read.m))
  for k, e := range read.m {
    if !e.tryExpungeLocked() { // 遍历 read，将非 expunge 的添加到 dirty 中
      m.dirty[k] = e
    }
  }
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
  p := atomic.LoadPointer(&e.p)
  for p == nil {
    if atomic.CompareAndSwapPointer(&e.p, nil, expunged) { // CAS 操作，将 e 中指针已经是 nil的标记为 expunged
      return true
    }
    p = atomic.LoadPointer(&e.p)
  }
  return p == expunged
}
```

#### Delete 源码解析

> 1. Delete 的第一阶段也是先根据 key 查询 read 中是否存在这个键值对，如果不存在，并且 dirty 中有 read 中不存在的 key，此时会上锁，再检查一遍 read，依旧不存在的情况下，则去查询 dirty，并调用 delete 将 dirty 中这个 key 删除。然后对 misses 加 1，
>
> 2. 第二阶段，对 entry 进行删除，实际上就是利用 CAS 操作将 entry 的指针设置为 nil。如果 entry 中指针不为 nil 或者 没有标记为 expunged，那么还会返回删除 key 对对应的值
>
>    ```go
>    
>    func (m *Map) Delete(key any) {
>      m.LoadAndDelete(key)
>    }
>    func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
>      read, _ := m.read.Load().(readOnly)
>      e, ok := read.m[key]
>      if !ok && read.amended {
>        m.mu.Lock()
>        read, _ = m.read.Load().(readOnly)
>        e, ok = read.m[key]
>        if !ok && read.amended { //双重检测加锁
>          e, ok = m.dirty[key]
>          delete(m.dirty, key)
>          m.missLocked()
>        }
>        m.mu.Unlock()
>      }
>      if ok {
>        return e.delete()
>      }
>      return nil, false
>    }
>    
>    func (m *Map) missLocked() {
>      m.misses++
>      if m.misses < len(m.dirty) {
>        return
>      }
>      m.read.Store(readOnly{m: m.dirty})
>      m.dirty = nil
>      m.misses = 0
>    }
>    
>    func (e *entry) delete() (value any, ok bool) {
>      for {
>        p := atomic.LoadPointer(&e.p)
>        if p == nil || p == expunged {
>          return nil, false
>        }
>        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
>          return *(*any)(p), true
>        }
>      }
>    }
>    ```
>
>    

#### Load 源码解析

Load 也就是所谓的查询，也分为两个阶段。

> 1. 第一阶段先查询 read 字段。如果经过双重检测加锁后，read 中依旧不存在这个 key，则会去 dirty 中进行查找，不论 dirty 中是否存在，都将 miss 数加 1
> 2. 第二阶段返回读取的对象，也就是 e.load 的方法，通过源码我们也能看到 e 可能来自 read 也可能来自 dirty，不过来自哪并不重要，毕竟 e 保存的是一个指针

```go
func (m *Map) Load(key any) (value any, ok bool) {
  read, _ := m.read.Load().(readOnly)
  e, ok := read.m[key]
  if !ok && read.amended {
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    e, ok = read.m[key]
    if !ok && read.amended { // 双重检测加锁
      e, ok = m.dirty[key]
      m.missLocked()
    }
    m.mu.Unlock()
  }
  if !ok {
    return nil, false
  }
  return e.load()
}

func (e *entry) load() (value any, ok bool) {
  p := atomic.LoadPointer(&e.p)
  if p == nil || p == expunged {
    return nil, false
  }
  return *(*any)(p), true
}
```



## 简单总结

![图片](https://mmbiz.qpic.cn/mmbiz_png/Rw0vaSVgVvJfJPLcOPlLEChwrYODc6LNq7CDOlEkUE9qRDwoibX18jdFnSboBVYo6J4Id2dxJM4bmKI38ANbIPA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)