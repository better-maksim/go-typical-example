# 线程安全的计数器

## 经典实现方式 MutexCounter

实现一个正确计数器的传统方式是使用互斥锁，保证任意时间只有一个协程操作计数器。Go 语言的话，我们可以使用 sync 包。

```go
type MutexCounter struct { 
 mu     *sync.RWMutex 
 number uint64 
} 
 
func NewMutexCounter() Counter { 
 return &MutexCounter{&sync.RWMutex{}, 0} 
} 
 
func (c *MutexCounter) Add(num uint64) { 
 c.mu.Lock() 
 defer c.mu.Unlock() 
 c.number = c.number + num 
} 
 
func (c *MutexCounter) Read() uint64 { 
 c.mu.RLock() 
 defer c.mu.RUnlock() 
 return c.number 
} 
```

## ChannelCounter

锁是一种保证同步的低级原语。Go 也提供了更高级实现方式 - channel。

我们使用 channel 来实现协程安全的计数器，使用 channel 充当队列，对计数器的操作(读、写)都缓存在队列中，按顺序操作。具体的操作通过传递 func() 实现。创建时，计数器会衍生出一个 goroutine 并且按顺序执行队列里的操作。

```golang
type ChannelCounter struct { 
 ch     chan func() 
 number uint64 
} 
 
func NewChannelCounter() Counter { 
 counter := &ChannelCounter{make(chan func(), 100), 0} 
 go func(counter *ChannelCounter) { 
  for f := range counter.ch { 
   f() 
  } 
 }(counter) 
 return counter 
} 
```

当一个协程调用 Add()，就往队列里面添加一个写操作：

```go
func (c *ChannelCounter) Add(num uint64) { 
 c.ch <- func() { 
  c.number = c.number + num 
 } 
} 
```

当一个协程调用 Read()，就往队列里面添加一个读操作：

```go
func (c *ChannelCounter) Read() uint64 { 
 ret := make(chan uint64) 
 c.ch <- func() { 
  ret <- c.number 
  close(ret) 
 } 
 return <-ret 
} 
```

## 原子方式

我们甚至可以用更低级别的原语，利用 sync/atomic 包执行原子操作。

```go
type AtomicCounter struct { 
 number uint64 
} 
 
func NewAtomicCounter() Counter { 
 return &AtomicCounter{0} 
} 
 
func (c *AtomicCounter) Add(num uint64) { 
 atomic.AddUint64(&c.number, num) 
} 
 
func (c *AtomicCounter) Read() uint64 { 
 return atomic.LoadUint64(&c.number) 
} 

```

## 比较和交换

我们可以使用非常经典的原语：CAS，对计时器进行计数。

```go

type CASCounter struct { 
 number uint64 
} 

func (c *CASCounter) Add(num uint64) { 
 for { 
  v := atomic.LoadUint64(&c.number) 
  if atomic.CompareAndSwapUint64(&c.number, v, v+num) { 
   return 
  } 
 } 
} 
 
func (c *CASCounter) Read() uint64 { 
 return atomic.LoadUint64(&c.number) 
}

```

代码地址：https://github.com/better-maksim/counter

##  性能测试

TODO

## ref 

1. [Go 语言实现安全计数的若干种方法](https://blog.csdn.net/asd1126163471/article/details/119122198)
2. [golang 做个 Mutex 与 atomic 性能测试](https://www.cnblogs.com/jackey2015/p/11737570.html)