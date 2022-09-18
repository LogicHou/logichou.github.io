---
title: "Go常见的并发模式 - 退出模式"
date: 2022-09-18T17:51:27+08:00

categories:
 - Go
tags:
 - Go
 - 并发

draft: false
toc: true
---

# 分离模式

分离模式是使用最为广泛的 goroutine 退出模式。对于分离的 goroutine，创建它的 goroutine 不需要关心它的退出，这类 goroutine 在启动后即与其创建者彻底分离，其生命周期与其执行者的主函数相关，函数返回即 goroutine 退出。这类 goroutine 有两个常见用途。

## 一次性任务

新创建的 goroutine 用来执行一个简单的任务，执行后即退出。

```go
package main

import (
	"time"
)

func stop(done chan struct{}) {
	go func() {
		timer := time.NewTimer(time.Second * 5)
		defer timer.Stop()
		select {
		case <-timer.C:
			println("timeout!")
		case <-done:
			println("done")
		}
	}()

}

func main() {
	done := make(chan struct{})
	stop(done)
	time.Sleep(time.Second * 3)
	done <- struct{}{}
	time.Sleep(time.Second * 7)
}
```

```shell
$ go run main.go
done

如果注释24 25行输出
$ go run main.go
timeout!
```

## 常驻后台执行一些特定任务

如监视(motion)、观察(watch)等。其实现通常采用 for{...} 或 for{ select{...} } 代码段形式，并多以定时器（timer）或事件（event）驱动执行。

```go
package main

import (
	"fmt"
	"time"
)

func monitor() chan int {
	event := make(chan int)
	ticker := time.NewTicker(time.Second * 1)

	go func() {
		defer ticker.Stop()
		for {
			select {
			case e := <-event:
				fmt.Println("Event at", e)
			case t := <-ticker.C:
				fmt.Println("Tick at", t)
			}
		}
	}()

	return event
}

func main() {
	event := monitor()

	event <- 1
	time.Sleep(time.Second * 2)
	event <- 2
	time.Sleep(time.Second * 3)
	event <- 3

	time.Sleep(time.Second * 5)
}
```

```shell
$ go run main.go
Event at 1
Tick at 2022-09-18 10:34:35.8135843 +0800 CST m=+1.013198701
Event at 2
Tick at 2022-09-18 10:34:36.8146391 +0800 CST m=+2.014253501
Tick at 2022-09-18 10:34:37.8163952 +0800 CST m=+3.016009601
Tick at 2022-09-18 10:34:38.8107951 +0800 CST m=+4.010409501
Tick at 2022-09-18 10:34:39.8063766 +0800 CST m=+5.005991001
Event at 3
Tick at 2022-09-18 10:34:40.8064387 +0800 CST m=+6.006053101
Tick at 2022-09-18 10:34:41.8058522 +0800 CST m=+7.005466601
Tick at 2022-09-18 10:34:42.8161325 +0800 CST m=+8.015746901
Tick at 2022-09-18 10:34:43.8137131 +0800 CST m=+9.013327501
Tick at 2022-09-18 10:34:44.814661 +0800 CST m=+10.014275401
```

# join 模式

有时候 goroutine 的创建者需要等待新 goroutine 结束

## 等待一个 goroutine 退出

```go
package main

import "time"

func worker(args ...interface{}) {
	if len(args) == 0 {
		return
	}
	interval, ok := args[0].(int) // 获取第一个参数作为time.Duration的参数，由于是interface所以需要通过类型断言转换成int类型
	if !ok {
		return
	}

	time.Sleep(time.Second * (time.Duration(interval)))
}

func spawn(f func(args ...interface{}), args ...interface{}) chan struct{} {
	c := make(chan struct{})
	go func() {
		f(args...)
		c <- struct{}{}
	}()
	return c
}

func main() {
	done := spawn(worker, 5)
	println("spawn a worker goroutine")
	<-done // 直到收到信号前，会一直在这里阻塞住main goroutine
	println("worker done")
}
```

```shell
$ go run main.go
spawn a worker goroutine
worker done
```

