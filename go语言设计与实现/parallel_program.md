# 第六章 并发编程

## 6.1 上下文 Context
Context 是一个非常重要的包，主要目的是在多个 goroutine 之间传递上下文信息，并提供一种机制来通知这些 goroutine 停止工作（例如超时或取消信号）。  

**核心功能：**  
1. 取消信号传递 ：当父协程需要取消子协程时，通过 context 发送信号，避免资源泄漏。
2. 超时控制 ：为任务设置最大执行时间，超时自动取消。
3. 传递请求元数据 ：例如 HTTP 请求的认证信息、请求 ID 等。  

context.Context 是一个接口，定义了以下方法：
````golang
type Context interface {
    Deadline() (deadline time.Time, ok bool) // 返回截止时间
    Done() <-chan struct{}                   // 返回一个 channel，用于监听取消信号
    Err() error                              // 返回 context 被取消的原因
    Value(key interface{}) interface{}       // 获取与 key 关联的值
}
````

### 6.1.1 设计原理
Go 服务每一个请求都是通过一个 goroutine 来进行处理的。我们可能需要多个 goroutine 来处理一个请求。而 context.Context 的作用就是在不同的 goroutine 之间同步请求特定数据，取消信号以及处理请求的截止日期。  

每一个 goroutine 都会从最顶层的 goroutine 一层一层传到最底层。context.Context 可以在上层 goroutine 出现错误时，将错误信号同步给下层，避免不必要的资源浪费。  

