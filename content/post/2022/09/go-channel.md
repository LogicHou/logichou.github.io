---
title: "Go语言基础 - channel速览"
date: 2022-09-21T18:23:10+08:00

categories:
 - Go
tags:
 - Go

draft: false
toc: true
---

# channel 的概念

可以实现 goroutine 减的通信，也可以实现 goroutine 间的同步。

channel 类型在 Go 中为“一等公民”，可以像使用普通变量那样使用 channel，比如：定义 channel 类型变量，为 channel 变量赋值，将 channel 作为参数传递给函数 / 方法，将 channel 作为返回值从函数 / 方法返回，甚至将 channel 发送到其他 channel 中。

# channel 用法

```go
c := make(chan int)      // 创建一个无缓冲（unbuffered）的int类型channel
c := make(chan int 5)    // 创建一个带缓冲的int类型的channel
c <-x                    // 向channel c中发送一个值
<-c                      // 从channel c中接收一个值
x = <- c                 // 从channel c接收一个值并将其存储到变量x中
x, ok = <-c              // 从channel c接收一个值。若channel关闭了，ok将被置为false
for i := range c {...}   // 将for range与channel结合使用，若channel关闭了，for range会接收完channel中的所有值（比如带缓冲channel）再结束
close(c)                 // 关闭channel c

c := make(chan chan int) // 创建一个无缓冲的chan int类型的channel
func stream(ctx context.Context, out chan <- Value) error // 将只发送（send-only）channel作为函数参数
func spawn(...) <-chan T // 将只接收（receive-only）类型channel作为返回值
```

# len(channel) 的应用

以 len(s) 为例：

* 当 s 为无缓冲 channel 时，len(s) 总是返回 0；
* 当 s 为带缓冲 channel 时，len(s) 返回当前 channel s 中尚未被读取的元素个数

不能简单的对 channel 使用 len 进行“判满”和“判空”逻辑，channel 原语用于多个 goroutine 间的通信，一旦多个 goroutine 共同对 channel 进行手法操作，len(channel) 就会在多个 goroutine 间形成竞态，单纯依靠 len(channel) 来判断 channel 中元素的状态，不能保证后续对 channel 进行收发时 channel 的状态不变。

常见的方法是将判空与读取放在一个事务中，将判满与写入放在一个事务中，而这类事务可以通过 select 实现。

```go
package main

import (
	"fmt"
	"time"
)

// 生产消费者模式中的生产者
func producer(c chan<- int) {
	var i int = 1
	for {
		time.Sleep(2 * time.Second)
		ok := trySend(c, i)
		if ok {
			fmt.Printf("[producer]: send [%d] to channel\n", i)
			i++
			continue
		}
		fmt.Printf("[producer]: try send [%d], but channel is full\n", i)
	}
}

func tryRecv(c <-chan int) (int, bool) {
	select {
	case i := <-c:    // 如果能从channel里接收数据，则自然说明是非空的
		return i, true

	default:
		return 0, false // channel为空的情况下，select就只能选择这个分支，表示channel是空的
	}
}

func trySend(c chan<- int, i int) bool {
	select {
	case c <- i:   // 如果能往channel里发送数据，则自然说明是不满的
		return true
	default:
		return false // channel满的情况下，select就只能选择这个分支，表示channel是满的
	}
}

// 生产消费者模式中的接收者
func consumer(c <-chan int) {
	for {
		i, ok := tryRecv(c)
		if !ok {
			fmt.Println("[consumer]: try to recv from channel, but the channel is empty")
			time.Sleep(1 * time.Second)
			continue
		}
		fmt.Printf("[consumer]: recv [%d] from channel\n", i)
		if i >= 3 {
			fmt.Println("[consumer]: exit")
			return
		}
	}
}

func main() {
	c := make(chan int, 3)
	go producer(c)
	go consumer(c)

	select {} // 故意阻塞在此
}
```

这么做有一个问题，就是改变了 channel 的状态：接收或发送了一个元素。想在不改变 channel 状态的前提下单纯的侦测 channel 的状态，又不会因 channel 满或空阻塞在 channel 上，目前没有一种方法既可以实现这样的功能又适合所有场合。

在一些特定的场景下，可以只用 len(channel) 实现“判满”和“判空”：

* 多发送单接收的场景，即有多个发送者，但**有且只有一个接收者**，这时可以在接收者 goroutine 中根据 len(channel) 是否大于 0 来判断 channel 中是否有数据需要接收
* 多接收单发送的场景，即有多个接收者，但**有且只有一个发送者**，这时可以在发送者 goroutine 中根据 len(channel) 是否小于 cap(channel) 来判断是否可以执行向 channel 发送数据

# close 使用上的一些细节

## 已关闭的 channel 仍能从中接收值

从一个已关闭的 channel 接收数据永远不会被阻塞，会得到该 channel 对应类型的零值。

```go
package main

func main() {
	c := make(chan int)

	go func() {
		c <- 99
		close(c)
	}()

	println(<-c)
	println(<-c)
	println(<-c)
}
```

