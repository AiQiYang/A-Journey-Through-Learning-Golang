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
