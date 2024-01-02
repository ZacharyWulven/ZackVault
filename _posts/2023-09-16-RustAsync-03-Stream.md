---
layout: post
title: RustAsync-03-Stream
date: 2023-09-16 16:45:30.000000000 +09:00
categories: [Rust, Rust Async]
tags: [Rust, Rust Async]
---


# 5 Stream

## Stream trait
* `Stream trait` 和 `Future trait` 类似，但它可以在完成前产生多个值，这点和标准库 `Iterator trait` 也很像

* Stream 实现

```rust
// Stream 
trait Stream {
    /// The type of the value yielded by the stream.
    type Item; // Stream 能产生的值的类型

    /// Attempt to resolve the next item in the stream.
    /// Returns `Poll::Pending` if not ready, `Poll::Ready(Some(x))` if a value
    /// is ready, and `Poll::Ready(None)` if the stream has completed.
    
    /// 尝试解析 Stream 里下一个数据项
    /// 如果没有准备好就返回 `Poll::Pending`
    /// 如果准备好了就返回 `Poll::Ready(Some(x))`
    /// 如果结束了就返回 `Poll::Ready(None)`
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<Self::Item>>;
}
```

* Stream 例子

```rust
async fn send_recv() {
    const BUFFER_SIZE: usize = 10;
    let (mut tx, mut rx) = mpsc::channel::<i32>(BUFFER_SIZE); // 创建管道

    tx.send(1).await.unwrap(); // 发送数据
    tx.send(2).await.unwrap(); // 发送数据
    drop(tx);

    // `StreamExt::next` is similar to `Iterator::next`, but returns a
    // type that implements `Future<Output = Option<T>>`.
    assert_eq!(Some(1), rx.next().await);
    assert_eq!(Some(2), rx.next().await);
    assert_eq!(None, rx.next().await);    // 最后 stream 完成了，接收端收到的是 None
}
```

## Iteration And Concurrency（迭代与并发）

* 与同步的 `Iterator` 类似，有很多种方法迭代和处理 `Stream` 中的值：
1. 组合器风格：`map、filter、fold`
2. 相应的 `early-exit-on-error` 版本（组合器遇到错误就退出的版本）：`try_map、try_filter、try_fold`

> `for` 循环无法和 `Stream` 一起使用，命令式的 `while let` 和 `next/try_next` 函数可以与 `Stream` 一起使用
{: .prompt-info }

* 例子：累加求和

```rust
async fn sum_with_next(mut stream: Pin<&mut dyn Stream<Item = i32>>) -> i32 {
    use futures::stream::StreamExt; // for `next`, next 方法来自于 futures::stream::StreamExt
    let mut sum = 0;
    while let Some(item) = stream.next().await { // 类似 iterator，对流中的数据进行累加
        sum += item;
    }
    sum
}

async fn sum_with_try_next(
    mut stream: Pin<&mut dyn Stream<Item = Result<i32, io::Error>>>,
) -> Result<i32, io::Error> {
    use futures::stream::TryStreamExt; // for `try_next`，try_next 方法来自于 futures::stream::TryStreamExt
    let mut sum = 0;
    while let Some(item) = stream.try_next().await? { // 与上边差不多，这里多个 ?，返回 Result 类型
        sum += item;
    }
    Ok(sum)
}
```


* 例子：从一个 Stream 中处理多个值

```rust
async fn jump_around(
    mut stream: Pin<&mut dyn Stream<Item = Result<u8, io::Error>>>,
) -> Result<(), io::Error> {
    use futures::stream::TryStreamExt; // for `try_for_each_concurrent`
    
    const MAX_CONCURRENT_JUMPERS: usize = 100;  // 设置最大并发为 100

    // 当 stream 里数据可用时，这个方法就会尝试并发的对流里数据执行闭包里的逻辑，
    // 如果遇到错误就退出，所以最后 .await 后边有个 ?
    stream.try_for_each_concurrent(MAX_CONCURRENT_JUMPERS, |num| async move {
        jump_n_times(num).await?;
        report_n_jumps(num).await?;
        Ok(())
    }).await?;

    Ok(())
}
```

