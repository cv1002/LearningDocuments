# 面试题
- [面试题](#面试题)
- [讲一讲 GMP 模型](#讲一讲-gmp-模型)
- [了解的GC算法有哪些？](#了解的gc算法有哪些)
  - [引用计数](#引用计数)
  - [标记-清除](#标记-清除)
  - [分代收集](#分代收集)
  - [三色标记法](#三色标记法)
- [go垃圾回收，什么时候触发](#go垃圾回收什么时候触发)
  - [主动触发](#主动触发)
  - [被动触发](#被动触发)
- [为什么不要大量使用goroutine](#为什么不要大量使用goroutine)
- [channel有缓冲和无缓冲在使用上有什么区别？](#channel有缓冲和无缓冲在使用上有什么区别)
- [对未初始化的的chan进行读写，会怎么样？为什么？](#对未初始化的的chan进行读写会怎么样为什么)
  - [写未初始化的 chan](#写未初始化的-chan)
  - [读未初始化的 chan](#读未初始化的-chan)
  - [为什么对未初始化的chan就会阻塞呢？](#为什么对未初始化的chan就会阻塞呢)
    - [对于写的情况](#对于写的情况)
    - [对于读的情况](#对于读的情况)

# 讲一讲 GMP 模型
三个字母的含义

- G（Goroutine）：G 就是我们所说的 Go 语言中的协程 Goroutine 的缩写，相当于操作系统中的进程控制块。其中存着 goroutine 的运行时栈信息，CPU 的一些寄存器的值以及执行的函数指令等。
- M（Machine）：代表一个操作系统的主线程，对内核级线程的封装，数量对应真实的 CPU 数。一个 M 直接关联一个 os 内核线程，用于执行 G。M 会优先从关联的 P 的本地队列中直接获取待执行的 G。M 保存了 M 自身使用的栈信息、当前正在 M上执行的 G 信息、与之绑定的 P 信息。
- P（Processor）：Processor 代表了 M 所需的上下文环境，代表 M 运行 G 所需要的资源。是处理用户级代码逻辑的处理器，可以将其看作一个局部调度器使 go 代码在一个线程上跑。当 P 有任务时，就需要创建或者唤醒一个系统线程来执行它队列里的任务，所以 P 和 M 是相互绑定的。总的来说，P 可以根据实际情况开启协程去工作，它包含了运行 goroutine 的资源，如果线程想运行 goroutine，必须先获取 P，P 中还包含了可运行的 G 队列。

# 了解的GC算法有哪些？
常见的垃圾回收算法有以下几种：

## 引用计数
- 引用计数：对每个对象维护一个引用计数，当引用该对象的对象被销毁时，引用计数减1，当引用计数器为0时回收该对象。
- 优点：对象可以很快的被回收，不会出现内存耗尽或达到某个阀值时才回收。
- 缺点：不能很好的处理循环引用，而且实时维护引用计数，有也一定的代价。
- 代表语言：Python、PHP

## 标记-清除
- 标记-清除：从根变量开始遍历所有引用的对象，引用的对象标记为"被引用"，没有被标记的进行回收。
- 优点：解决了引用计数的缺点。
- 缺点：需要STW，即要暂时停掉程序运行。
- 代表语言：Golang(其采用三色标记法)

## 分代收集
- 分代收集：按照对象生命周期长短划分不同的代空间，生命周期长的放入老年代，而短的放入新生代，不同代有不能的回收算法和回收频率。
- 优点：回收性能好
- 缺点：算法复杂
- 代表语言： JAVA

## 三色标记法
- 初始状态下所有对象都是白色的。
- 从根节点开始遍历所有对象，把遍历到的对象变成灰色对象
- 遍历灰色对象，将灰色对象引用的对象也变成灰色，然后将遍历过的灰色对象变成黑色对象。
- 循环步骤3，直到灰色对象全部变黑色。
- 回收所有白色对象（垃圾）。

# go垃圾回收，什么时候触发
## 主动触发
主动触发(手动触发)，通过调用 runtime.GC 来触发GC，此调用阻塞式地等待当前GC运行完毕。

## 被动触发
被动触发，分为两种方式：

- 使用步调（Pacing）算法，其核心思想是控制内存增长的比例,每次内存分配时检查当前内存分配量是否已达到阈值（环境变量GOGC）：默认100%，即当内存扩大一倍时启用GC。
- 使用系统监控，当超过两分钟没有产生任何GC时，强制触发 GC。

# 为什么不要大量使用goroutine
合理复用协程

使用goroutine可以帮助提高程序的并发性和性能，但是过度使用goroutine会带来一些问题，例如：

- 内存占用增加，因为每个goroutine都需要占用一定的内存
- 过多的goroutine会导致CPU上下文切换频繁，影响程序性能
- 如果goroutine没有正确的管理，可能会导致资源泄漏或死锁
- 为了优化这些问题，可以考虑以下方法：
- 确定适当的goroutine数量，避免过度使用goroutine。
- 使用有限的goroutine池，以限制goroutine的总数，并避免内存占用过多。
- 优化goroutine的调度，以减少CPU上下文切换的次数。
- 使用通道和其他同步原语来避免竞争条件和死锁。

# channel有缓冲和无缓冲在使用上有什么区别？
- 无缓冲：发送和接收需要同步。
- 有缓冲：不要求发送和接收同步，缓冲满时发送阻塞。
- 因此 channel 无缓冲时，发送阻塞直到数据被接收，接收阻塞直到读到数据；channel有缓冲时，当缓冲满时发送阻塞，当缓冲空时接收阻塞。

# 对未初始化的的chan进行读写，会怎么样？为什么？

读写未初始化的chan都会阻塞。

## 写未初始化的 chan

```go
package main
// 写未初始化的chan
func main() {
	var c chan int
	c <- 1
}
```

```
// 输出结果
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send (nil chan)]:
main.main()
        /Users/admin18/go/src/repos/main.go:6 +0x36
```

## 读未初始化的 chan
```go
package main
import "fmt"
// 读未初始化的chan
func main() {
	var c chan int
	num, ok := <-c
	fmt.Printf("读chan的协程结束, num=%v, ok=%v\n", num, ok)
}
```

```
// 输出结果
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive (nil chan)]:
main.main()
        /Users/admin18/go/src/repos/main.go:6 +0x46
```

## 为什么对未初始化的chan就会阻塞呢？
### 对于写的情况
```go
//在 src/runtime/chan.go中
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
      // 不能阻塞，直接返回 false，表示未发送成功
      if !block {
        return false
      }
      gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
      throw("unreachable")
	}
  // 省略其他逻辑
}
```

- 未初始化的chan此时是等于nil，当它不能阻塞的情况下，直接返回 false，表示写 chan 失败
- 当chan能阻塞的情况下，则直接阻塞 gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2) , 然后调用throw(s string)抛出错误,其中waitReasonChanSendNilChan就是刚刚提到的报错"chan send (nil chan)"
### 对于读的情况
```go
//在 src/runtime/chan.go中
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    //省略逻辑...
    if c == nil {
        if !block {
          return
        }
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    //省略逻辑...
}
```

- 未初始化的chan此时是等于nil，当它不能阻塞的情况下，直接返回 false，表示读 chan 失败
- 当chan能阻塞的情况下，则直接阻塞 gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2) , 然后调用throw(s string)抛出错误,其中waitReasonChanReceiveNilChan就是刚刚提到的报错"chan receive (nil chan)"

