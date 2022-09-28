## 简介

sync.Once是go语言中，原子操作的一个很重要的结构

## 实现一个单例模式

在go语言中要实现一个单例模式，代码很简单，如下

```go
package singleton

import "sync"

type Singleton struct {}

var singleton *Singleton
var once sync.Once

func GetInstance() *Singleton {
  once.Do(func(){
    singleton = &Singleton{}
  })
  return singleton
}

```

## 源码解析

### sync.Once结构体
```go
type Once struct {
  done uint32
  m Mutex
}
```
Once结构体非常简单
- done 调用标识符，once对象初始化时，其done值默认为0
- m 锁，用于初始化时候竞争控制，当调用once的Do方法时候，会通过m加锁。

### once的方法

once只包含一个Do方法，其入参是一个func回调函数，其代码如下
```go
func (o *Once) Do(func(){
  if atomic.LoadUint32(&o.done) == 0 {
    o.doSlow(f)
  }
})

func (o *Once) doSlow(f func(){
  o.m.Lock()
  defer o.m.Unlock()
  if o.done == 0 {
    defer atomic.StoreUint32(&o.done,1)
    f()
  }

})
```
执行流程
- 当o.done值为0时，调用doSlow方法进行实例化
- 在doSlow方法中调用once的m属性进行加锁操作，执行f函数，通过原子操作将o.done标识符置为1
- 释放锁

## 总结
sync.Once在源码层次是很简明的，很多实际的应用场景都能使用他做一次加载，常见的场景 
- 单例模式
- 配置文件的初始化
- 任何只需要初始化一次的对象初始化