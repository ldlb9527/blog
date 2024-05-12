+++
author = "旅店老板"
title = "Golang中channel的基本使用和原理学习"
date = "2024-05-01"
description = "golang管道原理学习"
tags = [
	"golang","channel",
]
categories = [
    "golang","channel",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "golang.png"
mermaid = true
+++
##创建channel的方式
### 缓冲型通道语法：`c := make(chan 数据类型, 通道大小)`
```go
ch := make(chan int, 1024)
```
### 非缓冲型通道语法：`c := make(chan 数据类型)`
```go
var ch chan string
ch = make(chan string)
```
* channel必须make后才能使用
## channel的基本操作
### 向channel写入数据
```go
ch := make(chan int,3)
ch <- 100
```
### 获取channel的length和cap
```go
ch := make(chan int,3)
ch <- 100
fmt.Println(len(ch)) //1
fmt.Println(cap(ch)) //3
```
* 通过`len()`获取channel已使用的长度，上面向channel写入了一个元素，所以打印为1
* 通过`cap()`获取channel的容量，即缓冲区的大小，打印为3，channel的大小在我们make初始化后就不能改变了
* 只有写入时，已使用长度超过容量时会产生死锁异常，代码如下：
```go
ch := make(chan int,3)
ch <- 100
ch <- 100
ch <- 100
ch <- 100 // fatal error: all goroutines are asleep - deadlock!
```
### 从channel读取数据
```go
ch := make(chan int,3)
ch <- 100
num := <-ch
fmt.Println(num) //100
```
* 我们写入了一个100，num为读取的值，打印为100
* 只有读取时，向已使用长度为0的channel读取会产生死锁异常
```go
ch := make(chan int,3)
ch <- 100

num := <-ch
fmt.Println(num) //100

num1 := <-ch  //fatal error: all goroutines are asleep - deadlock!
fmt.Println(num1)
```
* channel的读取遵循队列的FIFO原则
### channel的关闭
* 使用`close()`可以关闭channel。当channel关闭后，就不能再向该channel写入数据,否则会产生死锁异常
```go
ch := make(chan int)
ch <- 100
close(ch)
ch <- 100 // fatal error: all goroutines are asleep - deadlock!
```
* 当channel关闭后，仍然可以从channel读取数据，所有元素读取完后，后续仍可读取，读取的值为该类型的零值
```go
ch := make(chan int, 3)
ch <- 100
ch <- 90
close(ch)

num := <-ch
num1 := <-ch
num2 := <-ch
num3 := <-ch

fmt.Println(num)  //100
fmt.Println(num1) //90
fmt.Println(num2) //0
fmt.Println(num3) //0
```
### channel的遍历
* channel支持for-range的方式进行遍历
* 在遍历时，如果channel没有关闭，则会出现死锁异常
* 在遍历时，如果channel已经关闭，遍历完成后会正常退出循环
```go
ch := make(chan int, 3)
ch <- 100
ch <- 90h
close(ch)

for num := range ch {
	fmt.Println(num)//打印100 90后退出循环
}
```
* 如有两个协程分别对channel进行一直读和写，写入速率快时，写入操作会发生阻塞；读取速率快时，读取操作会发生阻塞
## channel的源码分析
`chan`是一个语法糖，它的数据结构在gosdk的`src/runtime/chan.go`中，定义如下：
```go
type hchan struct {
	qcount   uint           // buffer中已放入的元素个数
	dataqsiz uint           // 用户构造channel时指定的buf大小,即channel的容量
	buf      unsafe.Pointer // 指向buffer的指针
	elemsize uint16         //buffer中每个元素的大小
	closed   uint32         //channel是否关闭，== 0代表未closed
	elemtype *_type // channel元素的类型信息
	sendx    uint   // buffer中已发送的索引位置
	recvx    uint   // buffer中已接收的索引位置
	recvq    waitq  // 等待接收的goroutine队列
	sendq    waitq  // 等待发送的goroutine队列
	lock mutex      //锁
}
```
已阻塞等待发送或接受的协程队列类型为`waitq`结构体，它的数据结构如下：
```go
type waitq struct {
	first *sudog
	last  *sudog
}
```
waitq是一个队列,实质是一个双向链表，`first`指向第一个元素,`last`指向最后一个元素，它们都是`sudog`类型的结构体指针,数据结构如下所示：
```go
type sudog struct {
	g *g //存储goroutine
	next *sudog //后继节点，指向下一个sudog
	prev *sudog //前驱节点，指向上一个sudog
	elem unsafe.Pointer // data element (may point to stack)
	acquiretime int64
	releasetime int64
	ticket      uint32
	isSelect bool
	success bool
	parent   *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c        *hchan // channel
}
```
原来是将我们的发送或接受阻塞的goroutine包装成一个sudog，存入发送等待队列(sendq)或接受等待队列(recvq)
***
通常我们们使用`ch <- x`这种方式向channel中写入数据，实际调用的函数为`chansend1`，其源码如下：
```go
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}
```
c为channel,elem为需要写入的元素,true表示需要阻塞，`chansend`源码如下：
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}
	
	if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```
该函数较长，我们对代码进行分块分析：
```go
        if c == nil {
            if !block {
                return false
            }
            gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
            throw("unreachable")
        }
```
* 当channel为nil且非阻塞写入时直接返回false,`var ch chan int`这样未初始化的channel为nil
* gopark函数的作用是挂起当前协程，c为nil，永远不会有读操作，即该协程永远不会被唤醒，会报死锁错误，即**向一个为nil的channel阻塞写会发生死锁错误**,与我们上一节的调试结果相符合
* throw("unreachable")不会被执行到
```go
        if debugChan {
            print("chansend: chan=", c, "\n")
        }
    
        if raceenabled {
            racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
        }
```
* `debugChan`和`raceenabled`默认值都为false，
```go
	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
````
`c.closed`是临界资源，需要加锁操作。xian`c.closed != 0`表示该channel已被关闭，**向一个已关闭的channel写操作时，会抛出panic异常：`send on closed channel`**
，