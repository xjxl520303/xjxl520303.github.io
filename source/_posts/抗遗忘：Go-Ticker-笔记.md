---
title: 抗遗忘：Go Ticker 笔记
date: 2024-05-09 19:51:39
topic: go
tags:
    - Go
---

`time.Ticker` 是周期性定时器，按指定的时间间隔重复向 Channel (`C`) 发送时间值。通过实例 ` ticker.Stop() ` 可以停止定时器。如果只想返回一个时间值，而不必关闭它，这个时候可以选择使用 `time.Tick()` 方法。

## 输出间隔时间值

创建一个定时器，每隔 1 秒发送一次当前时间，在 `main` 函数中使用 `time.Sleep(5 * time.Second)` 等待 5 秒给定时器所在的协程执行，5 秒后主函数退出时内部的协程也将被终止，所以输出了 4 次日志。

除了使用时间去阻塞主协程外，我们还可以利用 Channel 在什么情况下会阻塞当前协程来实现 `main` 函数一直挂起，比如：`make(chan struct{}) <- struct{}{}`，`<-make(chan struct{})`, `<-chan struct{})(nil)` 和 `select{}`，当然我们也可以暴力直接写一个 `for{}`。

```go
package main

import (
	"log"
	"time"
)

func main() {
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()

	go func() {
		for t := range ticker.C {
			log.Printf("Tick at %v\n", t.UTC())
		}
	}()

	time.Sleep(5 * time.Second)
}
```

输出：

```
2024/04/10 15:15:03 Tick at 2024-04-10 07:15:03.9905864 +0000 UTC
2024/04/10 15:15:04 Tick at 2024-04-10 07:15:04.9827231 +0000 UTC
2024/04/10 15:15:05 Tick at 2024-04-10 07:15:05.989957 +0000 UTC
2024/04/10 15:15:06 Tick at 2024-04-10 07:15:06.980916 +0000 UTC
```

## 立即执行，不用等待第一次间隔

上一个示例中如果我们在 `main()` 函数内第一行加上 `fmt.Println("Starting ticker: ", time.Now().UTC())`，查看终端输出会发现实际上是间隔了我们设定的间隔值才输出定时器发送的时间值。如果我们需要在 `for` 中立即拿到定时器发送的时间，可以这样子来实现：

```go
package main

import (
	"fmt"
	"log"
	"time"
)

func main() {
	fmt.Println("Starting ticker: ", time.Now().UTC())
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()

	go func() {
		for ; ; <-ticker.C {
			log.Printf("Tick at %v\n", time.Now().UTC())
		}
	}()

	time.Sleep(5 * time.Second)
}
```

输出：

```
Starting ticker:  2024-04-10 07:57:05.6394014 +0000 UTC
2024/04/10 15:57:05 Tick at 2024-04-10 07:57:05.6394014 +0000 UTC
2024/04/10 15:57:06 Tick at 2024-04-10 07:57:06.6450901 +0000 UTC
2024/04/10 15:57:07 Tick at 2024-04-10 07:57:07.6513953 +0000 UTC
2024/04/10 15:57:08 Tick at 2024-04-10 07:57:08.6409958 +0000 UTC
2024/04/10 15:57:09 Tick at 2024-04-10 07:57:09.6480314 +0000 UTC
```

上面的代码中我们注意到 `for ; ; <-ticker.C` 这里我们并没有读取定时器发送过来的时间，并且日志输出的时间是使用的 `time.Now().UTC()`，这种方式并不是很好的实践，还可能存在意想不到的副作用，所以改成下面这种方式才是最佳实践：

```go
for t := time.Now(); ; t = <-ticker.C {
	log.Printf("Tick at: %v\n", t.UTC())
}
```

## 停止发送，并关闭 Channel 