spawn 函数使用典型的 goroutine 创建模式创建了一个 goroutine，main goroutine 作为创建者通过 spawn 函数返回的 channel 与新 goroutine 建立联系，这个 channel 的用途就是在两个 goroutine 之间建立退出事件的“信号”通信机制。main goroutine 在创建完新 goroutine 后便在该 channel 上阻塞等待，直到新 goroutine 退出前向该 channel 发送了一个信号，此时 main goroutine 中的 done channel 收到信号后解除了阻塞继续向下执行，直到 main goroutine 退出。

## 获取 goroutine 的退出状态

如果新 goroutine 的创建者不仅要等待 goroutine 的退出，还要精准获取其结束状态，同样可以通过自定义类型的 channel 来实现这一场景需求

```go
package main

import (
	"errors"
	"fmt"
	"time"
)

var OK = errors.New("ok")

func worker(args ...interface{}) error {
	if len(args) == 0 {
		return errors.New("invalid args")
	}
	interval, ok := args[0].(int)
	if !ok {
		return errors.New("invalid interval arg")
	}

	time.Sleep(time.Second * (time.Duration(interval)))
	return OK
}

func spawn(f func(args ...interface{}) error, args ...interface{}) chan error {
	c := make(chan error) // 承载的类型由struct{}改成了error，这样channel承载的信息就不只是一个信号了，还携带了有价值的信息：新goroutine的结束状态
	go func() {
		c <- f(args...)
	}()
	return c
}

func main() {
	done := spawn(worker, 5)
	println("spawn worker1")
	err := <-done
	fmt.Println("worker1 done:", err)
	done = spawn(worker)
	println("spawn worker2")
	err = <-done
	fmt.Println("worker2 done:", err)
}
```

```shell
$ go run main.go
spawn worker1
ok
worker1 done: ok
spawn worker2
worker2 done: invalid args
```

## 等待多个 goroutine 退出

在有些场景中，goroutine 的创建者可能会创建不止一个 goroutine，并且需要等待全部新 goroutine 退出，这时可以通过 Go 语言提供的 sync.WaitGroup 实现等待多个 goroutine 退出。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(args ...interface{}) {
	if len(args) == 0 {
		return
	}

	interval, ok := args[0].(int)
	if !ok {
		return
	}

	time.Sleep(time.Second * (time.Duration(interval)))
}

func spawnGroup(n int, f func(args ...interface{}), args ...interface{}) chan struct{} {
	c := make(chan struct{})
	var wg sync.WaitGroup

	for i := 0; i < n; i++ { // spawnGroup创建了3个新的goroutine
		wg.Add(1)
		go func(i int) {
			name := fmt.Sprintf("worker-%d:", i)
			f(args...)
			println(name, "done")
			wg.Done() // worker done!
		}(i)
	}

	go func() {
		wg.Wait()       // 上面创建的3个goroutine执行结束后，wg.Wait()会停止阻塞，继续向下执行
		c <- struct{}{} // 向done channel发送结束信号
	}()

	return c
}

func main() {
	done := spawnGroup(5, worker, 3)
	println("spawn a group of workers")
	<-done  // 收到信号后解除阻塞
	println("group workers done")
}
```

```shell
$ go run main.go
spawn a group of workers
worker-0: done
worker-4: done
worker-2: done
worker-1: done
worker-3: done
group workers done
```

## 支持超时机制的等待

如果不想无限阻塞等待所有新创建 goroutine 的退出，可以仅等待一段合理的时间，如果在这段时间内 goroutine 没有退出（触发done），则创建者会继续向下执行或主动退出

```go
// 省略部分代码...和上面例子中的相同...

func main() {
	done := spawnGroup(5, worker, 30)
	println("spawn a group of workers")

	timer := time.NewTimer(time.Second * 5)
	defer timer.Stop()
	select {
	case <-timer.C:   // 5秒后这个channel会接收到一个通知，如果这之前done channel没有被触发，则主程序继续向下运行直到退出
		println("wait group workers exit timeout!")
	case <-done:
		println("group workers done")
	}
}
```

```shell
$ go run main.go
spawn a group of workers
wait group workers exit timeout!
```

# notify-and-wait 模式

在前面的几个场景中，goroutine 的创建者都是在被动地等待着新 goroutine 的退出。但很多时候，goroutine 创建者需要主动通知那些新 goroutine 退出，尤其是当 main goroutine 作为创建者时。main goroutine 退出意味着 Go 程序终止，而粗暴地直接让 main goroutine 退出的方式可能会导致业务数据损坏、不完整或丢失。

可以通过 notify-and-wait（通知并等待）模式来满足这一场景的要求。虽然这一模式也不能完全避免损失，但是给了各个 goroutine 一个挽救数据的机会，从而尽可能减少损失。

## 通知并等待一个 goroutine 退出

```go
package main

