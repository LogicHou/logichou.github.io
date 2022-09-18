---
title: "Go常见的并发模式 - 创建模式"
date: 2022-09-17T22:51:19+08:00
draft: false
toc: true
---

在内部创建一个 goroutine 并返回一个 channel 类型的变量的函数，是 Go 中最常见的 goroutine 创建模式。

<!--more-->

```go
package main

import "fmt"

func spawn(f func() int) chan int {
	c := make(chan int)
	go func() {
		c <- f()
	}()

	return c
}

func main() {
	c := spawn(func() int {
		return 100
	})
	fmt.Println(<-c)
}
```

spawn 函数创建的新 goroutine 与调用 spawn 函数的 goroutine 之间通过一个 channel 建立起了联系：两个 goroutine 可以通过这个 channel 进行通信。