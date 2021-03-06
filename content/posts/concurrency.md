---
title: "Concurrency Patterns In Go"
date: 2021-01-10T23:27:01+08:00
tags: ["concurrency", "Go"]
categories: ["Go语言学习"]
draft: false
---

## 什么是并发编程

**并发** `concurrency` 是一种**设计**：

* 将程序设计成一个包含许多独立过程的集合；
* 允许这些过程最终可以**并行** `parallel` 地去执行；
>尽管 ***并发*** 并不要求必须同时运行，例如在单核 CPU 物理机上无法实现并行却可以依靠操作系统时间片轮转等调度来实现并发；
并发是一种操作系统在时间维度的虚拟，即时分复用。
* 代码运行的结果总是相同，无论是以并行还是顺序的方式。

## 具体要求

* 通过划分多个独立的任务来将代码以及数据分组
* 没有竞争条件[^1]
* 没有死锁[^2]
* 比常规的程序更有效，即执行更快

## Go如何并发

>Don't communicate by sharing memory; share memory by communicating. (R. Pike)

不要通过共享内存来通信，这意味着不同的并发实体不应该通过遵守严格、容易出错的内存可见性和同步策略 (比如内存屏障[^3])来进行通信 (这是可以做到的，但是会让情况变得复杂，而且由于数据竞争会造成不可预期的结果)。

应该通过通信来共享内存 (数据) : Go 使用了基于 CSP[^4] (Communicating Sequential Process，通信顺序程序) 的并发模型，goroutine 做为并发通信的实体，通过 channel 来实现数据共享。
> A send on a channel happens before the corresponding receive from that channel completes. (Golang Spec)

`happens before` 保证了一个 goroutine 对 channel 的写入对另一个 goroutine 的读取操作是可见的。这是一种简单的顺序传递模式，由于 goroutine 之间通过 channel 来共享对内存的引用 (通过拷贝)，所以没有修改共享内存，也就不需要 mutex 之类的同步操作。理想中的 Go 并发，应该没有共享空间，每个并发主体 goroutine 仅能看到自己拥有的内存部分。

### channel

* 无缓冲通道

向无缓冲通道发送和从无缓冲通道中读取都会被阻塞，知道发送方和接收方同时准备好，才得意继续运行，就像同一赛道上两个交接接力棒的运动员。

```go
	ch := make(chan int)
	ch <- 10

$ fatal error: all goroutines are asleep - deadlock!

	ch := make(chan int)
	<-ch

$ fatal error: all goroutines are asleep - deadlock!

	ch := make(chan string)
	go func() {
		msg := <-ch
		fmt.Println(msg)
	}()
	ch <- "hello world"

$ hello world

```

* 缓冲通道

当缓冲区已满时向通道发送数据 或者 从通道读取数据时缓冲区为空 (没有数据就意味着阻塞，同样体现为，不使用 `make` 初始化一个通道，从此通道读取数据将导致程序永久阻塞 )，程序会阻塞，其余情况下可以继续运行。

```go
	ch := make(chan int, 2)
	ch <- 1
	fmt.Println("send 1 unblocking...")
	ch <- 2
	fmt.Println("send 2 unblocking...")
	ch <- 3
	fmt.Println("send 3 unblocking...")
	
$ send 1 unblocking...
$ send 2 unblocking...
$ fatal error: all goroutines are asleep - deadlock!
```

* 已关闭通道

使用 `close(ch chan)` 可以将通道 ch 关闭。

从已关闭通道读取，总能得到返回。第一个返回值是通道类型对应的零值，第二个返回值是一个 `boolean` 类型的 false，代表数据不可用，通道未关闭时其为 true 。

```go
	ch := make(chan int)
	close(ch)
	for i := 0; i < 3; i++ {
		val, ok := <-ch
		fmt.Println(val, ok)
	}

$ 0 false
$ 0 false
$ 0 false

```

向已关闭的通道发送数据，会引起运行时恐慌 `panic`：

```go
	ch := make(chan int)
	close(ch)
	ch <- 1024

$ panic: send on closed channel

```

### select

select 的语法结构和 switch 很像，不过它是专门用来进行通道的接收、发送操作的。