# 6 Executing Multiple Futures at a Time（同时执行多个 Future）

* 真正的异步应用程序通常需要同时执行几个不同的操作
* 本章介绍一些可同时执行多个异步操作的方式
1. `join!`：用于等待所有的 Future 完成
2. `select!`：用于等待多个 Future 中的一个完成
3. `Spawning`：创建一个顶级任务，它会运行一个 Future 直到其完成
4. `FuturesUnordered`：一组 Future，它们会产生每个子 Future 的结果


## 6.1 `join!` 宏
* 来源于 `futures::join` 宏，它使得在等待多个 Future 完成的时候，可以同时并发的执行它们


* 错误的姿势，下面并不会同时执行，还是串行的

```rust
async fn get_book_and_music() -> (Book, Music) {
    let book = get_book().await;
    let music = get_music().await;
    (book, music)
}

// WRONG -- don't do this
async fn get_book_and_music() -> (Book, Music) {
    let book_future = get_book();
    let music_future = get_music();
    (book_future.await, music_future.await)
}
```

* `join!` 正确的姿势，两个 `Future` 并发的执行

```rust
use futures::join;

async fn get_book_and_music() -> (Book, Music) {
    let book_fut = get_book();
    let music_fut = get_music();
    join!(book_fut, music_fut)
}
```

### try_join!
* 对于返回 `Result` 的 `Future`，应该优先考虑使用 `try_join!` 
1. 如果`子 Future` 中某一个返回了错误，`try_join!` 会立即完成

```rust
use futures::try_join;

async fn get_book() -> Result<Book, String> { /* ... */ Ok(Book) }
async fn get_music() -> Result<Music, String> { /* ... */ Ok(Music) }

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book();
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```

> Note：`book_fut` 和 `music_fut` 这个两个 `Future` 的错误类型必须是一致的。这里都是 `String` 
{: .prompt-info }

* 如果上边错误类型不一致怎么办呢？
1. 下例错误类型一个是 `()`，一个是 `String`

* 解决方案：可以使用 `futures::future::TryFutureExt` 的 `.map_err(|e| ...)` 和 `.err_into()` 函数来对错误类型进行转换


```rust
use futures::{
    future::TryFutureExt,
    try_join,
};

async fn get_book() -> Result<Book, ()> { /* ... */ Ok(Book) }
async fn get_music() -> Result<Music, String> { /* ... */ Ok(Music) }

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book().map_err(|()| "Unable to get book".to_string());
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```

## 6.2 `select!` 宏
* `futures::select` 宏可以同时运行多个 `Future`，并且允许用户在任意一个 `Future` 完成时进行响应

```rust
use futures::{
    future::FutureExt, // for `.fuse()`
    pin_mut,
    select,
};

async fn task_one() { /* ... */ }
async fn task_two() { /* ... */ }

async fn race_tasks() {
    let t1 = task_one().fuse();
    let t2 = task_two().fuse();

    pin_mut!(t1, t2);  // 把 t1，t2 Pin 一下

    /*
      调用 `select!` 宏
      这里意思就是并发的运行 t1，t2 这俩任务，等其中一个完成的时候就会打印相应的语句，
      这时整个函数也就完成，它并不会等待另一个任务的完成。即只要一个完成了就完成了
    */
    select! {
        () = t1 => println!("task one completed first"),
        () = t2 => println!("task two completed first"),
    }
}
```

### 与 `Unpin` 和 `FusedFuture` 交互
* 上边代码，需要在返回的 `Future` 上调用 `.fuse()`，也调用了 `pin_mut` 
1. 这是因为 `select` 里边的 `Future` 必须实现 `Unpin` 和 `FusedFuture` 这两个 `trait`

* 为什么必须 `Unpin` 呢？
1. 因为 `select` 使用的 `Future` 不是按值的，而是按可变引用，所以它没有获得所有权
2. 未完成的 `Future` 在调用 `select` 后仍然是可以使用的

