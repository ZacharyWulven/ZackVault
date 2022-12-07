---
layout: post
title: Go 语言设计-07-goroutine 和通道
date: 2022-07-24 16:45:30.000000000 +09:00
tag: Go
---


Go 有两种并发编程的风格，
* goroutine 和通道，它们支持通信顺序进程（Communicating Sequential Process, CSP）CSP 是一种并发的模式，在不同的 goroutine 之间传递值，
但是变量本身局限于单一的 goroutine。
* 共享内存多线程传统模型，它们和在其他主流语言中使用线程类似

## 0x01 goroutine
在 Go 中每一个并发执行活动称为 goroutine，如果你使用过操作系统或其他语言中的线程，可以假设 goroutine 类似于线程。
但 goroutine 和线程之间在数量上有非常大的差别。

{% highlight ruby %}
当一个程序启动时，只有一个 goroutine 来调用 main 函数，称它为主 goroutine。
{% endhighlight %}

新的 goroutine 通过 go 语言进行创建，语法上在函数或方法前加 go 关键字。

{% highlight ruby %}
go 语句使函数在一个新创建的 goroutine 中调用。go 语句本身的执行立即完成

f() //调用 f()，等待它返回
go f() // 新建一个调用 f() 的 goroutine，不用等待
{% endhighlight %}

## 0x02 时钟例子
{% highlight ruby %}
clock1.go

func main() {
  // 创建一个 net.Listen 对象，它在一个网络端口上监听进来的连接，这里使用 TCP 端口 8000
  listener, err := net.Listen("tcp", "localhost:8000")
  if err != nil {
    log.Fatal(err)
  }
  for {
    // 监听器的 Accept 方法被阻塞，直到有连接请求进来，然后返回 net.Conn 对象来代表一个连接
    conn, err := listener.Accept()
    if err != nil {
      log.Print(err)
      continue
    }
    handleConn(conn) // 一次处理一个连接
  }

}

func handleConn(c net.Conn) {
  defer c.Close()
  for {
    // time.Time.Format 提供了格式化日期和时间的方式，参数是一个模板
    _, err := io.WriteString(c, time.Now().Format("15:04:05\n"))
    if err != nil {
      return
    }
    time.Sleep(1 * time.Second)
  }
}
{% endhighlight %}

* 为了连接服务器需要像 nc 这样的程序；$ nc localhost 8000
或用 net.Dial 实现 Go 版本的 netcat 程序，如下

{% highlight ruby %}
netcat1.go
/*
  这个程序从网络连接中读取，然后写入标准输出，直到 EOF 或出错
*/
func main() {
  conn, err := net.Dial("tcp", "localhost:8000")
  if err != nil {
    log.Fatal(err)
  }
  defer conn.Close()
  mustCopy(os.Stdout, conn)
}
func mustCopy(dst io.Writer, src io.Reader) {
  if _, err := io.Copy(dst, src); err != nil {
    log.Fatal(err)
  }
}

$ killall clock1
killall 命令时 UNIX 一个实用程序用来终止所有指定名字的进程
{% endhighlight %}

clock1.go 不支持多个客户端同时请求，只能等一个客户端请求完才能继续下一个，这时需要使用 go
{% highlight ruby %}
clock2.go
只需在 handleConn 调用前加 go
$./clock2 &

func main() {
  // 创建一个 net.Listen 对象，它在一个网络端口上监听进来的连接，这里使用 TCP 端口 8000
  listener, err := net.Listen("tcp", "localhost:8000")
  if err != nil {
    log.Fatal(err)
  }
  for {
    // 监听器的 Accept 方法被阻塞，直到有连接请求进来，然后返回 net.Conn 对象来代表一个连接
    conn, err := listener.Accept()
    if err != nil {
      log.Print(err)
      continue
    }
    go handleConn(conn) // 并发处理连接
  }

}
{% endhighlight %}

## 0x03 并发回声服务器
{% highlight ruby %}
reverb1.go

