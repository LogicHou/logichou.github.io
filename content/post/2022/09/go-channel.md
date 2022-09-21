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

# close 使用上的一些细节

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