* 为什么必须 `FusedFuture` 呢？
1. 因为在 `Future` 完成后，`select` 就不可以对它进行 `poll` 了
2. 实现了 `FusedFuture` 的 `Future 会追踪其完成状态，这样在 `select` 循环里就只会 `poll` 那些没有完成的 `Future`，而完成的 `Future` 就不会再被 `poll` 了


### `select` 中的 `default` 和 `complete` 分支
* `select!` 宏，它支持 `default` 和 `complete` 分支

* `default` 分支：即如果选中的 `Future` 尚未完成，就会运行 `default` 分支
1. 而拥有 `default` 分支的 `select!` 它总是会立即返回

* `complete` 分支：即用于所有选中的 Future `都已完成`的情况，经常与 `loop` 一起用

```rust
use futures::{future, select};

async fn count() {
    let mut a_fut = future::ready(4);
    let mut b_fut = future::ready(6);
    let mut total = 0;

    loop {
        select! {
            a = a_fut => total += a,
            b = b_fut => total += b,
            complete => break,
            // 每次 loop 看 select 中 a 或 b 如果都未完成，就立即返回 `default` 分支
            default => unreachable!(), // never runs (futures are ready, then complete)
        };
    }
    assert_eq!(total, 10);
}
```

### Stream 也有 FusedStream trait

```rust
use futures::{
    stream::{Stream, StreamExt, FusedStream},
    select,
};

async fn add_two_streams(
    mut s1: impl Stream<Item = u8> + FusedStream + Unpin, // 参数必须实现 FusedStream 和 Unpin trait
    mut s2: impl Stream<Item = u8> + FusedStream + Unpin,
) -> u8 {
    let mut total = 0;

    loop {
        let item = select! {
            // Note：
            // 调用 next 或 try_next 就会返回 FusedFuture，所以它们也可以在 select! 宏中使用
            // 这个返回的 FusedFuture 在完成之后就不会再被 poll 了
            x = s1.next() => x, 
            x = s2.next() => x,
            complete => break,
        };
        if let Some(next_num) = item {
            total += next_num;
        }
    }

    total
}
```

### 在 `select` 循环里使用 `Fuse 和 FuturesUnordered`
* `Fuse::terminated()` 允许构建空的、已完成的 `Future`，后续可以为它填充一个需要运行的 `Future`
1. 适用场景：在 `select` 循环里产生并且需要在这运行的任务

```rust
use futures::{
    future::{Fuse, FusedFuture, FutureExt},
    stream::{FusedStream, Stream, StreamExt},
    pin_mut,
    select,
};

async fn get_new_num() -> u8 { /* ... */ 5 } // 返回一个数 5

async fn run_on_new_num(_: u8) { /* ... */ } // 运行在这个新的数上

// 循环异步函数
async fn run_loop(
    mut interval_timer: impl Stream<Item = ()> + FusedStream + Unpin, // 定时器
    starting_num: u8,
) {
    let run_on_new_num_fut = run_on_new_num(starting_num).fuse(); // 返回一个 FusedFuture
    let get_new_num_fut = Fuse::terminated();                     // 创建一个空的已经完成的 Future，后续可为它填充一个要运行的 Future
    pin_mut!(run_on_new_num_fut, get_new_num_fut);
    loop {
        select! {
            // select_next_some 如果返回一些值才会运行里边的代码
            () = interval_timer.select_next_some() => {
                // The timer has elapsed. Start a new `get_new_num_fut`
                // if one was not already running.
                // 如果定时器时间已过，没有正在运行的任务，
                // 这时就开始一个新的 `get_new_num_fut`，就是使用 Fuse::terminated() 创建的那个空 Future
                
                if get_new_num_fut.is_terminated() { // 判断是否终止，如果已终止就给它设置一个值
                    get_new_num_fut.set(get_new_num().fuse()); // 这个 Future 就相对于在 select 中创建的 Future
                }
            },
            new_num = get_new_num_fut => { 
                // A new number has arrived -- start a new `run_on_new_num_fut`,
                // dropping the old one.
                // 新的数到达了，就开始一个新的 `run_on_new_num_fut`
                // 丢弃旧的
                run_on_new_num_fut.set(run_on_new_num(new_num).fuse());
            },
            // Run the `run_on_new_num_fut`
            // 运行 `run_on_new_num_fut`
            () = run_on_new_num_fut => {},
            // panic if everything completed, since the `interval_timer` should
            // keep yielding values indefinitely.
            // 如果完成了就 panic， `interval_timer` 应该一直产生值
            complete => panic!("`interval_timer` completed unexpectedly"),
        }
    }
}
```

* 当同一个 `Future` 的多个副本需要同时运行时，使用 `FuturesUnordered` 类型

```rust
use futures::{
    future::{Fuse, FusedFuture, FutureExt},
    stream::{FusedStream, FuturesUnordered, Stream, StreamExt},
    pin_mut,
    select,
};