```go
	ch := make(chan int)
	select {
	case <-ch:
	}

// 此代码会永久阻塞
$ fatal error: all goroutines are asleep - deadlock!
```

select 会在某个 case 的通道收发操作准备好时进入分支执行，当没有任何一个分支准备好时，程序会阻塞并等待。当有多个分支同时准备好时，会随机选择一个分支执行，注意，执行顺序并非按照书写 case 的顺序，此机制可以防止饥饿。

```go
	ch := make(chan int, 5)
	done := make(chan struct{})
	defer close(ch)
	go func() {
		for {
			select {
			case <-ch:
				fmt.Println("choose case a")
			case <-ch:
				fmt.Println("choose case b")
			case <-done:
				return
			}
		}
	}()
	for i := 0; i < 5; i++ {
		ch <- i
	}
	time.Sleep(time.Second)
	close(done)

$ choose case b
$ choose case a
$ choose case b
$ choose case b
$ choose case a

```

select 可以实现非阻塞通道收发操作，只需要添加一个 `default` 分支。

```go
	ch := make(chan int)
	select {
	case <-ch:
	default:
		fmt.Println("enter default case")
	}

$ enter default case
```

### 使用 channel 应该注意什么

1. 不要造成死锁。 channel 涉及很多阻塞，请确保各种通信方之间的顺序在掌控之中，否则可能造成死锁而无法推进。
1. 数据在 channel 中传递同样是值拷贝，因此当传递一些大对象时，请考虑性能影响。
1. 通过 channel 传递指针会打破不同并发主体之间的内存隔离，造成数据竞争，不要这样做。

### 原子操作

程序演进的三种类型:

