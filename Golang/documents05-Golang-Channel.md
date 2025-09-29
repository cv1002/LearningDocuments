# Channel
- [Channel](#channel)
  - [channel的基本概念](#channel的基本概念)
  - [channel的用法](#channel的用法)
    - [发送操作](#发送操作)
    - [接收操作](#接收操作)
    - [关闭操作](#关闭操作)
  - [channel的特性](#channel的特性)
    - [线程安全](#线程安全)
    - [阻塞式发送和接收](#阻塞式发送和接收)
    - [顺序性](#顺序性)
    - [可以关闭](#可以关闭)
    - [缓冲区大小](#缓冲区大小)
  - [channel 数据结构](#channel-数据结构)
    - [hchan](#hchan)
    - [waitq](#waitq)
    - [sudog](#sudog)
  - [channel 底层实现逻辑](#channel-底层实现逻辑)
    - [通道创建](#通道创建)
    - [通道写入](#通道写入)
      - [写时有阻塞读协程](#写时有阻塞读协程)
      - [写时无阻塞读协程但环形缓冲区仍有空间](#写时无阻塞读协程但环形缓冲区仍有空间)
      - [写时无阻塞读协程且环形缓冲区无空间](#写时无阻塞读协程且环形缓冲区无空间)
      - [整体写流程](#整体写流程)
    - [通道读](#通道读)
      - [2种异常情况处理](#2种异常情况处理)
      - [读时有阻塞写协程](#读时有阻塞写协程)
      - [读时无阻塞写协程且缓冲区有数据](#读时无阻塞写协程且缓冲区有数据)
      - [读时无阻塞写协程且缓冲区无数据](#读时无阻塞写协程且缓冲区无数据)
      - [整体读流程](#整体读流程)
    - [通道关闭](#通道关闭)
  - [高频面试题](#高频面试题)

## channel的基本概念
在Go语言中，channel是一种类型，它可以用来在协程之间传递数据。channel类型的定义如下：

```go
chan T
```

其中T表示channel中可以传递的数据类型。例如，如果我们想要创建一个channel，用来传递字符串类型的数据，可以这样定义：

```go
ch := make(chan string)
```

这样就创建了一个可以传递字符串类型数据的channel。我们可以把channel看作是一个管道，数据从一个协程流入管道，再从管道流出到另一个协程中。在Go语言中，我们可以使用channel来实现协程之间的同步和通信。

## channel的用法
Go语言中的channel有三种基本操作：发送、接收、关闭。下面我们将详细介绍这三种操作的用法。

### 发送操作
发送操作用于将数据发送到channel中。发送操作的语法如下：

```go
ch <- x
```

其中ch表示要发送数据的channel，x表示要发送的数据。例如，下面的代码将字符串"hello"发送到了ch这个channel中：

```go
ch <- "hello"
```

注意，如果channel已经满了，发送操作会被阻塞，直到有其他协程从channel中取走了数据。这样可以保证在channel已经没有空间存放数据的情况下，发送操作不会丢失数据。

### 接收操作
接收操作用于从channel中取出数据。接收操作的语法如下：

```go
x := <- ch
```

其中ch表示要接收数据的channel，x表示接收到的数据。例如，下面的代码从ch这个channel中取出了一个字符串：

```go
x := <- ch
```

注意，如果channel中没有数据可供接收，接收操作会被阻塞，直到有其他协程向channel中发送了数据。这样可以保证在channel中没有数据可供接收的情况下，接收操作不会返回错误。

### 关闭操作
关闭操作用于关闭channel。关闭一个channel之后，就不能再向它发送数据了，但是仍然可以从它接收数据。关闭操作的语法如下：

```go
close(ch)
```

其中ch表示要关闭的channel。例如，下面的代码关闭了ch这个channel：

```go
close(ch)
```

注意，如果一个channel已经被关闭，再向它发送数据会导致panic错误。

## channel的特性
Go语言中的channel具有以下几个特性：

### 线程安全
channel是线程安全的，多个协程可以同时读写一个channel，而不会发生数据竞争的问题。这是因为Go语言中的channel内部实现了锁机制，保证了多个协程之间对channel的访问是安全的。

### 阻塞式发送和接收
当一个协程向一个channel发送数据时，如果channel已经满了，发送操作会被阻塞，直到有其他协程从channel中取走了数据。同样地，当一个协程从一个channel中接收数据时，如果channel中没有数据可供接收，接收操作会被阻塞，直到有其他协程向channel中发送了数据。这种阻塞式的机制可以保证协程之间的同步和通信。

### 顺序性
通过channel发送的数据是按照发送的顺序进行排列的。也就是说，如果协程A先向channel中发送了数据x，而协程B再向channel中发送了数据y，那么从channel中接收数据时，先接收到的一定是x，后接收到的一定是y。

### 可以关闭
通过关闭channel可以通知其他协程这个channel已经不再使用了。关闭一个channel之后，其他协程仍然可以从中接收数据，但是不能再向其中发送数据了。关闭channel的操作可以避免内存泄漏等问题。

### 缓冲区大小
channel可以带有一个缓冲区，用于存储一定量的数据。如果缓冲区已经满了，发送操作会被阻塞，直到有其他协程从channel中取走了数据；如果缓冲区已经空了，接收操作会被阻塞，直到有其他协程向channel中发送了数据。缓冲区的大小可以在创建channel时指定，例如：

```go
ch := make(chan int, 10)
```

这样就创建了一个带有10个缓冲区的channel。

## channel 数据结构
channel涉及到的核心数据结构包含3个。

### hchan

```go
// channel
type hchan struct {
    // 循环队列
    qcount   uint           // 通道中数据个数
    dataqsiz uint           // buf长度
    buf      unsafe.Pointer // 数组指针
    sendx    uint   // send index
    recvx    uint   // receive index
    elemsize uint16 // 元素大小
    elemtype *_type // 元素类型
    
    closed   uint32 // 通道关闭标志
    
    recvq    waitq  // 由双向链表实现的recv waiters队列
    sendq    waitq  // 由双向链表实现的send waiters队列
    lock mutex
}
```

hchan是channel底层的数据结构，其核心是由数组实现的一个环形缓冲区：

qcount 通道中数据个数

dataqsiz 数组长度

buf 指向数组的指针，数组中存储往channel发送的数据

sendx 发送元素到数组的index

recvx 从数组中接收元素的index

elemsize channel中元素类型的大小

elemtype channel中的元素类型

closed 通道关闭标志

recvq 因读取channel而陷入阻塞的协程等待队列

sendq 因发送channel而陷入阻塞的协程等待队列

lock 锁

### waitq

```go
// 等待队列(双向链表)
type waitq struct {
    first *sudog
    last  *sudog
}
```

waitq是因读写channel而陷入阻塞的协程等待队列。

first 队列头部

last 队列尾部

### sudog

```go
// sudog represents a g in a wait list, such as for sending/receiving
// on a channel.
type sudog struct {
    g *g // 等待send或recv的协程g
    next *sudog // 等待队列下一个结点next
    prev *sudog // 等待队列前一个结点prev
    elem unsafe.Pointer // data element (may point to stack)
    
    success bool // 标记协程g被唤醒是因为数据传递(true)还是channel被关闭(false)
    c        *hchan // channel
}
```

sudog是协程等待队列的节点：

g 因读写而陷入阻塞的协程

next 等待队列下一个节点

prev 等待队列前一个节点

elem 对于写channel，表示需要发送到channel的数据指针；对于读channel，表示需要被赋值的数据指针。

success 标记协程被唤醒是因为数据传递(true)还是channel被关闭(false)

c 指向channel的指针

## channel 底层实现逻辑
### 通道创建
```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem
    // buf数组所需分配内存大小
    mem := elem.size*uintptr(size)
    var c *hchan
    switch {
    case mem == 0:// Unbuffered channels，buf无需内存分配
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        // Race detector uses this location for synchronization.
        c.buf = c.raceaddr()
    case elem.ptrdata == 0: // Buffered channels，通道元素类型非指针
        c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
        c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
        // Buffered channels，通道元素类型是指针
        c = new(hchan)
        c.buf = mallocgc(mem, elem, true)
    }

    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)
    return c
}
```
通道创建主要是分配内存并构建hchan对象。

### 通道写入
3种异常情况处理
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 1.channel为nil
    if c == nil {
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    
    lock(&c.lock) //加锁
    
    // 2.如果channel已关闭，直接panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
   
    // Block on the channel. 
    mysg := acquireSudog()
    c.sendq.enqueue(mysg) // 入sendq等待队列
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
    
    
    closed := !mysg.success // 协程被唤醒的原因是因为数据传递还是通道被关闭
    // 3.因channel被关闭导致阻塞写协程被唤醒并panic
    if closed {
        panic(plainError("send on closed channel"))
    }
}
```

对 nil channel写入，会死锁

对被关闭的channel写入，会panic

对因写入而陷入阻塞的协程，如果channel被关闭，阻塞协程会被唤醒并panic

#### 写时有阻塞读协程
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    lock(&c.lock) //加锁
    // 1、当存在等待接收的Goroutine
    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        send(c, sg, ep, func() { unlock(&c.lock) }, 3) // 直接把正在发送的值发送给等待接收的Goroutine，并将此接收协程放入可调度队列等待调度
        return true
    }
}

// send processes a send operation on an empty channel c.
// The value ep sent by the sender is copied to the receiver sg.
// The receiver is then woken up to go on its merry way.
// Channel c must be empty and locked.  send unlocks c with unlockf.
// sg must already be dequeued from c.
// ep must be non-nil and point to the heap or the caller's stack.
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // 将ep写入sg中的elem
    if sg.elem != nil {
         t:=c.elemtype
         dst := sg.elem
        
         // memmove copies n bytes from "from" to "to".
         memmove(dst, ep, t.size)
         sg.elem = nil // 数据已经被写入到<- c变量，因此sg.elem指针可以置空了
    }
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    sg.success = true
    
    // 唤醒receiver协程gp
    goready(gp, skip+1)
}

// 唤醒receiver协程gp，将其放入可运行队列中等待调度执行
func goready(gp *g, traceskip int) {
    systemstack(func() {
        ready(gp, traceskip, true)
    })
}
// Mark gp ready to run.
func ready(gp *g, traceskip int, next bool) {
    status := readgstatus(gp)
    // Mark runnable.
    _g_ := getg()
    mp := acquirem() // disable preemption because it can be holding p in a local var
    // status is Gwaiting or Gscanwaiting, make Grunnable and put on runq
    casgstatus(gp, _Gwaiting, _Grunnable)
    runqput(_g_.m.p.ptr(), gp, next)
    wakep()
    releasem(mp)
}
```

加锁

从阻塞读协程队列取出sudog节点

在send方法中，调用memmove方法将数据拷贝给sudog.elem指向的变量。

goready方法唤醒接收到数据的阻塞读协程g，将其放入协程可运行队列中等待调度

解锁

#### 写时无阻塞读协程但环形缓冲区仍有空间
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    lock(&c.lock) //加锁
    // 当缓冲区未满时
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx) // 获取指向缓冲区数组中位于sendx位置的元素的指针
        typedmemmove(c.elemtype, qp, ep) // 将当前发送的值拷贝到缓冲区
        c.sendx++ 
        if c.sendx == c.dataqsiz {
            c.sendx = 0 // 因为是循环队列，sendx等于队列长度时置为0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }
}
```

加锁

将数据放入环形缓冲区

解锁

#### 写时无阻塞读协程且环形缓冲区无空间
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    lock(&c.lock) //加锁
    
    // Block on the channel. 
    // 将当前的Goroutine打包成一个sudog节点，并加入到阻塞写队列sendq里
    gp := getg()
    mysg := acquireSudog()
    mysg.elem = ep
    mysg.g = gp
    mysg.c = c
    gp.waiting = mysg
    c.sendq.enqueue(mysg) // 入sendq等待队列
    
   
    // 调用gopark将当前Goroutine设置为等待状态并解锁，进入休眠等待被唤醒，触发协程调度
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
    
    // 被唤醒之后执行清理工作并释放sudog结构体
    gp.waiting = nil
    gp.activeStackChans = false
    closed := !mysg.success // gp被唤醒的原因是因为数据传递还是通道被关闭
    gp.param = nil
  
    mysg.c = nil
    releaseSudog(mysg)
    // 因关闭被唤醒则panic
    if closed {
        panic(plainError("send on closed channel"))
    }
    // 数据成功传递
    return true
}
```

加锁。

将当前协程gp封装成sudog节点，并加入channel的阻塞写队列sendq。

调用gopark将当前协程设置为等待状态并解锁，触发调度其它协程运行。

因数据被读或者channel被关闭，协程从park中被唤醒，清理sudog结构。

因channel被关闭导致协程唤醒，panic

返回

#### 整体写流程
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 1.channel为nil
    if c == nil {
        // 当前Goroutine阻塞挂起
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    // 2.加锁
    lock(&c.lock) 
    
    // 3.如果channel已关闭，直接panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
    // 4、存在阻塞读协程
    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        send(c, sg, ep, func() { unlock(&c.lock) }, 3) // 直接把正在发送的值发送给等待接收的Goroutine，并将此接收协程放入可调度队列等待调度
        return true
    }
    // 5、缓冲区未满时
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx) // 获取指向缓冲区数组中位于sendx位置的元素的指针
        typedmemmove(c.elemtype, qp, ep) // 将当前发送的值拷贝到缓冲区
        c.sendx++ 
        if c.sendx == c.dataqsiz {
            c.sendx = 0 // 因为是循环队列，sendx等于队列长度时置为0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }
    // Block on the channel. 
    // 6、将当前协程打包成一个sudog结构体，并加入到channel的阻塞写队列sendq
    gp := getg()
    mysg := acquireSudog()
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.sendq.enqueue(mysg) // 入sendq等待队列
    
    atomic.Store8(&gp.parkingOnChan, 1)
    
    // 7.调用gopark将当前协程设置为等待状态并解锁，进入休眠，等待被唤醒，并触发协程调度
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
    
    // 8. 被唤醒之后执行清理工作并释放sudog结构体
    gp.waiting = nil
    gp.activeStackChans = false
    closed := !mysg.success // g被唤醒的原因是因为数据传递还是通道被关闭
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    // 9.因关闭被唤醒则panic
    if closed {
        panic(plainError("send on closed channel"))
    }
    // 10.数据成功传递
    return true
}
```

channel为nil检查。为空则死锁。

加锁

如果channel已关闭，直接panic。

当存在阻塞读协程，直接把数据发送给读协程，唤醒并将其放入协程可运行队列中等待调度运行。

当缓冲区未满时，将当前发送的数据拷贝到缓冲区。

当既没有阻塞读协程，缓冲区也没有剩余空间时，将协程加入阻塞写队列sendq。

调用gopark将当前协程设置为等待状态，进入休眠等待被唤醒，触发协程调度。

被唤醒之后执行清理工作并释放sudog结构体

唤醒之后检查，因channel被关闭导致协程唤醒则panic。

返回。

### 通道读

#### 2种异常情况处理
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // 1.channel为nil
    if c == nil {
        // 否则，当前Goroutine阻塞挂起
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }

    lock(&c.lock)
    // 2.如果channel已关闭，并且缓冲区无元素，返回(true,false)
    if c.closed != 0 {
        if c.qcount == 0 {
            unlock(&c.lock)
            if ep != nil {
                //根据channel元素的类型清理ep对应地址的内存，即ep接收了channel元素类型的零值
                typedmemclr(c.elemtype, ep)
            }
            return true, false
        }
    }
}
```

channel未初始化，读操作会死锁

channel已关闭且缓冲区无数据，给读变量赋零值。

#### 读时有阻塞写协程
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    lock(&c.lock)
    
    // Just found waiting sender with not closed.
    // 等待发送的队列sendq里存在Goroutine
    if sg := c.sendq.dequeue(); sg != nil {
        // Found a waiting sender. If buffer is size 0, receive value
        // directly from sender. Otherwise, receive from head of queue
        // and add sender's value to the tail of the queue (both map to
        // the same buffer slot because the queue is full).
        // 如果无缓冲区，那么直接从sender接收数据；否则，从buf队列的头部接收数据，并把sender的数据加到buf队列的尾部
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true // 接收成功
    }
    
    
}

// recv processes a receive operation on a full channel c.
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // channel无缓冲区，直接从sender读
    if c.dataqsiz == 0 {
        if ep != nil {
            // copy data from sender
            t := c.elemtype
            src := sg.elem
            typeBitsBulkBarrier(t, uintptr(ep), uintptr(src), t.size)
            memmove(dst, src, t.size)
        }
    } else {
        // 从队列读,sender再写入队列
        qp := chanbuf(c, c.recvx)
        // copy data from queue to receiver
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        // copy data from sender to queue
        typedmemmove(c.elemtype, qp, sg.elem)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
    }
    // 唤醒sender队列协程sg
    sg.elem = nil
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    sg.success = true
    // 唤醒协程
    goready(gp, skip+1)
}
```

加锁

从阻塞写队列取出sudog节点

假如channel为无缓冲区通道，则直接读取sudog对应写协程数据，唤醒写协程。

假如channel为缓冲区通道，从channel缓冲区头部(recvx)读数据，将sudog对应写协程数据，写入缓冲区尾部(sendx)，唤醒写协程。

解锁

#### 读时无阻塞写协程且缓冲区有数据
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    lock(&c.lock)
    // 缓冲区buf中有元素,直接从buf拷贝元素到当前协程(在已关闭的情况下，队列有数据依然会读)
    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)// 将从buf中取出的元素拷贝到当前协程
        }
        typedmemclr(c.elemtype, qp) // 同时将取出的数据所在的内存清空
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true // 接收成功
    }
}
```

加锁

从环形缓冲区读数据。在channel已关闭的情况下，缓冲区有数据依然可以被读。

解锁

#### 读时无阻塞写协程且缓冲区无数据
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    lock(&c.lock)

    // no sender available: block on this channel.
    // 阻塞模式，获取当前Goroutine，打包一个sudog，并加入到channel的接收队列recvq里
    gp := getg()
    mysg := acquireSudog()
    mysg.elem = ep
    gp.waiting = mysg
    mysg.g = gp
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg) // 入接收队列recvq
    
    // 挂起当前Goroutine，设置为_Gwaiting状态，进入休眠等待被唤醒
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

    // 因通道关闭或者读到数据被唤醒
    gp.waiting = nil
    success := mysg.success
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, success // 10.返回成功
}
```

加锁。

将当前协程gp封装成sudog节点，加入channel的阻塞读队列recvq。

调用gopark将当前协程设置为等待状态并解锁，触发调度其它协程运行。

因读到数据或者channel被关闭，协程从park中被唤醒，清理sudog结构。

返回

#### 整体读流程
```go
// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // 1.channel为nil
    if c == nil {
        // 否则，当前Goroutine阻塞挂起
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    // 2.加锁
    lock(&c.lock)
    // 3.如果channel已关闭，并且缓冲区无元素，返回(true,false)
    if c.closed != 0 {
        if c.qcount == 0 {
            unlock(&c.lock)
            if ep != nil {
                //根据channel元素的类型清理ep对应地址的内存，即ep接收了channel元素类型的零值
                typedmemclr(c.elemtype, ep)
            }
            return true, false
        }
        // The channel has been closed, but the channel's buffer have data.
    } else {
        // Just found waiting sender with not closed.
        // 4.存在阻塞写协程
        if sg := c.sendq.dequeue(); sg != nil {
            // Found a waiting sender. If buffer is size 0, receive value
            // directly from sender. Otherwise, receive from head of queue
            // and add sender's value to the tail of the queue (both map to
            // the same buffer slot because the queue is full).
            // 如果无缓冲区，那么直接从sender接收数据；否则，从buf队列的头部接收数据，并把sender的数据加到buf队列的尾部
            recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
            return true, true // 接收成功
        }
    }
    // 5.缓冲区buf中有元素,直接从buf拷贝元素到当前协程(在已关闭的情况下，队列有数据依然会读)
    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)// 将从buf中取出的元素拷贝到当前协程
        }
        typedmemclr(c.elemtype, qp) // 同时将取出的数据所在的内存清空
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true // 接收成功
    }

    // no sender available: block on this channel.
    // 6.获取当前Goroutine，封装成sudog节点，加入channel阻塞读队列recvq
    gp := getg()
    mysg := acquireSudog()
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg) // 入接收队列recvq
    
    atomic.Store8(&gp.parkingOnChan, 1)
    // 7.挂起当前Goroutine，设置为_Gwaiting状态，进入休眠等待被唤醒
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

    // 8.因通道关闭或者可读被唤醒
    gp.waiting = nil
    gp.activeStackChans = false
    success := mysg.success
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    // 9.返回
    return true, success 
}
```

通道读流程如下:

channel为nil检查。空则死锁。

加锁。

如果channel已关闭，并且缓冲区无数据，读变量赋零值，返回。

当存在阻塞写协程，如果缓冲区已满，则直接从sender接收数据；否则，从环形缓冲区头部接收数据，并把sender的数据加到环形缓冲区尾部。唤醒sender，将其放入协程可运行队列中等待调度运行，返回。

如果缓冲区中有数据，直接从缓冲区拷贝数据到当前协程，返回。

当既没有阻塞写协程，缓冲区也没有数据时，将协程加入阻塞读队列recvq。

调用gopark将当前协程设置为等待状态，进入休眠等待被唤醒，触发协程调度。

因通道关闭或者可读被唤醒。

返回。

### 通道关闭
```go
func closechan(c *hchan) {
    // // 1.channel为nil则panic
    if c == nil {
        panic(plainError("close of nil channel"))
    }
    lock(&c.lock)
    // 2.已关闭的channel再次关闭则panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }
    // 设置关闭标记
    c.closed = 1

    var glist gList
    // 遍历recvq和sendq中的协程放入glist
    // release all readers
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = unsafe.Pointer(sg)
        sg.success = false
        glist.push(gp)
    }

    // release all writers (they will panic)
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = unsafe.Pointer(sg)
        sg.success = false
        glist.push(gp)
    }
    unlock(&c.lock)

    // 3.将glist中所有Goroutine的状态置为_Grunnable，等待调度器进行调度
    for !glist.empty() {
        gp := glist.pop()
        gp.schedlink = 0
        goready(gp, 3)
    }
}
```

channel为nil检查。为空则panic

已关闭channel再次被关闭，panic

将sendq和recvq所有Goroutine的状态置为_Grunnable，放入协程调度队列等待调度器调度

## 高频面试题
channel 的底层实现原理 （数据结构）

nil、关闭的 channel、有数据的 channel，再进行读、写、关闭会怎么样？（各类变种题型）

有缓冲channel和无缓冲channel的区别