async fn get_new_num() -> u8 { /* ... */ 5 }

async fn run_on_new_num(_: u8) -> u8 { /* ... */ 5 }

async fn run_loop(
    mut interval_timer: impl Stream<Item = ()> + FusedStream + Unpin,
    starting_num: u8,
) {
    // 创建一个空的 FuturesUnordered 类型，可容纳同类型 Future 容器
    let mut run_on_new_num_futs = FuturesUnordered::new();  
    
    // 添加一个 Future 到容器中
    run_on_new_num_futs.push(run_on_new_num(starting_num)); 
    
    let get_new_num_fut = Fuse::terminated();
    pin_mut!(get_new_num_fut);
    loop {
        select! {
            () = interval_timer.select_next_some() => {
                // The timer has elapsed. Start a new `get_new_num_fut`
                // if one was not already running.
                if get_new_num_fut.is_terminated() {
                    get_new_num_fut.set(get_new_num().fuse());
                }
            },
            new_num = get_new_num_fut => {
                // A new number has arrived -- start a new `run_on_new_num_fut`.
                // 新的数到达了，就开始一个新的 `run_on_new_num_fut`
                run_on_new_num_futs.push(run_on_new_num(new_num));
            },
            // Run the `run_on_new_num_futs` and check if any have completed
            // 检查一下 run_on_new_num_futs 里边是否有任意一个 Future 已经完成了
            // 如果有的话旧打印后边的代码
            res = run_on_new_num_futs.select_next_some() => {
                println!("run_on_new_num_fut returned {:?}", res);
            },
            // panic if everything completed, since the `interval_timer` should
            // keep yielding values indefinitely.
            complete => panic!("`interval_timer` completed unexpectedly"),
        }
    }
}

```

# 7 一些问题的临时解决方案

## 7.1 `async` 块中的 ？
* `async` 块中使用 ? 是比较常见的，但是 `async` 块的返回类型没有明确说明，这种情况就可能导致编译器无法推断出 `async` 块的错误类型


* 下边代码 `Error`, 无法推断 `Result` 上边的 `E` 的类型
```rust
#![allow(unused)]
fn main() {
    struct MyError;
    
    async fn foo() -> Result<(), MyError> {
        Ok(())
    }

    async fn bar() -> Result<(), MyError> {
        Ok(())
    }
    
    // 目前无法给 fut 一个类型，也无法指明其类型
    let fut = async {
        foo().await?;
        bar().await?;
        Ok(()) // Error, 无法推断 Result 上边的 E 的类型
        Ok::<(), MyError>(()) // 临时解决方案
    };
}
```

> 临时解决方案：使用 `turbofish` 运算符，为 `async 块` 提供成功和错误类型。返回这样即可： `Ok::<(), MyError>(())`
{: .prompt-info }


## 7.2 Send Trait Approximation
* 有些 `async fn` 状态机是可以安全的跨线程发送的，而有些则不行
* `async fn` 是否是 `Send` 的，取决于在 `.await` 点是否持有非 `Send` 的类型（看它持有的里边是否有非 Send 类型的）
* 当值可能在跨越 `.await` 点被持有时，编译器会尽力近似估算，但这种估算在很多地方显得过于保守了


```rust
use std::rc::Rc;
#[derive(Default)]
struct NotSend(Rc<()>); // 不是 Send 的，里边有个 Rc
async fn bar() {}
async fn foo() {
    //  因为 NotSend::default() 这样调用，只在 foo 里出现了一下，这样是没有问题的，编译器不会报错
    // NotSend::default();
    
    // 但如果我们获取 NotSend 的返回值，下边 main 函数就会报错
    // Error: future returned by `foo` is not `Send
    // let x = NotSend::default();

    // 临时解决方案：引入块作用域
    {
        let x = NotSend::default();

    }

    bar().await;
}

