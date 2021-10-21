# Go 语言中 Select 高级用法

Golang 的 select 的功能和 `select, poll, epoll` 相似， 就是监听 IO 操作，当 IO 操作发生时，触发相应的动作。



##使用 Select 实现 资源监控机制

功能：

- 利用 Select 的 Default 特性，监测资源是否已经全部被占用

```go
func main() {
  ch := make(chan int, 1)
  ch <- 1  // 向通道发送数据，此时通道已满
  
  select {
    case ch <- 2:
    // 因为通道已满，此时通道被阻塞
    default:
    fmt.Println("channel is full!")
  }
}
```

解析：

因为 ch 在创建后，直接发送了数据，导致通道已满；在 Select 执行 `case ch <- 2`时，会被阻塞；此时，Select 会去执行 default 分支。这样，可以实现对 Channel 是否已满的检测，而不会一直等待。



如果我们有一个服务，当请求进来的时候会生成一个 Job 并扔进 Channel 中，其他处理 Go 程或去从 Channel 中获取 Job 并执行；使用该方式，可以实现当 Job 达到设置的上限值时（如 Channel 缓存数量），可以友好的返回`【服务繁忙，请稍后再试】`信息，而不会一直阻塞。



## 使用 Select 实现 Timout 及超时时间刷新机制

功能：

- 主Go程或其他Go程，可以主动请求退出：使用 isQuit 通道
- 主Go程或其他Go程，可主动刷新超时时间：使用 isReset 通道
- 主动监测超时时间，在一段时间没有相应操作时，超时退出：使用 time.After() 功能

```go
func main() {
  isQuit := make(chan bool)
  isReset := make(chan bool)
  
  // 业务逻辑
  for {
    // 业务在处理中时，刷新超时标识
    isReset <- true  // 如果chan对方无判断，传输的值无所谓
  }
Quit

func watch(isQuit,isReset chan bool) {
  for {
    select {
      case <-isQuit:
      fmt.Println("客户端请求退出")
      //相应的退出处理流程
      return
      case <-time.After(3 * time.Second):
      fmt.Println("超时退出")
      //相应的超时退出处理流程
      return
      case <-isReset:
      fmt.Println("业务处理中，刷新超时时间")
    }
  }
}
```

解析：

使用 watch() 函数进行退出信号的检测、超时检测，并进行相应的退出操作。

- 接收到 isQuit 信号时，主动进行清理及关闭资源操作
- 监测超时时间，在超时时间到达时，主动进行清理及关闭资源操作
- 接收到 isReset 信号时，激活 Select，使 Select 进入下一次检测循环；此时 time.After() 会创建新的定时器，重新计时



## 参考

- [Go语言学习-select用法](https://blog.csdn.net/zhonglinzhang/article/details/45913443)

  