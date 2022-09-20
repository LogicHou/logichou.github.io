---
title: "Go常见的并发模式 - 超时与取消模式"
date: 2022-09-20T20:32:11+08:00

categories:
 - Go
tags:
 - Go
 - 并发

draft: false
toc: true
---

# 示例1 尝试从天气中心获取数据

编写一个从气象数据中心获取天气信息的客户端。该客户端每次会并发向三个气象数据服务中心发起数据查询请求，并以最快返回的那个响应信息作为此次请求的应答返回值。

```go
package main

import (
	"io/ioutil"
	"log"
	"net/http"
	"net/http/httptest"
	"time"
)

type result struct {
	value string
}

func first(servers ...*httptest.Server) (result, error) {
	c := make(chan result, len(servers)) // 带缓冲的channel
	queryFunc := func(server *httptest.Server) {
		url := server.URL
		resp, err := http.Get(url)
		if err != nil {
			log.Printf("http get error: %s\n", err)
			return
		}
		defer resp.Body.Close()
		body, _ := ioutil.ReadAll(resp.Body)
		c <- result{ // 三个发起查询的goroutine都会将应答结构写入这里，由于是带缓冲的channel最大同时可以放入3个数据
			value: string(body),
		}
	}
	for _, serv := range servers {
		go queryFunc(serv)
	}
	return <-c, nil // 从这个带缓冲的channel中返回最先进入的那个，也就是最快返回气象数据的那个
}

func fakeWeatherServer(name string) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Printf("%s receive a http request\n", name)
		time.Sleep(1 * time.Second)
		w.Write([]byte(name + ":ok"))
	}))
}

func main() {
	result, err := first(fakeWeatherServer("open-weather-1"),
		fakeWeatherServer("open-weather-2"),
		fakeWeatherServer("open-weather-3"))
	if err != nil {
		log.Println("invoke first error:", err)
		return
	}

	log.Println(result)
}
```

```shell
$ go run main.go
2022/09/18 22:41:59 open-weather-3 receive a http request
2022/09/18 22:41:59 open-weather-2 receive a http request
2022/09/18 22:41:59 open-weather-1 receive a http request
2022/09/18 22:42:00 {open-weather-2:ok}
```

# 示例2 超时模式

现实中的网络情况错综复杂，远程服务器的状态也未必能保持百分百可用。为了保证良好的用户体验，需要对客户端的行为进行**精细化的控制**，比如：只等待 500ms，超过 500ms 仍然没有收到任何一个气象数据服务中心的响应，first 函数就返回失败。

在上面例子的基础上对 first 函数进行了调整：

```go
func first(servers ...*httptest.Server) (result, error) {
	c := make(chan result, len(servers))
	queryFunc := func(server *httptest.Server) {
		url := server.URL
		resp, err := http.Get(url)
		if err != nil {
			log.Printf("http get error: %s\n", err)
			return
		}
		defer resp.Body.Close()
		body, _ := ioutil.ReadAll(resp.Body)
		c <- result{
			value: string(body),
		}
	}
	for _, serv := range servers {
		go queryFunc(serv)
	}

	select {
	case r := <-c:
		return r, nil
	case <-time.After(500 * time.Millisecond): // 主要就是增加了这里的一个定时器
		return result{}, errors.New("timeout")
	}
}
```

```shell
$ go run main.go
2022/09/18 22:58:13 open-weather-2 receive a http request
2022/09/18 22:58:13 open-weather-3 receive a http request
2022/09/18 22:58:13 open-weather-1 receive a http request
2022/09/18 22:58:14 invoke first error: timeout
```

# 示例3 取消模式

加上了**超时模式**的版本依然有一个明显的问题，即便 first 函数因超时而返回，三个已经创建的 goroutine 可能依然处在向气象数据中心请求或等待应答的状态，没有返回，也没有被回收，资源依然在占用，即使它们的存在已经没有任何意义。一种合理的思路是让这三个 goroutine 支持取消操作，可以使用 Go 的 context 包来实现取消模式。

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"net/http/httptest"
	"time"
)

type result struct {
	value string
}

func first(servers ...*httptest.Server) (result, error) {
	c := make(chan result)
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	queryFunc := func(i int, server *httptest.Server) {
		url := server.URL
		req, err := http.NewRequest("GET", url, nil) // 不在使用http.Get，使用http.NewRequest可以创建一个带有context的req
		if err != nil {
			log.Printf("query goroutine-%d: http NewRequest error: %s\n", i, err)
			return
		}
		req = req.WithContext(ctx)

		log.Printf("query goroutine-%d: send request...\n", i)
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			log.Printf("query goroutine-%d: get return error: %s\n", i, err)
			return
		}
		log.Printf("query goroutine-%d: get response\n", i)
		defer resp.Body.Close()
		body, _ := ioutil.ReadAll(resp.Body)

		c <- result{
			value: string(body),
		}
		return
	}

	for i, serv := range servers {
		go queryFunc(i, serv)
	}

	select {
	case r := <-c: // 如果取到了一个最先的数据，函数进行返回，在返回前触发了defer中的cancel()函数对其他的goroutine进行取消操作
		return r, nil
	case <-time.After(500 * time.Millisecond):
		return result{}, errors.New("timeout")
	}
}

func fakeWeatherServer(name string, interval int) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Printf("%s receive a http request\n", name)
		time.Sleep(time.Duration(interval) * time.Millisecond)
		w.Write([]byte(name + ":ok"))
	}))
}

func main() {
	result, err := first(fakeWeatherServer("open-weather-1", 200), // 只有200的这个会被执行到
		fakeWeatherServer("open-weather-2", 1000),
		fakeWeatherServer("open-weather-3", 600))
	if err != nil {
		log.Println("invoke first error:", err)
		return
	}

	fmt.Println(result)
	time.Sleep(10 * time.Second)
}
```

```shell
$ go run main.go
2022/09/18 23:47:35 query goroutine-1: send request...
2022/09/18 23:47:35 query goroutine-2: send request...
2022/09/18 23:47:35 query goroutine-0: send request...
2022/09/18 23:47:35 open-weather-2 receive a http request
2022/09/18 23:47:35 open-weather-3 receive a http request
2022/09/18 23:47:35 open-weather-1 receive a http request
2022/09/18 23:47:35 query goroutine-0: get response
{open-weather-1:ok}
2022/09/18 23:47:35 query goroutine-1: get return error: Get "http://127.0.0.1:64178": context canceled
2022/09/18 23:47:35 query goroutine-2: get return error: Get "http://127.0.0.1:64179": context canceled
```