fn require_send(_: impl Send) {}

fn main() {
    require_send(foo());
}
```

> 临时解决方案：引入块作用域，把 `non-Send` 变量隔离
{: .prompt-info }

## 7.3 Recursion（递归）
* 在内部，`async fn` 会创建一个状态机类型，它包含每个被 `.await` 的子 `Future`
1. 这点就有点麻烦，因为状态机是需要包含其本身的

```rust
async fn foo() {
    step_one().await;
    step_two().await;
}
// 编译后对 foo 函数解析，产生的类型
enum Foo {
   First(StepOne),
   Second(StepTwo),
}



async fn recursive() {
    recursive().await;
    recursive().await;
}
/*
    编译后对 recursive 函数解析，产生的类型
    这样就不行了，因为这样会产生一个无限大小的类型，编译器无法知道其大小，这时就会报错
    Error: a recursive `async fn` must be rewritten to return a boxed `dyn Future`
*/
enum Recursive {
    First(Recursive),
    Second(Recursive),
}
```

* 递归的临时解决方案：使用 `Box`，并把 `recursive` 放入非 `async` 的函数，它会返回 `.boxed() async 块`

```rust
// .toml
[dependencies]
futures = "0.3.21"


// main.rs
use futures::future::{BoxFuture, FutureExt};

fn recursive() -> BoxFuture<'static, ()> {
    async move {
        recursive().await;
        recursive().await;
    }.boxed()
}
```

## 7.4 async in Traits
* 目前 `async fn` 不可以在 `trait 的定义` 中使用

* 临时解决方案：使用第三方 `crate`，它叫 `async-trait`


# 8 async 生态
* Rust 在 `async` 这块提供的其实不多，Rust 目前只提供编写 `async` 代码的基本要素，标准库中尚未提供执行器、任务、反应器、组合器、以及低级 `I/O future` 和 `trait`，
而社区提供的 `async` 生态系统填补了这些空白

## async runtime
* `async` 运行时是用于执行 `async` 应用程序的库
* 运行时通常是将一个反应器与一个或多个执行器捆绑在一起
* 反应器为`异步 I/O`、进程间通信、计时器等外部事件提供订阅机制
* 在 `async` 运行时中，订阅者通常是代表`低级别 I/O 操作的 Future`
* 执行者（器）负责安排和执行任务
1. 它们跟踪正在运行和暂停的任务，对 Future 进行 poll 直到完成，并在任务能够取得进展时唤醒任务
2. `执行者`一词经常与`运行时`互换使用
* 我们使用 `生态系统` 一词来描述一个与兼容 `trait` 和特性捆绑在一起的运行时


## 社区提供的 `async crates`
* 提供了 `futures crate`：里边包含 `Stream、Sink、AsyncRead、AsyncWrite 等 trait`，以及组合器等工具，这些可能最终成为标准库的一部分。
* `futures crate` 有自己的执行器，但没有自己的反应器，因此它不支持 `async I/O` 或计时器相关 `Future` 的执行，因此它并不认为是完整的运行时
* 通常的选择是：使用另外一个 `crate` 中的执行器与 `futures crate` 提供的工具一起使用


## 比较流行的运行时
1. `Tokio`：是一个流行的 `async` 生态系统，包含 `HTTP`、`gRPC` 和跟踪框架
2. `async-std`: 提供标准库的 `async` 副本，语法参照的标准库
3. `smol`: 小型、简化的 `async` 运行时。提供可用于保证 `UnixStream` 或 `TcpListener` 等结构的 `async trait`
4. `fuchsia-async`: 用于 `Fuchsia OS` 的执行器


## 确定生态兼容性（非常重要）
* 与 `async I/O`、计时器、进程间通信或任务交互的 `async` 代码通常取决于特定的异步执行器或反应器（这些与生态有关）
* 所有其他 `async` 代码，如异步表达式、组合器、同步类型和流，通常与生态系统无关。无关也是有个前提，即任何嵌套的 `future` 也与生态系统无关


> 在开始一个项目之前，建议研究相关的 `async` 框架和库，以确保与您选择的运行时以及彼此之间的兼容性。
{: .prompt-info }


## 单线程 VS 多线程执行器
* `async` 执行器可以使单线程或多线程的
* 多线程执行器可以在多个任务上同时取得进展，对于有许多任务的工作负载，它可以大大加快执行速度，但在任务之间同步数据通常更昂贵
* 在单线程运行时和多线程运行时之间进行选择时，建议测量应用程序的性能
* 任务可以在创建它们的线程上运行，也可以在单独的线程上运行
* 异步运行时通常提供将任务生成到单独线程的功能
1. 即使任务在不同的线程上执行，它们也应该是非阻塞的
* 为了在多线程执行器上调度任务，它们必须是 `Send` 的
* 一些运行时提供了生成非 `Send` 的任务的函数，确保每个任务都在生成它的线程上执行
* 它们还可以提供将阻塞任务生成到专用线程的函数，这对于运行来自其他库的阻塞同步代码非常有用


# 9 项目：并发的 WebServer

## 构建并发的（单线程的）Web Server
* 使用 `async Rust` 把 `《The Rust Programming Language》` 中的第 20 张例子改为可并发处理请求的 Web Server


### 非 async 版本

```rust
use std::fs;
use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;