```shell
$ go run main.go
99
0
0
```

通过将 channel 设置为 nil 进行阻塞行为

```go
package main

import "fmt"
import "time"

func main() {
	c1, c2 := make(chan int), make(chan int)
	go func() {
		time.Sleep(time.Second * 5)
		c1 <- 5
		close(c1)
	}()

	go func() {
		time.Sleep(time.Second * 7)
		c2 <- 7
		close(c2)
	}()

	for {
		select {
		case x, ok := <-c1:
			if !ok {
				c1 = nil        // 不设置为nil的话，会一直从c1中接收到对应通道类型的零值
			} else {
				fmt.Println(x)
			}
		case x, ok := <-c2:
			if !ok {
				c2 = nil
			} else {
				fmt.Println(x)
			}
		}
		if c1 == nil && c2 == nil {
			break
		}
	}
	fmt.Println("program end")
}
```

```shell
$ go run main.go
5
7
program end
```

## close 后 for range 的行为

关闭一个正在通过 for range 接收数据的通道后，for range 会接收完通道内的数据才结束，以带缓冲的通道作为演示：

```go
package main

import (
	"time"
)

func main() {

	buffch := make(chan int, 5)

	go func() {
		for v := range buffch {
			println(v)
			time.Sleep(time.Second * 1)
		}
	}()

	buffch <- 1
	buffch <- 2
	buffch <- 3
	buffch <- 4
	buffch <- 5

	time.Sleep(time.Second * 3)
	close(buffch)
	println("buffch closed")

	time.Sleep(time.Second * 10)
}
```

```shell
$ go run main.go
1
2
3
buffch closed
4
5
```

## 可以使所有接收者收到广播

有一种场景，只有一个发送者，但是有多个接收者，这时如果 channel 作为信号来使用，只往 channel 发送数据一次只能由一个接收者接收到数据，但是使用 close 可以同时触发多个接收者接收到通知，不过这种方式是一次性的，适合用来作为退出信号

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup
	quit := make(chan struct{})
	go func() {
		wg.Add(1)
	LOOP:
		for {
			select {
			default:
				time.Sleep(time.Second * 1)
			case <-quit:
				fmt.Println("break loop 1")
				break LOOP
			}
		}
		wg.Done()
	}()

	go func() {
		wg.Add(1)
	LOOP:
		for {
			select {
			default:
				time.Sleep(time.Second * 1)
			case <-quit:
				fmt.Println("break loop 2")
				break LOOP
			}
		}
		wg.Done()
	}()

	time.Sleep(time.Second * 3)
	close(quit)
	// quit <- struct{}{}

	wg.Wait()
}
```

```shell
$ go run main.go
break loop 2
break loop 1
```

如果将 close(quit) 替换成 quit <- struct{}{} 将只有一个接收者接收到数据

```shell
$ go run main.go
break loop 1
```

# nil channel 的应用

对没有初始化的 channel（nil channel）进行读写操作将会发生阻塞。

```go
package main

func main() {
	var c chan int
	<-c
}

or

package main

func main() {
	var c chan int
	c <- 1
}
```

```shell
$ go run main.go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive (nil chan)]:

or 

$ go run main.go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send (nil chan)]:
```

# select

通过 select，可以同时在多个 channel 上进行发送 / 接收操作

```go
select {
  case x := <- c1:    // 从channel c1 接收数据
    ...
  case y, ok := <-c2: // 从channel c2 接收数据，并根据ok值判断c2是否已经关闭
    ...
  case c3 := <- z:    // 将z值发送到channel c3中
    ...
  default:            // 当上面case中的channel通信均无法实施时，执行该默认分支
}
```

## 惯用法

### 利用 default 分支避免阻塞

```go
func sendTime(c interface{}, seq uintptr) {
  // 无阻塞地向c发送当前时间
  // ...
  select {
    case c.(chan Time) <- Noe():
    default:
  }
}
```

### 实现超时机制

通过超时事件，可以避免长长期陷入某种操作的等待中，也可以做一些异常处理工作。

一次具有 30s 超时的 select 的实现：

```go
func worker() {
  select {
    case <-c:
      // ...
    case <-time.After(30 * time.Second):
      return
  }
}
```

在应用具有超时机制的 select 时，要特别注意 timer 使用后的释放，尤其是在大量创建 timer 时。

### 实现心跳机制

这种机制可以在监听 channel 的同时，执行一些周期性的任务：

```go
func worker() {
	heartbeat := time.NewTicker(30 * time.Second)
	defer heartbeat.Stop()
	for {
		select {
		case <-c:
			// ... 处理业务逻辑
		case <-heartbeat.C:
			// ... 处理心跳
		}
	}
}
```

与 timer 一样，在使用完 ticker 之后，要记得调用其 Stop 方法停止 ticker 的运作，这样在 heartbeat.C 上就不会再持续产生心跳事件了。