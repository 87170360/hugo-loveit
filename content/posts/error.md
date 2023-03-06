---
title: Golang的错误处理
date: 2021-08-16T10:18:00+08:00
tags: ["golang"]
series: ["Go进阶训练营"]
categories: ["technology"]
---

![](https://z3.ax1x.com/2021/08/18/fIfZ2F.png)

#### 错误与异常
在golang中错误与异常处理是区分很清晰的
错误：在预期内的非正常行为，开发人员可以对错误进行捕获和处理, 人为恢复错误
异常：非预期内的非正常行为，无法控制的错误，环境问题，栈溢出，goroutine异常等待，程序进入崩溃

Golang处理错误范式, 第一个返回值为正常值，第二返回值为错误，当error为nil时候，第一个返回值才可正常使用。
``` golang
func DoSomething() (int, error) {
    	
}
```

Golang捕获异常,如果不捕获程序就崩溃掉 
``` golang
func DoSomething() (int, error) {
    //直接制造一个异常
    panic("make panic)
}

func main() {
    defer func() {
    //使用recover捕获异常, 让程序继续往后执行
    if recover() != nil {
        fmt.Println("cache panic")     
    }
    DoSomething()
}
```

#### Go的error 接口
这个契约非常简单，支持返回一个字符串即可
``` golang
type error interface{
    Error() string
}
```

#### 错误处理方式
当函数返回了错误类型，我们要对错误类型进行不同处理，比如说最基础的打印日志

##### 处理方式1 Sentinel Error
这种方式最直观，预先定义一系列已知的错误类型，逐个判断属于那个错误即可
``` golang
//获取到了一个错误
_, err := DoSomething()
//错误与已知类型比较, 这个ErrSomething 是已经定义好的错误类型比如io.EOF，syscal.ENOENT
if err == ErrSomething {
}
```
特征：这样做的话, Sentinel Error会成为API公共部分，增加了API的表面积。引入了依赖问题
结论:避免Sentinel Error

##### 处理方式2 Error types
把错误的type断言出来,使用switch case 分类
``` golang 
//自定义一个错误
type MyError struct {
    Msg string
    File string
    Line int
}

func (e *MyError) Error() string {
    return fmt.Sprintf("%v:%v:%v", e.File, e.Line, e.Msg)
}

//获取一个自定义错误
func createError() error {
    return  &MyError{"something happened", "main.go", 10}
}

//main函数中通过断言应用
func main() {
    err := createError()
    //关键操作，获取error接口的具体类型
    switch err := err.(type) {
        case nil:
        case *MyError:
            //这里拿到了自定义的类型，可以进行更灵活操作，而不是限定于只能使用error接口的Error
            fmt.Println("get MyEroro type, line:%v", err.Line)
        default: 
    }
}
```
特征： 因为要使用类型断言，所有自定义的error类型必须变成public的。引入了依赖问题
结论：避免作为公共API使用

##### 处理方式3 Opaque errors
不透明错误处理, 不解析错误的具体类型，根据行为接口来处理
Assert errors for behaviour, not type
``` golang 
//自定义的错误类型的MyError, 添加一个行为接口Temporary
package other

import "fmt"

type MyError struct {
	Msg string
}

func (e *MyError) Error() string {
	return fmt.Sprintf("msg:%v", e.Msg)
}

//实现一个特别的接口
func (e *MyError) Temporary() bool {
	return true
}
```

创建MyError的函数
``` golang
func CreateMyError() error {
	return &MyError{Msg: "something"}
}
```

main中使用
``` golang
import (
	"err/other"
	"fmt"
	"io"
)
// 定义一个接口,定义一个函数Temporary跟MyError一样
type MyTemporary interface {
	Temporary() bool
}

func main() {
    //err1 内部实现了Temporary()满足了MyTemporary接口
	err1 := other.CreateMyError()
	te, ok := err1.(MyTemporary)
	fmt.Printf("err1 has Temporary:%v, Temporary():%v\n", ok,te.Temporary())

    //err2 不满足MyTemporary接口
	err2 := io.EOF
	_, ok2 := err2.(MyTemporary)
	fmt.Printf("err2 has Temporary:%v\n", ok2)
}
```
output:
``` shell
err1 has Temporary:true, Temporary():true
err2 has Temporary:false
```
特点：在main中本地定义了MyTemporary接口，可以用来判断MyError中是否定义了Tempporary(), 这避免了依赖. main中也不关系错误具体是什么类型，只关心行为

### Handing Error
#### Indented flow is for errors
代码规范，处理错误的代码缩进
比如
``` golang 
if err != nil {
    //handler error 处理错误缩进
}
//do stuff 正常代码不缩进
```
不规范展示
``` golang 
if err == nil {
    // do stuff
}
//handle error
```

#### Eliminate error handing by elimination errors
处理错误的不正确方式，消除处理的方式来消除错误，跟杀掉花刺子信使的方式来处理坏消息一样
``` golang 
func AuthenticateRequest(r *Request) error {
    return authenticate(r.User)
}
```
直接把错误上抛，导致上层处理不了错误，底层返回了一个找不到文件错误，调用层就很莫名其妙，起码要告诉我是什么文件，或者那行代码错误吧。
产生了错误，但是没用添加足够的信息返回上层，那么这个错误就没意义。

正确姿势，应该使用errors.Wrap添加信息返回去, 返回结果可以通过errors.Cause取原生错误，也可以取调用堆栈
``` golang 
func ReadFile(path string) error {
	f, err := os.Open(path)
	if err != nil {
        //在原有错误情况下，添加一点信息"open failed",再返回出去
		return errors.Wrap(err, "open failed")
	}
	defer f.Close()
	return nil
}

func main() {
	err := ReadFile("aa")
	if err != nil {
	    //显示添加的信息和原错误信息
		fmt.Println(err)
		//显示原生错误类型和信息
		fmt.Printf("origin error: type:%T, value:%v\n", errors.Cause(err), errors.Cause(err))
		//打印堆栈
		fmt.Printf("stack %+v\n", err)
	}
}

//output
open failed: open aa: The system cannot find the file specified.

origin error: type:*os.PathError, value:open aa: The system cannot find the file specified.

stack open aa: The system cannot find the file specified.
open failed
main.ReadFile
	E:/test/main.go:30
main.main
	E:/test/main.go:13
runtime.main
	C:/Program Files/Go/src/runtime/proc.go:204
runtime.goexit
	C:/Program Files/Go/src/runtime/asm_amd64.s:1374

```
errors.WithMessage功能类似errors.Wrap, 但是不能打印调用堆栈
``` golang 
func ReadFile2(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return errors.WithMessage(err, "some message")
	}
	defer f.Close()
	return nil
}

func main() {
	err = ReadFile2("bb")
	if err != nil {
		fmt.Println(err)
		fmt.Printf("origin error: type:%T, value:%v\n", errors.Cause(err), errors.Cause(err))
		//打印了也没有调用堆栈
		fmt.Printf("stack %+v\n", err)
	}
}

//output
some message: open bb: The system cannot find the file specified.

origin error: type:*os.PathError, value:open bb: The system cannot find the file specified.

stack open bb: The system cannot find the file specified.
some message
```
不在每个错误产生的地方导出打印日志
通过使用errors.Wrap把本地错误信息添加，在最上层调用是%+v打印堆栈详情
错误处理总结：.当函数调用产生了错误，要么本地处理掉错误，返回降级数据，要么添加上下文信息，通过wrap返回给调用方

##### 错误包装和拆包
上述的errors.wrap就是一个go官方定义的错误包装，通过Cause进行拆包，这也可以自定义包装和拆包
在自定义错误类型实现Unwrap接口，就可以使用errors的Is和As函数

``` golang 
func main() {
	err := CreateQueryError()
	//Is依赖于QueryError实现了Unwrap函数
	if errors.Is(err, io.ErrClosedPipe) {
		fmt.Println("Is")
	}

	var e *QueryError
	if errors.As(err, &e) {
		fmt.Println("As")
	}
}

func CreateQueryError() error {
	return &QueryError{
		Query: "select * from user",
		Err:   io.ErrClosedPipe,
	}
}

type QueryError struct {
	Query string
	Err error
}

func (e *QueryError) Error() string {
	return "QueryError"
}

func (e *QueryError) Unwrap() error {
	return e.Err
}

//output
Is
As
```

前面这种要自定义一个错误类型进行wrap， 还有一种更快捷的方式使用%w, 返回的error带了Unwrap方法，返回%w的参数
``` golang 
if err != nil {
    // Return an error which unwraps to err.
    return fmt.Errorf("decompress %v: %w", name, err)
}
```
同样支持error.Is, error.As
``` golang 
err := fmt.Errorf("access denied: %w", ErrPermission)
...
if errors.Is(err, ErrPermission) ...
```