fn main() {
    // Listen for incoming TCP connections on localhost port 7878
    // 监听进来的 TCP 链接，绑定到本地的 7878 端口
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    // Block forever, handling each request that arrives at this IP address
    // 使用 listener 监听进来的请求，
    // 永远阻塞，处理每个到达此 IP 的请求
    // stream 类型为 Result<TcpStream, Error>
    for stream in listener.incoming() {
        let stream = stream.unwrap();

        handle_connection(stream);
    }
}

fn handle_connection(mut stream: TcpStream) {
    // Read the first 1024 bytes of data from the stream
    // 从 Stream 里读取数据的前 1024 个字节
    let mut buffer = [0; 1024];
    stream.read(&mut buffer).unwrap();

    let get = b"GET / HTTP/1.1\r\n";

    // Respond with greetings or a 404,
    // depending on the data in the request
    // 根据请求中的数据响应 200 或 404
    // 如果以 get 开头，就返回 hello.html
    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "404.html")
    };
    // 然后读取 html 的内容
    let contents = fs::read_to_string(filename).unwrap();

    // Write response back to the stream,
    // and flush the stream to ensure the response is sent back to the client
    // 将响应写回 Stream，并 flush Stream
    // 保证响应被送回客户端
    let response = format!("{status_line}{contents}");
    stream.write_all(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```


### 改为 async 版本
* 添加 futures、async-std依赖

* Cargo.toml

```rust
[package]
name = "web_server_09"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
async-std = {version = "1.11.0", features = ["attributes"]} // 这里启用 attributes 这个特性
futures = "0.3.21"
```

* main.rs

```rust
use async_std::{ net::{TcpListener, TcpStream}, prelude::*, task::{self, spawn} };
use futures::stream::StreamExt;
use std::fs;
use std::time::Duration;

// use for tests 
use async_std::io::{Read, Write};


#[async_std::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").await.unwrap();
    /*
        标准库中的 TcpListener 的 incoming() 是阻塞的，
        这里 async_std 的 TcpListener 的 incoming() 就不是非阻塞的，它返回一个流

        for_each_concurrent 用于并发处理 stream 中的元素
        参数 1：是并发处理最大极限，这里传 None
        参数 2：是个闭包
     */
    listener
        .incoming()
        .for_each_concurrent(/* limit */ None, |tcpstream| async move {
            let tcpstream = tcpstream.unwrap();
            //handle_connection(tcpstream).await;
            spawn(handle_connection(tcpstream));  // 使用 `spawn` 放到单独的线程中执行
        })
        .await;
}
// TcpStream 来自 async_std，之前 TcpStream 是标准库的
// async fn handle_connection(mut stream: TcpStream) {
async fn handle_connection(mut stream: impl Read + Write + Unpin) {

    let mut buffer = [0; 1024];
    stream.read(&mut buffer).await.unwrap(); // async version

    let get = b"GET / HTTP/1.1\r\n";
    let sleep = b"GET /sleep HTTP/1.1\r\n";

    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else if buffer.starts_with(sleep) {
        task::sleep(Duration::from_secs(5)).await;
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "404.html")
    };
    let contents = fs::read_to_string(filename).unwrap();

    let response = format!("{status_line}{contents}");
    stream.write(response.as_bytes()).await.unwrap();
    stream.flush().await.unwrap();
}