1. Blocking：程序可能进入时间未知的阻塞状态而无法推进 (make progress)，比如使用 mutex/lock。
2. Lock free：程序中至少有一部分 (一个或多个并发主体)总是在向前推进，通过不使用 mutex 而是 CAS (Compare And Swap) 来实现。
3. Wait free：程序中的每一个部分 (每一个并发主体)都能在有限的步骤或者时间内向前推进。有兴趣可以转向 [Lock-Free Single-Producer - Single Consumer Circular Queue](https://www.codeproject.com/articles/43510/lock-free-single-producer-single-consumer-circular)。

接入来介绍几种借助 Go 的 atomic package 来实现的 lock free 并发模式。

#### Spinning CAS 自旋锁

使用 CAS 实现最简单的自旋锁只需要两个东西：一个表示状态的变量以及一个表示 free 状态的常量。

```go
const free = int32(0) // 定义无争用状态

type Spinlock struct {
	state int32
}

func (sl *Spinlock) Lock() {
  for !atomic.CompareAndSwapInt32(&sl.state, free, 1) { // 1可以是任何不等于 free(0) 的值
		runtime.Gosched() // 主动让出cpu，此操作不会导致当前协程挂起，其会自动恢复运行
	}
}

func (sl *Spinlock) Unlock() {
	atomic.StoreInt32(&sl.state, free)
}
```

以上就是一个简单的自旋锁的实现，来看一下实现的效果。

下面的这段代码在没有使用任何锁的情况下，可能会导致最终的结果不为0，用例少时可能观察不出来，我们让它运行在一个较大的循环。

```go
func main() {
	passed := 0
	for i := 0; i < 1000000; i++ {
		man := &Man{}
		wg := sync.WaitGroup{}
		wg.Add(2)
		go func() {
			defer wg.Done()
			man.DecrMoney() // 对金额减1
		}()
		go func() {
			defer wg.Done()
			man.IncrMoney() // 对金额加1
		}()
		wg.Wait()
    // 此时预期的 man.money 值应该为0
		if man.money != 0 {
			fmt.Printf("passedStep: %d\nunexpeted man's money：%d", passed, man.money)
			return
		}
		passed++
	}
	fmt.Printf("all cases pass：%d", passed)
}

type Man struct {
	Spinlock
	money int
}

func (m *Man) DecrMoney() {
	// m.Lock()
	m.money -= 1
	// m.Unlock()
}

func (m *Man) IncrMoney() {
	// m.Lock()
	m.money += 1
	// m.Unlock()
}
```

使用 -race 选项启用 Go 的 data race 检测，此时的输出很有可能是：

```shell
$ go run -race main.go

# 输出
$ WARNING: DATA RACE
$ Read at 0x00c00001c0a8 by goroutine 8:
$  main.(*Man).IncrMoney()
$      /Users/malishen/go/src/awesomeProject/main.go:48 +0x7b
$  main.main.func2()
$      /Users/malishen/go/src/awesomeProject/main.go:23 +0x69
...

$ passedStep: 23517
$ unexpeted man's money：1
$ Process finished with exit code 1

```

当启动 Spinlock 后：

```go
func (m *Man) DecrMoney() {
	m.Lock()
	m.money -= 1
	m.Unlock()
}

func (m *Man) IncrMoney() {
	m.Lock()
	m.money += 1
	m.Unlock()
}
```

此时的结果无论循环变量 i 设置多大，结果都是：

```shell
$ go run -race main.go

# 输出,没有 data race
$ all cases pass：1000000
```

#### Ticket Storage

再来使用 CAS 实现一个 lock free 的本地存储，并且具有先到先得的顺序公平性，需要一个线性的结构以及一个 ticket 变量和 done 变量。
```go
type TicketStorage struct {
	ticket uint64 // 任务号，只增不减
	done   uint64 // 已经完成的任务
	slots  []string // 为了简单，这里想象成是无限容量的
}

func (ts *TicketStorage) Put(msg string) {
	t := atomic.AddUint64(&ts.ticket, 1) - 1 // 减1是获取到的ticket对应的slots下标
	ts.slots[t] = msg
	for !atomic.CompareAndSwapUint64(&ts.done, t, t+1) {
		runtime.Gosched()
	}
}

func (ts *TicketStorage) GetDone() []string {
	return ts.slots[:atomic.LoadUint64(&ts.done)+1]
}
```

> Note：上面这个 demo 仅仅是一个并发设计模式的演示，由于 slots 其实并不是一个无限容量的切片，所以此代码是无法直接运行的。

总结一下，这个 Ticket Storage 在 `Put()` 时，ticket 原子地单调递增，所以其是全局唯一的，从而保证在对 slots 相应索引赋值时没有 data race。类似于很多人去银行排队办理业务，银行只有一个窗口，所以每个人可以先取一个号，等叫到号了才可以去办理业务。这个模式的最大特点是对于 `GetDone()`的读取操作，是 wait free 的，可以仔细体会一下。

### 总结

CSP 是 Go 并发理论的基石，Go 向开发者提供了强大的轻量级线程 goroutine 和 通道 channel 让并发编程变得简单、高效，但是这种简单性意味着难以深入掌握，在这里给出几个建议：

* 避免阻塞，避免数据竞争 data race；
* 使用 channel 进行通信而不是直接共享内存，使用 select 去管理 channel，channel 应该由创建者/发送方决定何时关闭；
* 当某些场景 channel 无法很好地帮助你时：

	* 优先使用 sync 并发工具包；
	* 在简单情况下或者确实需要时，尝试使用 atomic 写出 lock free 的代码。

[^1]:数据竞争 <https://zh.wikipedia.org/wiki/%E7%AB%B6%E7%88%AD%E5%8D%B1%E5%AE%B3>

[^2]: 死锁 <https://zhuanlan.zhihu.com/p/26945588>

[^3]: 内存屏障 <https://zh.wikipedia.org/wiki/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C>

[^4]: CSP<https://zh.wikipedia.org/wiki/%E4%BA%A4%E8%AB%87%E5%BE%AA%E5%BA%8F%E7%A8%8B%E5%BC%8F>

# 参考资料

1. The Go Memory Model <https://golang.org/ref/mem>
1. 《 Go 语言设计与实现》<https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/>
1. 深入golang之---goroutine并发控制与通信, 知乎 <https://zhuanlan.zhihu.com/p/36907022>
1. Share Memory By Communicating, The Go Blog <https://blog.golang.org/codelab-share>
1. 为什么使用通信来共享内存 <https://draveness.me/whys-the-design-communication-shared-memory/>