下面代码创建了一个过期时间为1秒的上下文，但是下层 goroutine 的工作时长为1.5秒，这样就能模拟当上层出现错误之后，信号是如何同步的。
````go
func handle(ctx context.Context, duration time.Duration) {
	select {
	case <-ctx.Done():
		fmt.Println("handle", ctx.Err())
	case <-time.After(duration):
		fmt.Println("process request with", duration)
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()

	go handle(ctx, 1500*time.Millisecond)

	select {
	case <-ctx.Done():
		fmt.Println("main", ctx.Err())
	}

	time.Sleep(5 * time.Second)
}
````
handle 没有继续自己的任务，而是在收到 ctx.Done() 的消息之后，就结束了任务的运行。
```
handle context deadline exceeded
main context deadline exceeded
```
由此可见多个 goroutine 同时订阅 ctx.Done() 管道中的消息，一旦接收到取消信号就立刻停止当前正在执行的工作。

### 6.1.2 常用 Context 类型
通过 context 包提供的函数生成不同类型的 Context：
+ context.Background()：  
  + 根节点 Context，通常作为主函数或请求入口的初始 Context 
+ context.TODO()：  
  + 临时占位符，用于未确定 Context 传递逻辑的场景  

在源代码上来看， context.Background() 和 context.TODO() 只是互为别名，只是在使用和语义上有区别。 

+ WithCancel(parent Context)：  
  + 生成可手动取消的 Context，返回 cancel 函数
  + 是从现有的 goroutine 中衍生一个新的上下文，并返回用于取消该上下文的函数。
+ WithTimeout(parent Context, timeout time.Duration)：  
  + 生成带超时时间的 Context，超时自动取消 
+ WithValue(parent Context, key, val interface{})：  
  + 生成携带键值对的 Context，用于传递请求范围数据 
````golang
func handleRequest(ctx context.Context) {
	userID := ctx.Value(string("user_id")).(string)
	fmt.Println("Proccessing data||use_id=", userID)
}

func main() {
	ctx := context.WithValue(context.Background(), string("user_id"), "12345")

	handleRequest(ctx)
}
````

## 6.1.3 小结
Go 语言中的 context.Context 的主要作用还是在多个 Goroutine 组成的树中同步取消信号以减少对资源的消耗和占用，虽然它也有传值的功能，但是这个功能我们还是很少用到。


## 6.2 同步原语和锁
Golang 作为一个原生支持用户态进程的语言，当提到并发，肯定少不了锁。锁是并发编程中的同步原语。

### 6.2.1 基本原语
Golang 在 sync 包中提供了用于同步的基本原语，如sync.Mutex、sync.RWMutex、sync.WaitGroup、sync.Once 和 sync.Cond。但这些都是一些相对比较原始的同步机制。大多数情况下，我们应该使用抽象等级更高的 channel 实现同步。

### 6.2.2 Mutex
Golang 中的 Mutex 是由 state 和 sema 组成的。其中 state 是当前互斥锁的状态，而 sema 是用于控制锁状态的信号量。
````golang
type Mutex struct {
	state int32
	sema  uint32
}
````
在状态中，其中最第三位分别是 mutexLocked、mutexWoken 和 mutexStarving，剩下的位置用来表示当前有多少个 Goroutine 在等待互斥锁的释放。
+ mutexLocked：表示互斥锁的锁定状态
+ mutexWoken：正常模式
  + 锁的等待者会按照先进先出的顺序获取锁。但是刚被唤起的 Goroutine 与新创建的 Goroutine 竞争时，大概率会获取不到锁。
+ mutexStarving：表示现在互斥锁处在饥饿模式  
  + 一旦 Goroutine 超过 1ms 没有获取到锁，互斥锁会直接交给等待队列最前面的 Goroutine。

**方法：：**
+ Lock()：获取锁，若已被占用则阻塞。
+ Unlock()：释放锁。
````golang
var mu sync.Mutex
var count int

func increment(wg *sync.WaitGroup) {
	defer wg.Done()
	mu.Lock()
	count++
	mu.Unlock()
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 50; i++ {
		wg.Add(2) // 每次循环启动 2 个协程
		go increment(&wg)
		go increment(&wg)
	}

	wg.Wait() // 等待所有协程完成
	fmt.Println(count)
}
````

### 6.2.3 RWMutex
它相比 Mutex，就是不限制读和读。
|    | 读   | 写   |
|----| ---- | ---- |
|读 | Y    | N    |
|写 | N    | N    |

RWMutex的结构体有四个字段：
````golang
type RWMutex struct {
	w           Mutex
	writerSem   uint32
	readerSem   uint32
	readerCount int32
	readerWait  int32
}
````
+ w： 复用互斥锁提供的能力
+ writerSem 和 readerSem：分别用于写等待读和读等待写
+ readerCount：存储了当前正在执行的读操作数量
+ readerWait：表示当写操作被阻塞时等待的读操作个数

**方法 ：**
+ RLock()：获取读锁
+ RUnlock()：释放读锁
+ Lock()：获取写锁
+ Unlock()：释放写锁

**关键步骤：**
+ 写锁的独占性
  + rw.Lock() 会锁定写锁（w 字段的互斥锁），阻止其他读写操作
+ 读锁的并发性
  + rw.RLock() 通过增加 readerCount 允许多个读操作并发执行
+ 写优先机制
  + 当写操作因读操作未完成而阻塞时，新来的读操作会被禁止获取锁，直到所有当前读操作完成并唤醒写操作
+ 唤醒逻辑
  + rw.Unlock() 释放写锁后，优先唤醒等待的写操作（若有），否则唤醒所有读操作
  + rw.RUnlock() 在 readerCount 归零时唤醒写操作

### 6.2.4 WaitGroup
sync.WaitGroup 用于等待一组 goroutine 完成，解决主协程需要等待其他子协程执行完的场景。  
waitGroup 的结构体有两个字段：
````golang
type WaitGroup struct {
    noCopy noCopy
    state1 [3]uint32
}
````
+ noCopy：保证 sync.WaitGroup 不会被开发者通过再赋值的方式拷贝
+ state1：保存着状态和信号量
  + 当前活跃的  goroutine 数量
  + 等待该 goroutine 的数量（及调用 wait 的数量）
  + 一个信号量（semaphore），用于阻塞和唤醒goroutine

**方法：**
+ Add(delta int)：（使用原子操作）
  + 通过原子操作更新 state1[0]，增加或减少计数器。如果计数器变为负数，会触发panic。
  + 若计数器归零，会通过信号量（state1[2]）唤醒所有等待的goroutine。
+ Done()：（使用原子操作）
  + 实际调用 Add(-1)，减少计数器。当计数器归零时，触发唤醒逻辑。
+ Wait()：检查计数器是否为零。
  + 若不为零，增加等待计数器（state1[1]），并阻塞当前goroutine，直到被信号量唤醒。
  + 若计数器已为零，直接返回。
  
### 6.2.5 Once
是一个用于确保某个操作仅执行一次的同步工具，它通过原子操作和互斥锁（Mutex）保证线程安全，且性能高效。适用于高并发场景下的初始化、资源加载或单例模式。  
Once 的结构体有两个字段：
````golang
type Once struct {
    done uint32  // 原子操作标记位
    m    Mutex   // 互斥锁
}
````
+ done：使用 atomic.LoadUint32 快速检查是否已执行过。
+ Mutex：互斥锁

**方法：**
+ Do(f func())：
  + 如果传入的函数已经执行过，会直接返回。
  + 如果传入的函数没有执行过，会调用 sync.Once.doSlow 执行传入的函数。
	1. 为当前 Goroutine 获取互斥锁；
	2. 执行传入的无入参函数；
	3. 运行延迟函数调用，将成员变量 done 更新成 1；

### 6.2.6 Cond
是一个用于 goroutine 之间同步的机制。可以让一组的 Goroutine 都在满足特定条件时被唤醒。  
Cond 的结构体有两个字段：
````golang
type Cond struct {
    noCopy noCopy
    L      Locker
}
````
+ L Locker：条件变量绑定的锁。
+ noCopy：防止条件变量被复制，避免潜在的错误。

**方法：**
+ Wait()：当前 goroutine 释放锁并进入等待状态，直到被其他 goroutine 唤醒。
  + 调用 Wait() 时，必须先持有锁（即调用 L.Lock()）。
  + Wait() 会自动释放锁，并将当前 goroutine 挂起。
  + 当被唤醒时，Wait() 会重新获取锁并继续执行。
+ Signal()：唤醒一个正在等待的 goroutine（如果有多个等待的 goroutine，则随机选择一个）。
  + 如果没有 goroutine 在等待，Signal() 不会有任何效果。
+ Broadcast()
  + 唤醒所有正在等待的 goroutine

**使用步骤：**  
+ 使用 sync.Cond 的典型步骤如下：
	1. 创建条件变量，并绑定一个锁。
	2. 在需要等待的 goroutine 中调用 Wait()。
	3. 在条件满足时，调用 Signal() 或 Broadcast() 唤醒等待的 goroutine。


## 6.3 计时器
Golang 中的计时器主要用于控制代码执行的时间逻辑，支持单次(Timer)或周期性(Ticker)任务触发。

### 6.3.1 计时器类型
1. Timer
   + 功能 ：在指定时间后触发一次事件，通过 time.Timer 实现。
   + 重置与停止 ：
		+ Reset()：可重置计时器时间。
		+ Stop()：停止计时器并释放资源，避免 goroutine 泄漏。
   + 适用场景 ：超时控制（如 HTTP 请求）、延迟任务（如缓存失效）。
2. Ticker
	+ 功能 ：按固定间隔重复触发事件，通过 time.Ticker 实现。
	+ 创建方式 ：
		````go
		ticker := time.NewTicker(1 * time.Second)
        for t := range ticker.C {
            fmt.Println("Tick at", t)
        }
		````
	+ 停止 ：调用 Stop() 终止周期行为.
	+ 适用场景：周期性任务（如日志轮转、状态检查）。

### 6.3.2 关键特性与注意事项
+ 资源管理 ：
  + 必须调用 Stop() 停止未使用的 Timer，否则可能导致 goroutine 泄漏。
  + Ticker 在不再需要时也需显式停止。
+ 时间精度 ：
  + Go 计时器基于单调时钟，避免受系统时间调整影响。
  + 实际触发时间可能因调度延迟略晚于设定值。


## 6.4 Channel
Channel 是实现并发编程的核心机制，用于在 Goroutine 之间安全的传输数据和同步操作。

### 6.4.1 设计原理
Golang 的并发设计原理是不要通过共享内存的方式进行通信，而是应该通过通信的方式共享内存。即通信顺序进程（Communicating sequential processes，CSP）。Goroutine 和 Channel 分别对应 CSP 中的实体和传递信息的媒介，Goroutine 之间会通过 Channel 传递数据。

**1. 先入先出**
目前的 Channel 收发操作均遵循了先进先出的设计，具体规则如下：
+ 先从 Channel 读取数据的 Goroutine 会先接收到数据。
+ 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利。  

**2. 缓冲区**
同步与异步 ：
+ 无缓冲 Channel ：发送和接收操作需配对，否则会阻塞（同步通信）。
+ 有缓冲 Channel ：缓冲区未满时发送非阻塞，缓冲区未空时接收非阻塞（异步通信）。

### 6.4.2 数据结构
Golang 的 channel 在运行时使用 runtime.hchan 结构体来表现。
````golang
type hchan struct {
	qcount   uint
	dataqsiz uint
	buf      unsafe.Pointer
	elemsize uint16
	closed   uint32
	elemtype *_type
	sendx    uint
	recvx    uint
	recvq    waitq
	sendq    waitq

	lock mutex
}
````
+ qcount：Channel 中的元素个数；
+ dataqsiz：Channel 中的循环队列的长度；
+ buf：Channel 的缓冲区数据指针；
+ sendx：Channel 的发送操作处理到的位置；
+ recvx：Channel 的接收操作处理到的位置；
+ elemsize：当前 Channel 能够收发的元素类型
+ elemtype：当前 Channel 能够收发的元素大小
+ sendq 和 recvq：当前 Channel 由于缓冲区空间不足而阻塞的 Goroutine 列表

### 6.4.3 创建与使用方式
**声明与初始化：**
````golang
// 无缓冲 Channel（同步）
ch1 := make(chan int)

// 有缓冲 Channel（异步，容量为 5）
ch2 := make(chan string, 5)
````
未初始化的 Channel 无法直接使用，读写会引发死锁。  
**使用方式：**  
+ 发送数据 ：
  + ch <- value 将值写入 Channel，若缓冲区满或无缓冲则阻塞。  
+ 接收数据 ：
  + value := <-ch 从 Channel 读取值，若无数据则阻塞。  
+ 关闭 Channel ：
  + close(ch) 通知接收方无更多数据，重复关闭会引发 panic。
