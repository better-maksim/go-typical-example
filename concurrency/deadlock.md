# 使用信号量解决死锁问题

![堵车](./images/3539174dd0d9c0d5c0e3e46e30ea7b17.jpeg)

## 死锁的四个条件与解决方案

**禁止抢占（no preemption）：** 系统资源不能被强制从一个线程中退出，计算机资源如果不是主动释放而是被抢夺有可能出现意想不到的现象。

**持有和等待（hold and wait）：** 一个线程在等待时持有并发资源。持有并发资源并还等待其它资源，也就是吃着碗里的望着锅里的，而等待的资源被其他线程持有。

**互斥（mutual exclusion）：** 资源只能同时分配给一个线程，无法多个线程共享。资源具有排他性。

**循环等待（circular waiting）：** 一系列线程互相持有其他进程所需要的资源。必须有一个循环依赖的关系。

死锁只有在四个条件同时满足时发生，那么预防死锁必须至少破坏其中一项，就可以解决死锁问题了。

## 模拟死锁

我们使用经典的哲学家问题来演示死锁的产生。

> 在1971年，著名的计算机科学家艾兹格·迪科斯彻(Edsger Dijkstra)提出了一个同步问题，即假设有五台计算机都试图访问五份共享的磁带驱动器。稍后，这个问题被托尼·霍尔(Tony Hoare)重新表述为哲学家就餐问题。这个问题可以用来解释死锁和资源耗尽。

在这里我们模拟孔子、墨子、老子、孙子、老子五人在一个房间内修炼冥想，在房间内一共有五只筷子，每个人必须同时获得两双筷子才可以进行吃饭的场景来模拟死锁情况的出现。


```go
package main

import (
	"fmt"
	"github.com/fatih/color"
	"math/rand"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// Chopstick 代表筷子
type Chopstick struct {
	sync.Mutex
}

// Philosopher 代表哲学家
type Philosopher struct {
	//哲学家的名字
	name string
	//筷子
	leftChopstick, rightChopstick *Chopstick
	//状态
	status string
}

// 需求之的进餐和冥想
// 吃完睡（冥想） 睡完吃
// 可以调整时间来增加或者减少抢夺筷子的机会
func (p *Philosopher) dine() {
	for {
		mark(p, "冥想")
		randomPause(10)
		mark(p, "饿了")
		p.leftChopstick.Lock() // 先尝试拿起左手边的筷子
		mark(p, "拿起左手筷子")
		p.rightChopstick.Lock() // 再尝试拿起右手边的筷子
		mark(p, "用膳")
		randomPause(10)
		p.rightChopstick.Unlock() // 先尝试放下右手边的筷子
		p.leftChopstick.Unlock()  // 再尝试拿起左手边的筷子
	}

}

// 随机暂停一段时
func randomPause(max int) {
	time.Sleep(time.Millisecond * time.Duration(rand.Intn(max)))
}

// 显示此哲学家的状态
func mark(p *Philosopher, action string) {
	fmt.Printf("%s开始%s\n", p.name, action)
	p.status = fmt.Sprintf("%s开始%s\n", p.name, action)
}

func main() {
	go func() {
		err := http.ListenAndServe("localhost:8972", nil)
		if err != nil {
			panic(err)
		}
	}()
	// 哲学家数量
	count := 5
	// 创建5根筷子
	chopsticks := make([]*Chopstick, count)
	for i := 0; i < count; i++ {
		chopsticks[i] = new(Chopstick)
	}
	//
	names := []string{color.RedString("孔子"), color.MagentaString("庄子"), color.CyanString("墨子"), color.GreenString("孙子"), color.WhiteString("老子")}
	// 创建哲学家, 分配给他们左右手边的叉子，领他们做到圆餐桌上.
	philosophers := make([]*Philosopher, count)
	for i := 0; i < count; i++ {
		philosophers[i] = &Philosopher{
			name: names[i], leftChopstick: chopsticks[i], rightChopstick: chopsticks[(i+1)%count],
		}
		go philosophers[i].dine()
	}
	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	<-sigs
	fmt.Println("退出中... 每个哲学家的状态:")
	for _, p := range philosophers {
		fmt.Print(p.status)
	}
}

```

