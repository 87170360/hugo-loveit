---
title: SingleFlight代码阅读和使用
date: 2021-://nexusgames.to/wp-content/uploads/2020/08/Microsoft-Flight-Simulator-Free-Download-By-Nexusgames.to-2-970x570.jpg07-20T6:00:00+08:00
tags: ["golang"]
series: ["Go进阶训练营"]
categories: ["technology"]
---

![](https://nexusgames.to/wp-content/uploads/2020/08/Microsoft-Flight-Simulator-Free-Download-By-Nexusgames.to-2-970x570.jpg)

### 为什么叫SingleFlight
字面直译是单飞，举例：学校宿舍里一群同学要下载同一份资料，每个人都自己进行下载，导致网络很卡每个人都要等很久
这时候，班长说，你们不用下载了，我一个人下载好了再发给你们。这样就节约了大家的时间和网络资源。

### SingleFlight 使用场景
把上述的同学下载资料替换成客户端请求服务器资源，就明白了使用场景，对服务器来说短时间内来了大量相同的客户端请求，每条请求都单独处理太消耗服务器资源，容易造成宕机事故。SingleFlight把多个相同请求合并到一起，服务器只处理一个请求即可，大大提高的服务器的效率

源码阅读意外收获
1. 错误处理
普通的错误和panic错误，分别处理，panic的错误打印调用堆栈
2. 解锁操作并没有无脑的使用defer unlock, 而是尽快解锁，避免一切无谓处理
3. 对panic部分还不了解，实际测试效果也没有区别
``` golang
    if e, ok := c.err.(*panicError); ok {
        // In order to prevent the waiting channels from being blocked forever,
        // needs to ensure that this panic cannot be recovered.
        if len(c.chans) > 0 {
            //迷之代码, 不使用go也不使用select{}也没有效果区别
            go panic(e)
            select {} // Keep this goroutine around so that it will appear in the crash dump.
        } else {
            panic(e)
        }
    }
```

全部源码阅读开始

``` golang
// Package singleflight provides a duplicate function call suppression
// mechanism. 重复函数调用抑制机制，(singleflight真正的含义)
package singleflight // import "golang.org/x/sync/singleflight"

import (
	"bytes"
	"errors"
	"fmt"
	"runtime"
	"runtime/debug"
	"sync"
)

// errGoexit indicates the runtime.Goexit was called in
// the user given function. 
// 定义goroutine退出错误
var errGoexit = errors.New("runtime.Goexit was called")

// A panicError is an arbitrary value recovered from a panic
// with the stack trace during the execution of given function.
// 定义一个panic错误，包含调用堆栈信息
type panicError struct {
	value interface{}
	stack []byte
}

// Error implements error interface.
// 调用堆栈格式化
func (p *panicError) Error() string {
	return fmt.Sprintf("%v\n\n%s", p.value, p.stack)
}

// 定义创建panicError方法，放置初始化处理
func newPanicError(v interface{}) error {
    // 获取堆栈信息
	stack := debug.Stack()

	// The first line of the stack trace is of the form "goroutine N [status]:"
	// but by the time the panic reaches Do the goroutine may no longer exist
	// and its status will have changed. Trim out the misleading line.
	// 因为如果panic时候，goroutine已经不存在，第一行堆栈信息已经失效, 删除这行信息
	if line := bytes.IndexByte(stack[:], '\n'); line >= 0 {
	    // 找出第一行结尾，删除
		stack = stack[line+1:]
	}
	return &panicError{value: v, stack: stack}
}

// call is an in-flight or completed singleflight.Do call
type call struct {
    //同类的调用每个是独立的goroutine, 这里使用WaitGroup统一管理
	wg sync.WaitGroup

	// These fields are written once before the WaitGroup is done
	// and are only read after the WaitGroup is done.
	val interface{}
	err error

	// forgotten indicates whether Forget was called with this call's key
	// while the call was still in flight.
	//用于标记调用在进行中的时候，发生了取消调用行为
	forgotten bool

	// These fields are read and written with the singleflight
	// mutex held before the WaitGroup is done, and are read but
	// not written after the WaitGroup is done.
	//标记该类请求次数
	dups  int
	//存放处理结果
	chans []chan<- Result
}

// Group represents a class of work and forms a namespace in
// which units of work can be executed with duplicate suppression.
//Group 抽象了一类工作任务，并存储了被抑制了的运行任务
type Group struct {
	mu sync.Mutex       // protects m
	//操作m的锁
	m  map[string]*call // lazily initialized
	//使用时候才开始初始化
}

// Result holds the results of Do, so they can be passed
// on a channel.
//定义返回结果数据格式
type Result struct {
	Val    interface{}
	Err    error
	Shared bool
}

// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
// The return value shared indicates whether v was given to multiple callers.
// 单飞启动入口，给定的key表示一类请求，当key相同的重复调用发生，重复调用回阻塞，等第一个飞行中(in-flight)的操作返回后，一起返回相同结果
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    //并发安全加锁
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		//只锁最小范围
		g.mu.Unlock()
		//阻塞调用者的goroutine
		c.wg.Wait()

        //处理错误， 这个是用断言来判断自定义的错误类型
		if e, ok := c.err.(*panicError); ok {
			panic(e)
		} else if c.err == errGoexit {
		    //当前goroutine运行终止
			runtime.Goexit()
		}
		return c.val, c.err, true
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}

// DoChan is like Do but returns a channel that will receive the
// results when they are ready.
//
// The returned channel will not be closed.
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
	ch := make(chan Result, 1)
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		c.chans = append(c.chans, ch)
		g.mu.Unlock()
		//返回ch给调用者，让调用方自己控制阻塞
		return ch
	}
	c := &call{chans: []chan<- Result{ch}}
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

    //单独开一个goroutine处理
	go g.doCall(c, key, fn)

	return ch
}

// doCall handles the single call for a key.
//执行函数调用
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	normalReturn := false
	recovered := false

	// use double-defer to distinguish panic from runtime.Goexit,
	// more details see https://golang.org/cl/134395
	defer func() {
		// the given function invoked runtime.Goexit
		if !normalReturn && !recovered {
			c.err = errGoexit
		}

		c.wg.Done()
		g.mu.Lock()
		defer g.mu.Unlock()
		if !c.forgotten {
			delete(g.m, key)
		}

		if e, ok := c.err.(*panicError); ok {
			// In order to prevent the waiting channels from being blocked forever,
			// needs to ensure that this panic cannot be recovered.
			// 怎么保证其他等待的channels不会blocker forever
			if len(c.chans) > 0 {
				//谜之代码, 经过测试这里不用go也没区别
				go panic(e)
				//这里不用select{}也没区别
				select {} // Keep this goroutine around so that it will appear in the crash dump.
			} else {
				panic(e)
			}
		} else if c.err == errGoexit {
			// Already in the process of goexit, no need to call again
		} else {
			// Normal return
			//正常退出，通知等待的chan退出和给出结果
			for _, ch := range c.chans {
				ch <- Result{c.val, c.err, c.dups > 0}
			}
		}
	}()

    //定义一个匿名函数并执行
	func() {
		defer func() {
			if !normalReturn {
				// Ideally, we would wait to take a stack trace until we've determined
				// whether this is a panic or a runtime.Goexit.
				//
				// Unfortunately, the only way we can distinguish the two is to see
				// whether the recover stopped the goroutine from terminating, and by
				// the time we know that, the part of the stack trace relevant to the
				// panic has been discarded.
				if r := recover(); r != nil {
					c.err = newPanicError(r)
				}
			}
		}()

		c.val, c.err = fn()
		//这个fn如果panic, 由上面的recover()进行恢复
		normalReturn = true
		//能执行到这里说fn已经正常返回，上述recover就判断normalReturn退出
	}()

	if !normalReturn {
		recovered = true
	}
}

// Forget tells the singleflight to forget about a key.  Future calls
// to Do for this key will call the function rather than waiting for
// an earlier call to complete.
// 通知singleflight取消一个key类型的调用
func (g *Group) Forget(key string) {
	g.mu.Lock()
	if c, ok := g.m[key]; ok {
		c.forgotten = true
	}
	delete(g.m, key)
	g.mu.Unlock()
}
```

Do和DoChan的区别：
Do内部进行阻塞, DoChan内部不阻塞，返回chan给调用者继续阻塞

###### 使用测试
模拟10个并发任务发生，使用singleflight和普通处理的区别

``` golang
package main

import (
	"fmt"
	"github.com/sync/singleflight"
	"sync"
	"time"
)

func main() {
	normal()
	//doTest()
	//doChanTest()
}

//不使用singleflight处理
func normal() {
	var wg sync.WaitGroup
	for i := 0; i <= 10; i++ {
		wg.Add(1)
		go func(index int) {
			//fmt.Println("goroutine start", index)
			//这个work会调用10次
			work()
			//fmt.Println("goroutine end", index)
			wg.Done()
		}(i)
	}

	wg.Wait()
}

//使用gourp.Do
func doTest() {
	var wg sync.WaitGroup
	group := singleflight.Group{}
	for i := 0; i <= 10; i++ {
		wg.Add(1)
		go func(index int) {
			//fmt.Println("goroutine start", index)
			//work只会执行一次
			group.Do("some", work)
			//fmt.Println("goroutine end", index)
			wg.Done()
		}(i)
	}

	wg.Wait()
}

//使用gourp.DoChan
func doChanTest() {
	var wg sync.WaitGroup
	group := singleflight.Group{}
	for i := 0; i <= 10; i++ {
		wg.Add(1)
		go func(index int) {
			//fmt.Println("goroutine start", index)
			//work只会执行一次
			ch := group.DoChan("some", work)
			<- ch
			//fmt.Println("goroutine end", index)
			wg.Done()
		}(i)
	}

	wg.Wait()
}

//模拟进行一些耗时操作
func work() (interface{}, error) {
	fmt.Println("work do")
	time.Sleep(time.Millisecond * 100)
	//panic("make panic")
	return nil, nil
}

```

nomal()的打印结果，work被重复调用了10次
``` bash
work do
work do
work do
work do
work do
work do
work do
work do
work do
work do
work do
```

使用singleflght的优化， work只被执行了一次
``` bash
work do
```