// 测试代码
#[cfg(test)]
mod tests {
    use super::*;
    use futures::io::Error;
    use futures::task::{Context, Poll};
    use std::cmp::min;
    use std::pin::Pin;

    struct  MockTcpStream {
        read_data: Vec<u8>,
        write_data: Vec<u8>,
    }

    impl Read for MockTcpStream {
        fn poll_read(
                    self: Pin<&mut Self>,
                    cx: &mut Context<'_>,
                    buf: &mut [u8],
                ) -> Poll<std::io::Result<usize>> {
                    // 读取 read_data 长度与 buffer 长度中比较小的值
                    let size: usize = min(self.read_data.len(), buf.len());
                    // 将 read_data 数据 copy 到 buffer 中
                    buf[..size].copy_from_slice(&self.read_data[..size]);
                    // 返回 Ready 表示完成
                    Poll::Ready(Ok(size))
        }
    }

    impl Write for MockTcpStream {
        // 把数据写入 TcpStream，
        fn poll_write(mut
                    self: Pin<&mut Self>,
                    cx: &mut Context<'_>,
                    buf: &[u8],
                ) -> Poll<std::io::Result<usize>> {
                    self.write_data = Vec::from(buf);
                    Poll::Ready(Ok(buf.len()))   
        }

        // 针对 MockTcpStream 而言，poll_flush 和 poll_close 就没啥用，返回 Poll::Ready 就可以了
        fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<std::io::Result<()>> {
            Poll::Ready(Ok(()))
        }

        fn poll_close(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<std::io::Result<()>> {
            Poll::Ready(Ok(()))
        }
    }

    // 实现 Unpin
    use std::marker::Unpin;
    impl Unpin for MockTcpStream {


    }

    use std::fs;

    #[async_std::test] // 标记为异步测试函数
    async fn test_handle_connection() {
        let input_bytes = b"GET / HTTP/1.1\r\n";
        let mut contents = vec![0u8; 1024];
        contents[..input_bytes.len()].clone_from_slice(input_bytes);

        // mock MockTcpStream
        let mut stream = MockTcpStream {
            read_data: contents,
            write_data: Vec::new(),
        };

        handle_connection(&mut stream).await;
        // let mut buf: [u8; 1024] = [0u8; 1024];
        // stream.read(&mut buf).await.unwrap();

        let expected_contents = fs::read_to_string("hello.html").unwrap();
        let expected_response = format!("HTTP/1.1 200 OK\r\n\r\n{}", expected_contents);
        // 看 stream.write_data 是不是以 expected_response 开头
        assert!(stream.write_data.starts_with(expected_response.as_bytes()));
    }

}
```
* `main` 函数中 `handle_connection` 处理，虽然是并发的但是都是在单线程中执行的，而由于 `handle_connection` 函数是非阻塞的并且也是 `Send` 的，
所以可以使用 `spawn` 放到单独的线程中执行