import "time"

func worker(j int) {
	time.Sleep(time.Second * (time.Duration(j)))
}

func spawn(f func(int)) chan string {
	quit := make(chan string)
	go func() {
		var job chan int // 模拟job channel
		for {
			select {
			case j := <-job: // 这个例子中这里会一直阻塞住不会被执行到，只是一种模拟
				f(j)
			case <-quit:
				quit <- "ok"  // 承载新 goroutine 返回给创建者的退出状态
				return
			}
		}
	}()
	return quit
}

func main() {
	quit := spawn(worker)
	println("spawn a worker goroutine")

	time.Sleep(5 * time.Second)

	// 通知子goroutine退出
	println("notify the worker to exit...")
	quit <- "exit"  // 承载创建者发送给新 goroutine 的退出信号

	timer := time.NewTimer(time.Second * 10)
	defer timer.Stop()
	select {
	case status := <-quit:
		println("worker done:", status)
	case <-timer.C:
		println("wait worker exit timeout")
	}
}
```

```shell
$ go run main.go
spawn a worker goroutine
notify the worker to exit...
worker done: ok
```

这里使用创建模式 goroutine 的 spawn 函数返回的 channel 的作用发生了变化，从原先的只是用于新 goroutine 发送退出信号给创建者，变成了一个**双向**的数据通道：既承载创建者发送给新 goroutine 的退出信号，也承载新 goroutine 返回给创建者的退出状态

## 通知并等待多个 goroutine 退出

Go 语言的 channel 有一个特性是，当使用 close 函数关闭 channel 时，所有阻塞到该 channel 上的 goroutine 都会得到通知。利用这一特性可以实现满足这一场景的模式。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(j int) {
	time.Sleep(time.Second * (time.Duration(j)))
}

func spawnGroup(n int, f func(int)) chan struct{} {
	quit := make(chan struct{})
	job := make(chan int)
	var wg sync.WaitGroup

	for i := 0; i < n; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done() // 保证wg.Done在goroutine退出前被执行
			name := fmt.Sprintf("worker-%d:", i)
			for {
				j, ok := <-job  // 通道被关闭后，ok会收到值false表示通道已经关闭了
				if !ok {        // 如果不用ok判断，依然可以从关闭的通道中收到通道类型的零值，所以这里用ok作为判断条件很重要
					println(name, "done")
					return
				}
				// do the job
				worker(j)
			}
		}(i)
	}

	go func() {
		<-quit
		close(job) // 广播给所有新goroutine
		wg.Wait()
		quit <- struct{}{}
	}()

	return quit
}

func main() {
	quit := spawnGroup(5, worker)
	println("spawn a group of workers")

	time.Sleep(5 * time.Second)
	// 通知 worker goroutine group 退出，接到通知后通过关闭job广播通知所有的子goroutine进行退出
	println("notify the worker group to exit...")
	quit <- struct{}{}

	timer := time.NewTimer(time.Second * 5)
	defer timer.Stop()
	select {
	case <-timer.C:
		println("wait group workers exit timeout!")
	case <-quit:
		println("group workers done")
	}
}
```

```shell
❯ go run main.go
spawn a group of workers
notify the worker group to exit...
worker-2: done
worker-3: done
worker-0: done
worker-1: done
worker-4: done
group workers done
```

# 退出模式的应用

很多时候，在程序中要启动多个 goroutine 协作完成应用的业务逻辑。但这些 goroutine 的运行形态很可能不同，有的扮演服务端，有的扮演客户端，等等，因此似乎很难用一种统一的框架全面管理它们的启动、运行和退出。但是可以尝试将问题范围缩小，聚焦在实现一个“超时等待退出”框架，以统一解决各种运行形态 goroutine 的优雅退出问题。

