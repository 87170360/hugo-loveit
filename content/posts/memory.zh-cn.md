---
title: Go内存原理
date: 2021-07-29T10:18:00+08:00
tags: ["golang"]
series: ["Go进阶训练营"]
categories: ["technology"]
---

![](https://z3.ax1x.com/2021/07/29/WHSuM6.jpg)

#### 堆和栈的定义
这是内存中最基本的两个定义， 堆和栈都是存放数据，区别是堆是全局范围可以使用，在整个程序生命周期内有效。栈的有效范围和生命周期比较小，一般仅限于函数的作用范围。
C++中栈和堆从语法上明确定义的，栈是明确通过new来定义的，堆的数据不会跑到栈，但是Go的栈不需要明确new定义，堆内的数据也可能逃逸到堆上。Go设计者明显更注重开发效率
C++中栈上分配的内存，开发人员必须自己负责回收， Go存在Runtime GC来处理

栈： 
栈区的内存一般有编译器自动进行释放和分配，随着函数创建而创建， 函数的返回而销毁。
函数可以直接访问栈内的内存，间接访问栈外内存

堆：
堆上分配由编译器和开发人员共同进行，堆上数据释放通过GC处理。在Go中只要变量被分享到函数作用域之外，那么就整个变量就会在栈上分配。
堆分配需要一块足够大的内存

从分配和回收来考虑，栈比堆都更消耗性能，所有stack allocation is cheap and heap allocation is expensive
Go编译器在函数内的变量分配到栈中，如果在函数返回后无法证明变量未被引用，则分配到对堆上。
从程序运行正确角度看， Go中变量存储位置是堆还是栈与语法无关，都能正确运行。
但是从程序高性能角度看，如果局部变量很大，应该放在堆上避免频繁创建和回收， 函数返回时候尽量少返回指针，大部分变量都应该在栈内，函数退出直接回收。

#### 逃逸分析
检查变量作用域超出所在栈
``` golang
go build-gcflags '-m'
```
逃逸案例
1 函数返回值是指针
``` golang 
func main() {
    num := getRandom()
    println(*num)
    
    
}

func getRandom() *int {
    tmp := rand.Intn(100)
    return &tmp
}
```
为什么会发生逃逸， 在getRandom中tmp如果是在分配在栈中，那么函数调用完后，tmp指向的内存就回收掉了，main函数中拿到的就是tmp指向的就是无效的内存地址。如果是C++中这种情况就是野指针，直接崩掉。
Go中就友好的多，编译器避免出现无效的内存地址，就直接把tmp的内存分配在堆中，getRandom回收掉不影响, main函数也能正常访问tmp的内存地址。

2 引用类型对象赋值
``` golang
A = B
```
如果A是引用数据类型会导致逃逸, Go中引用类型数据由 func, interface{}, slice, map, chan, *Type
3 for循环外声明，循环内分配
4 指针或带指针的值发生到channel中
5 在slice中存储指针或带指针的值
6 slice重新分配
7 在interface类型上调用方法

#### 分段栈(Segmented stacks) Go v1.0-1.2
Go应用程序运行时， 每个goroutine都维护一个自己的栈区，自己的栈区不能被其他goroutine使用，v1.4后最小栈区内存降低到2KB, 栈区最大值64位系统默认是1G
分段栈实现：在扩容的时候分配一块新的内存并链接到老的栈内存块 ，新旧两块内存是分开的，所有叫分段栈
![](https://z3.ax1x.com/2021/07/29/WqKvKf.png)

"hot split": 如果在栈快满的情况下，下一次函数调用就触发栈扩容， 函数返回时候新分配的"stack chunk"会被清理掉，如果这个情况刚好发生在循环中，就会导致频繁alloc/free，严重影响性能
Gov1.2采用把最低栈默认改成8KB降低触发热分裂问题发生概率

#### 连续栈（Contiguous status）Go v1.3-1.4
为了解决上述的热分裂问题，使用连续栈。当触发栈扩容的时候，分配一个原来两倍大的内存块，并把老的内存块内容复制到新的内存块。思路与slice，c++的vector扩容相似
那么栈区空间有扩容就有收缩， 如果使用率不超过1/4，垃圾回收时候进行栈缩容
![](https://z3.ax1x.com/2021/07/31/WjHL11.png)


#### 内存管理
##### 内存碎片
随着内存不断申请和释放， 内存上会存在大量碎片， 需要将2个连续未使用的内存块合并，减少碎片
![内存碎片产生](https://z3.ax1x.com/2021/08/02/f9jC79.png)
这是因为不同大小的变量分配一块连续的内存
解决方法: Slab-Allactor 将相同规格的变量分配在一起，减少碎片产生和方便碎片回收, 类似的思路在Kernal, Memcache 都可以见
##### 大锁
同一进程下的所有线程共享相同的内存空间，它们申请内存时候需要加锁
##### 内存布局
##### page
内存页，一块8K大小的内存空间。Go与操作系统之间的内存申请和释放，都是以page为单位的
##### span
内存块，一个或者多个连续的page组成一个span
##### object
对象， 用来存储一个变量的内存空间
##### sizeclass
空间规格， 每个span带一个sizeclass, 标记span的page应该如何使用, 通过sizeclass标记本span的page是通过什么规格进行切割，每一个切割内容的数量就是一个object
比如说按32Byte划分一个span, 用来存储16~32Byte的变量
![](https://z3.ax1x.com/2021/08/02/fCSuxP.png)
不同span的szieclass下能装的ojbect数量
蓝色图例：sizeclass=K，格子数 64    
绿色图例：sizeclass=K+1，格子数 16    
蓝色图例：sizeclass=K+2，格子数 8    
![](https://z3.ax1x.com/2021/08/02/fCSra4.jpg)

#### Go runtime的内存模型
在Go调度模型中P绑定一个本地缓存mcache, 当需要进行内存分配时候，当前的goroutine从mcache中查找可以用的mspan, 因为是本地专用内存，所以不需要加锁.
每个mspan代表一类规格大小，总共重8b到32k类mspan
![](https://z3.ax1x.com/2021/08/03/fCORXD.png)
#### 小于32kb内存分配
##### mcentral
优先从本地mcache分配内存，每次都那一个mspan， 如果本地没有对口的sizeclass就从其他地方拿， 那么拿？ Go为每个sizeclass的mspan维护者一个mcentral
![](https://z3.ax1x.com/2021/08/03/fCzExx.png)
mcentral 内部有两个双向链表，分别表示空闲的mspan和占用的mspan
申请内存过程： mcache已经没有空闲mspan, 那么工作线程去mcentral里申请 
获取： 加锁，从mcentral的nonempty链表中找到一个可用的mspan，并将其从链表中删除，取出的mspan放入mcache的empty链表， 将mspan给工作线程来用，解锁
当mcentral没有空闲的mspan时候，会向mheap申请， mheap没有资源就向系统申请新内存。
申请路径: mcache -> mcentral -> mheap

#### 大于32kb内存
超过32kb的内存申请，直接从堆上分配对应数量的内存页（每页8K）

#### 内存分配组成图
![](https://z3.ax1x.com/2021/08/05/fe3UaR.png)
总结
内存管理组件
mcache  管理线程本地缓存的mspan
mcentral 管理全局的mspan 给所以线程
mheap 从系统分配内存， 大对象也直接从这里分配 
mspan 基本单元， 每种类型分配特定大小object

### GC原理
#### Garbage Collection
C, C++等早期编程语言需要工程师手动管理内存，工程师管理等当，内存管理效率很高，但是这样就影响了开发效率。
后续出来的编程语言php,java,golang都出现了垃圾回收机制，语言内部进行统一管理内存分配和释放。

#### 主流GC算法
1. 引用计数
2. 追踪式垃圾回收, Golang使用的三色标记法属于这种算法实现的一种

#### Mark&Sweep 标记清除法
在三色标记法之前，使用的一种算法
过程
1. STW(stop the world), 停止运转避免引用关系变化，这是GC算法的优化重点
2. 从Root对象开始追踪其他存活对象，进行标记
(Root根对象，可以直接访问的对象，比如全局对象，栈对象中的数据。通过Root对象追踪其他存活的对象)
3. 对堆对象进行迭代，已经标记的对象置位，未标记对象加入freelist, 可用于再分配
4. Start the World
最种朴素版本的在进行Mark或者Sweep时候都需要完全暂停,所以暂停时间太长 

通过Root追踪并标记存活的对象
![](https://z3.ax1x.com/2021/08/06/fuc1v8.png)

#### 三色标记法
三色标记法，缩短了暂停时间
简单过程
1. 所有对象默认标记为白色
2. 根对象标记为灰色
3. 从灰色追踪到的对象标记为灰色，原来的灰色标记为黑色，重复进行本步骤，直到没有灰色对象为止
4. 删除白色标记
为什么比Mark&Sweep有更短的暂停时间？
仅需要在标记前后需要STW,使用了写屏障

#### GC触发条件
1. GC Percentage配置选项，默认配置100，（比如当前使用heap内存5M, 当内存扩展了100%为10M时触发GC）
2. 超过2分钟没有触发GC