运行这个程序你会很快就发现这个程序会hang住，每个哲学家都处于拿到左手筷子等待右手筷子的状态。

在我们实际的应用中，死锁问题并不是这么容易的被发现的，很可能在一些非常特定的场景(也被称之为corner case)才会被触发和发现。

### 解决

我们知道，解决死锁的问题就是破坏死锁形成的四个条件之一就可以。一般来说，禁止抢占和互斥是我们必须的条件，所以其它两个条件是我们重点突破的点。

针对哲学家就餐问题，如果我们限制最多允许四位哲学家同时就餐，就可以避免循环依赖的条件，因为依照抽屉原理，总是会有一位哲学家可以拿到两根筷子，所以程序可以运行下去。

针对限制特定数量资源的情况，最好用的并发原语就是信号量(Semaphore)。Go官方提供了一个扩展库，提供了一个Semaphore的实现：`semaphore/Weighted`。

我们把这个信号量初始值设置为4，代表最多允许同时4位哲学家就餐。把这个信号量传给哲学家对象，哲学家想就餐时就请求这个信号量，如果能得到一个许可，就可以就餐，吃完把许可释放回给信号量。

```go
package main

import (
	"context"
	"fmt"
	"github.com/fatih/color"
	"golang.org/x/sync/semaphore"
	"math/rand"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// Chopstick 代表筷子
type Chopstick struct {
	sync.Mutex
}

// Philosopher 代表哲学家
type Philosopher struct {
	//哲学家的名字
	name string
	//筷子
	leftChopstick, rightChopstick *Chopstick
	//状态
	status string
	// 信号量
	sema *semaphore.Weighted
}

// 需求之的进餐和冥想
// 吃完睡（冥想） 睡完吃
// 可以调整赤水的时间来增加或者减少抢夺筷子的机会
func (p *Philosopher) dine() {
	for {
		mark(p, "冥想")
		randomPause(10)
		mark(p, "饿了")
		_ = p.sema.Acquire(context.Background(), 1)
		//允许一个请求
		p.leftChopstick.Lock() // 先尝试拿起左手边的筷子
		mark(p, "拿起左手筷子")
		p.rightChopstick.Lock() // 再尝试拿起右手边的筷子
		mark(p, "用膳")
		randomPause(10)
		p.rightChopstick.Unlock() // 先尝试放下右手边的筷子
		p.leftChopstick.Unlock()  // 再尝试拿起左手边的筷子
		p.sema.Release(1)
	}

}

// 随机暂停一段时
func randomPause(max int) {
	time.Sleep(time.Millisecond * time.Duration(rand.Intn(max)))
}

// 显示此哲学家的状态
func mark(p *Philosopher, action string) {
	fmt.Printf("%s开始%s\n", p.name, action)
	p.status = fmt.Sprintf("%s开始%s\n", p.name, action)
}

func main() {
	sema := semaphore.NewWeighted(4)
	go func() {
		err := http.ListenAndServe("localhost:8972", nil)
		if err != nil {
			panic(err)
		}
	}()
	// 哲学家数量
	count := 5
	// 创建5根筷子
	chopsticks := make([]*Chopstick, count)
	for i := 0; i < count; i++ {
		chopsticks[i] = new(Chopstick)
	}
	//
	names := []string{color.RedString("孔子"), color.MagentaString("庄子"), color.CyanString("墨子"), color.GreenString("孙子"), color.WhiteString("老子")}
	// 创建哲学家, 分配给他们左右手边的叉子，领他们做到圆餐桌上.
	philosophers := make([]*Philosopher, count)
	for i := 0; i < count; i++ {
		philosophers[i] = &Philosopher{
			name: names[i], leftChopstick: chopsticks[i], rightChopstick: chopsticks[(i+1)%count],
			sema: sema,
		}
		go philosophers[i].dine()
	}
	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	<-sigs
	fmt.Println("退出中... 每个哲学家的状态:")
	for _, p := range philosophers {
		fmt.Print(p.status)
	}
}

```

你可以运行这个程序，看看是否程序是否还会被hang住。

## ref

1. https://colobu.com/2022/02/13/dining-philosophers-problem/
2. http://c.biancheng.net/view/1236.html