**一组 goroutine 的退出**总体上有两种情况。一种是**并发退出**，在这类退出方式下，各个 goroutine 的退出先后次序对数据处理无影响，因此各个 goroutine 可以并发执行退出逻辑；另一种则是**串行退出**，即各个 goroutine 之间的退出时按照一定次序这个执行的，次序若错了可能会导致程序的状态混乱和错误。

示例通用部分代码：

```go
package main

import (
	"errors"
	"fmt"
	"sync"
	"time"
)

type GracefullyShutdowner interface {
	Shutdown(waitTimeout time.Duration) error
}

type ShutdownerFunc func(time.Duration) error

func (f ShutdownerFunc) Shutdown(waitTimeout time.Duration) error {
	return f(waitTimeout) // 这里f的实参是shutdownMaker
}

// 并发退出
func ConcurrentShutdown(waitTimeout time.Duration, shutdowners ...GracefullyShutdowner) error {
	c := make(chan struct{})

	go func() {
		var wg sync.WaitGroup
		for _, g := range shutdowners {
			wg.Add(1)
			go func(shutdowner GracefullyShutdowner) {
				defer wg.Done()
				shutdowner.Shutdown(waitTimeout)
			}(g)
		}
		wg.Wait()
		c <- struct{}{}
	}()

	timer := time.NewTimer(waitTimeout)
	defer timer.Stop()

	select {
	case <-c:       // 正常退出
		return nil
	case <-timer.C: // 超时退出
		return errors.New("wait timeout")
	}
}

// 串行退出
func SequentialShutdown(waitTimeout time.Duration, shutdowners ...GracefullyShutdowner) error {
	start := time.Now()
	var left time.Duration
	timer := time.NewTimer(waitTimeout)

	for _, g := range shutdowners {
		elapsed := time.Since(start)
		left = waitTimeout - elapsed

		c := make(chan struct{})
		go func(shutdowner GracefullyShutdowner) {
			shutdowner.Shutdown(left)
			c <- struct{}{}
		}(g)

		timer.Reset(left)
		select {
		case <-c:       // 正常退出
			//continue
		case <-timer.C: // 超时退出
			return errors.New("wait timeout")
		}
	}

	return nil
}

func shutdownMaker(processTm int) func(time.Duration) error {
	return func(time.Duration) error {
		time.Sleep(time.Second * time.Duration(processTm))
		return nil
	}
}
```

## 并发退出

```go
func main() {
	// TestConcurrentShutdown
	f1 := shutdownMaker(2)
	f2 := shutdownMaker(6)

	err := ConcurrentShutdown(10*time.Second, ShutdownerFunc(f1), ShutdownerFunc(f2))
	if err != nil {
		fmt.Printf("TestConcurrentShutdown want nil, actual: %s", err)
	}

	err = ConcurrentShutdown(4*time.Second, ShutdownerFunc(f1), ShutdownerFunc(f2))
	if err == nil {
		fmt.Printf("TestConcurrentShutdown want timeout, actual nil")
	}

	println("OK")
}
```

## 串行退出

```go
func main() {
	// TestSequentialShutdown
	f1 := shutdownMaker(2)
	f2 := shutdownMaker(6)

	err := SequentialShutdown(10*time.Second, ShutdownerFunc(f1), ShutdownerFunc(f2))
	if err != nil {
		fmt.Printf("TestSequentialShutdown want nil, actual: %s", err)
	}

	err = SequentialShutdown(5*time.Second, ShutdownerFunc(f1), ShutdownerFunc(f2))
	if err == nil {
		fmt.Printf("TestSequentialShutdown want timeout, actual nil")
	}

	println("OK")
}
```

串行退出有个问题是 waitTimeout 值的确定，因为这个超时时间是所有 goroutine 的退出时间之和。上面示例将每次的 left（剩余时间）传入下一个要执行的 goroutine 的 Shutdown 方法中。select 同样使用这个 left 作为 timeout 的值（通过 timer.Reset 重新设置 timer 定时器周期）。