我们调用 `ticker.Stop()` 只是停止了定时器再次发送值，并没有关闭 Channel，要关闭 Channel 还需要一个额外信号来通知关闭。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ticker := time.NewTicker(time.Second)
	done := make(chan bool)

	go func() {
		for {
			select {
			case <-done:
				return
			case t := <-ticker.C:
				fmt.Println("Tick at", t.UTC())
			}
		}
	}()
	time.Sleep(5 * time.Second)
	ticker.Stop()
	done <- true
	fmt.Println("Ticker stopped")
}
```

输出：

```go
Tick at 2024-04-10 08:53:42.403751 +0000 UTC
Tick at 2024-04-10 08:53:43.3943259 +0000 UTC
Tick at 2024-04-10 08:53:44.4026484 +0000 UTC
Tick at 2024-04-10 08:53:45.3922398 +0000 UTC
Tick at 2024-04-10 08:53:46.400103 +0000 UTC
Ticker stopped
```

## 根据条件关闭定时器

每隔一秒给 `counter` 加一，当其值大于 `5` 时关闭定时器，并退出。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ticker := time.NewTicker(time.Second)
	counter := 0

	for {
		select {
		case <-ticker.C:
			counter++
			fmt.Println("Counter:", counter)
			if counter >= 5 {
				ticker.Stop()
				fmt.Println("Ticker stopped")
				return
			}
		}
	}
}
```

## 举个公交车场景例子

公交车每隔5分钟发一班，不管是否已坐满乘客，已经坐满乘客情况下，不足五分钟也发车。

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

type Passenger struct{ id int }
type Bus struct {
	maxCap     int
	passengers []Passenger
	cap        int
}

func NewBus(maxCap int) *Bus {
	return &Bus{
		maxCap:     maxCap,
		passengers: make([]Passenger, 0),
		cap:        0,
	}
}

func (b *Bus) AddPassenger(p Passenger) {
	if b.cap < b.maxCap {
		fmt.Printf("乘客#%d 上车\n", p.id)
		b.passengers = append(b.passengers, p)
		b.cap++
		time.Sleep(time.Duration(rand.Intn(5000)) * time.Millisecond)
		fmt.Printf("乘客#%d 下车\n", p.id)
		b.passengers = DeleteSlice(b.passengers, p)
		b.cap--
		fmt.Println("当前乘客：", b.passengers)
	} else {
		done <- true
	}
}

var done = make(chan bool)

func main() {
	bus := NewBus(50)
	ticker := time.NewTicker(3 * time.Second)
	defer ticker.Stop()

	for passengerId := 0; ; passengerId++ {
		select {
		case <-done:
			fmt.Println("座位满了")
			return
		case <-ticker.C:
			fmt.Println("时间到了，开始发车，总人数：", bus.cap)
			return
		default:
			time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
			passenger := Passenger{id: passengerId}
			go bus.AddPassenger(passenger)
		}
	}
}

// 删除切片中的元素
func DeleteSlice(s []Passenger, elem Passenger) []Passenger {
	r := s[:0]
	for _, v := range s {
		if v != elem {
			r = append(r, v)
		}
	}
	return r
}

```

时间到了输出：

```
乘客#0 上车
乘客#1 上车
乘客#2 上车
乘客#3 上车
乘客#4 上车
乘客#3 下车
当前乘客： [{0} {1} {2} {4}]
乘客#5 上车
乘客#6 上车
乘客#6 下车
当前乘客： [{0} {1} {2} {4} {5}]
乘客#7 上车
乘客#0 下车
当前乘客： [{1} {2} {4} {5} {7}]
乘客#8 上车
乘客#9 上车
乘客#1 下车
当前乘客： [{2} {4} {5} {7} {8} {9}]
乘客#10 上车
乘客#9 下车
当前乘客： [{2} {4} {5} {7} {8} {10}]
乘客#4 下车
当前乘客： [{2} {5} {7} {8} {10}]
时间到了，开始发车，总人数： 5
```

将人数设置为 `3` 人，下车的时间设置成 `time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)` 输出：

```
乘客#0 上车
乘客#1 上车
乘客#2 上车
乘客#0 下车
当前乘客： [{1} {2}]
乘客#2 下车
当前乘客： [{1}]
乘客#3 上车
乘客#1 下车
当前乘客： [{3}]
乘客#4 上车
乘客#5 上车
乘客#3 下车
当前乘客： [{4} {5}]
乘客#7 上车
座位满了
```

## 参考：

- [go定时器--Ticker - failymao - 博客园 (cnblogs.com)](https://www.cnblogs.com/failymao/p/15068712.html)
- [5 different ways to loop over a time.Ticker in Go (Golang) (gosamples.dev)](https://gosamples.dev/range-over-ticker/)
- [Some Simple Summaries -Go 101](https://go101.org/article/summaries.html#block-forever)