func main() {
  // 创建一个 net.Listen 对象，它在一个网络端口上监听进来的连接，这里使用 TCP 端口 8000
  listener, err := net.Listen("tcp", "localhost:8000")
  if err != nil {
    log.Fatal(err)
  }
  for {
    // 监听器的 Accept 方法被阻塞，直到有连接请求进来，然后返回 net.Conn 对象来代表一个连接
    conn, err := listener.Accept()
    if err != nil {
      log.Print(err)
      continue
    }
    go handleConn3(conn) // 一次处理一个连接
  }

}

func echo(c net.Conn, shout string, delay time.Duration) {
  fmt.Fprintln(c, "\t", strings.ToUpper(shout))
  time.Sleep(delay)
  fmt.Fprintln(c, "\t", shout)
  time.Sleep(delay)
  fmt.Fprintln(c, "\t", strings.ToLower(shout))

}

func handleConn3(c net.Conn) {

  input := bufio.NewScanner(c)
  for input.Scan() {
    echo(c, input.Text(), 1*time.Second)
  }
  // 注意忽略 input.Err() 中的错误
  c.Close()

}

{% endhighlight %}

{% highlight ruby %}
netcat2.go

func main() {
  conn, err := net.Dial("tcp", "localhost:8000")
  if err != nil {
    log.Fatal(err)
  }
  defer conn.Close()
  go mustCopy2(os.Stdout, conn)
  mustCopy2(conn, os.Stdin)
}

func mustCopy2(dst io.Writer, src io.Reader) {
  if _, err := io.Copy(dst, src); err != nil {
    log.Fatal(err)
  }
}

{% endhighlight %}

## 0x04 通道
* 如果说 goroutine 是 go 程序并发执行体，通道就是它们之间的连接。
* 通道是可以让一个 goroutine 发送特定的值到另一个 goroutine 的通信机制
* 每一个通道是一个具体类型的导管，叫做通道的元素类型
* 一个有 int 类型元素的通道写为 chan int
* 使用内置的 make 创建一个通道 ch := make(chan int) // ch 的类型是 'chan int'
* 通道是一个用 make 创建的数据结构的引用，复制或参数传递到函数时复制的是引用，这样调用者和被调用者使用同一份引用
* 通道的零值是 nil
* 同类型的通道可以用 == 进行比较，当二者是同一数据的引用时比较结果是 true，也可以跟 nil 进行比较

### 通道有连个主要操作，两者统称为通信，俩个操作都是要 <- 符号进行书写
1. 发送 send
* send 语句从一个 goroutine 传输一个值到另一个在执行接收表达式的 goroutine
* send 语句中 通道和值分别在 <- 左右两边
2. 接收 receive
* 在 receive 表达式中 <- 放在通道操作数前面
* 在 receive 表达式中，其结果未被使用也是合法的
{% highlight ruby %}
ch <- x   // 发送语句
x = <-ch  // 赋值语句中的接受表达式
<- ch 接收语句，丢弃结果
{% endhighlight %}

3. 通道支持的第三个操作，关闭（close）
* 它设置一个标志位来指示当前已经发送完毕，这个通道后面没有值了
* 关闭后的发送操作将导致宕机
* 在一个已经关闭的通道上进行接收操作，将获取所有已经发送的值，直到通道为空，这时任何接收操作会立即完成，同时获取到一个通道元素类型对应的零值。
* 调用内置函数 close(ch) 关闭通道

### 无缓冲通道
使用 make 创建的通道叫无缓冲（unbuffered）通道，但 make 还可以接受第二个可选的参数，一个表示通道容量的整数，如果容量是 0，make 创建无缓冲通道。
{% highlight ruby %}
ch = make(chan int)     // 无缓冲通道
ch = make(chan int, 0)  // 无缓冲通道
ch = make(chan int, 3)  // 容量为 3 的缓冲通道
{% endhighlight %}
