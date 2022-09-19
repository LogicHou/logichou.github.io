---
title: "Go常见的并发模式 - 管道模式"
date: 2022-09-19T16:11:47+08:00

categories:
 - Go
tags:
 - Go
 - 并发

draft: false
toc: true
---

# 管道模式

在 Go 中管道模式被实现成了一条由 channel 连接的一条“数据流水线”。在该流水线中，每个数据处理环节都由**一组功能相同的 goroutine** 完成。在每个数据处理环节，goroutine 都要从数据输入 channel 获取前一个环节产生的数据，然后对这些数据进行处理，并将处理后的结果数据通过数据输出 channel 发往下一个环节。

```go
package main

func newNumGenerator(start, count int) <-chan int {
	c := make(chan int)
	go func() {
		for i := start; i < start+count; i++ {
			c <- i
		}
		close(c)
	}()
	return c
}

// 过滤掉奇数，保留偶数
func filterOdd(in int) (int, bool) {
	if in%2 != 0 {
		return 0, false
	}
	return in, true
}

// 求平方
func square(in int) (int, bool) {
	return in * in, true
}

func spawn(f func(int) (int, bool), in <-chan int) <-chan int {
	out := make(chan int)

	go func() {
		for v := range in {
			r, ok := f(v)
			if ok {
				out <- r
			}
		}
		close(out)
	}()

	return out
}

func main() {
	in := newNumGenerator(1, 20)
	out := spawn(square, spawn(filterOdd, in)) // 可以把spawn(filterOdd, in)看成是一个in

	for v := range out { // 不关闭out通道，range会一直从out里读取数据，如果没有数据返回就读取为通道类型的零值
		println(v)
	}
}
```

```shell
$ go run main.go
4
16 
36 
64 
100
144
196
256
324
400
```

管道模式具有良好的**可扩展性**，可以继续连接新的管道进行数据过滤和处理。

基于管道模式的另外两种**扩展模式**

## 扇出模式 fan-out

在某个处理环节，多个功能相同的 goroutine 从同一个 channel 读取数据并处理，直到该 channel 关闭。使用扇出模式可以在一组 goroutine 中均衡分配工作量，从而更均衡地利用 CPU。

## 扇入模式 fan-in

在某个处理环节，处理程序面对不止一个输入 channel。我们把所有输入 channel 的数据汇聚到一个统一的输入 channel，然后处理程序再从这个 channel 中读取数据并处理，直到该 channel 因所有输入 channel 关闭而关闭。

## 综合示例

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func newNumGenerator(start, count int) <-chan int {
	c := make(chan int)
	go func() {
		for i := start; i < start+count; i++ {
			c <- i
		}
		close(c)
	}()
	return c
}

// 取偶数
func filterOdd(in int) (int, bool) {
	if in%2 != 0 {
		return 0, false
	}
	return in, true
}

// 求平方
func square(in int) (int, bool) {
	return in * in, true
}

func spawnGroup(name string, num int, f func(int) (int, bool), in <-chan int) <-chan int {
	groupOut := make(chan int)
	// Fan-out
	//
	//  	   /-- out --\
	//        /			  \
	// in ---- --> out ---- --> outSlice
	//        \			  /
	//         \-- out --/
	//
	var outSlice []chan int
	for i := 0; i < num; i++ {
		out := make(chan int)
		go func(i int) { // 开启num个goroutine分别从in中接收数据
			name := fmt.Sprintf("%s-%d:", name, i)
			fmt.Printf("%s begin to work...\n", name)

			for v := range in {
				r, ok := f(v)
				if ok {
					out <- r
				}
			}
			close(out)
			fmt.Printf("%s work done\n", name)
		}(i)
		outSlice = append(outSlice, out) // 将多个out打包到一个slice中，提供给后续的Fan-in使用
	}

	// Fan-in
	//
	// out --\
	//        \
	// out ---- --> groupOut
	//        /
	// out --/
	//
	go func() {
		var wg sync.WaitGroup
		for _, out := range outSlice {
			wg.Add(1)
			go func(out <-chan int) {
				for v := range out {
					groupOut <- v
				}
				wg.Done()
			}(out)
		}
		wg.Wait()
		close(groupOut)
	}()

	return groupOut
}

func main() {
	in := newNumGenerator(1, 20)
	out := spawnGroup("square", 2, square, spawnGroup("filterOdd", 3, filterOdd, in))

	time.Sleep(3 * time.Second)

	for v := range out {
		fmt.Println(v)
	}
}
```

```shell
❯ go run main.go
square-0: begin to work...
filterOdd-0: begin to work...
filterOdd-1: begin to work...
filterOdd-2: begin to work...
square-1: begin to work...
4
36
16
100
196
144
64
256
324
400
```