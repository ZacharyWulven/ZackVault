---
layout: post
title: Rust Async 简易教程
date: 2024-01-01 16:45:30.000000000 +09:00
categories: [Rust, Rust Async]
tags: [Rust, Rust Async]
---


# 一个简单例子

* 使用多线程方式

```rust
use std::thread::{self, sleep};
use std::time::Duration;

fn main() {
    println!("Hello before reading file!");
    let handle1 = thread::spawn(|| {
        let file1_contents = read_from_file1();
        println!("{:?}", file1_contents);
    });

    let handle2 = thread::spawn(|| {
        let file2_contents = read_from_file2();
        println!("{:?}", file2_contents);
    });

    handle1.join().unwrap();
    handle2.join().unwrap();
}

fn read_from_file1() -> String {
    sleep(Duration::new(4, 0));
    String::from("Hello, there from file 1")
}

fn read_from_file2() -> String {
    sleep(Duration::new(2, 0));
    String::from("Hi, there from file 2")
}
```

* 使用异步方式运行

```rust
use std::thread::{self, sleep};
use std::time::Duration;

/*
    #[tokio::main] 告诉编译器它使用 tokio 作为 async 运行时
*/

#[tokio::main]
async fn main() {
    println!("Hello before reading file!");

    // 生成异步运行任务，由 tokio 进行管理
    let h1 = tokio::spawn(async {
        /*
            Note：由于 async 函数是惰性的，只有遇到 await 才会执行
            这里必须调用 await 否则函数不会执行
         */
        let _file1_contents = read_from_file1().await;
    });

    let h2 = tokio::spawn(async {
        let _file2_contents = read_from_file2().await;
    });

    let _ = tokio::join!(h1, h2);
}

/*
    async fn 变为可被 tokio 安排的异步运行时任务
    async 函数是惰性的，只有遇到 await 才会执行
*/
async fn read_from_file1() -> String {
    sleep(Duration::new(4, 0));
    println!("{:?}", "Processing file 1");
    String::from("Hello, there from file 1")
}

async fn read_from_file2() -> String {
    sleep(Duration::new(2, 0));
    println!("{:?}", "Processing file 2");
    String::from("Hi, there from file 2")
}
```

> 上边异步方式，与多线程方式类似。但区别是异步执行可能在同一个线程执行，也可能在多个线程执行，这依赖于 `tokio` 的运行时配置
{: .prompt-info }


# 理解 Async 
* 异步编程，就是允许我们在同一个系统线程上同时运行多个任务
* 实现异步编程的诀窍：就是当 CPU 等待外部事件或动作时（例如等待读写文件到磁盘、等待网络数据到达、定时器完成等），其实就是一段代码或一个函数在等待的同时，这个异步运行时（例如 tokio）就会
安排其他可继续执行的任务在 CPU 上执行。而当从磁盘或 I/O 子系统的系统中断到达时，异步运行时会知道识别这件事，并安排原来的任务继续执行。
* 一般来说，I/O 受限的程序（程序执行的速度依赖于 I/O 子系统的速度）比起 CPU 受限的任务（程序执行速度依赖于 CPU 速度）可能更适合异步任务的执行。但也不是绝对的。
* `async`、`.await` 关键字是 Rust 标准库里用于异步编程的内置核心原语集的代表，就是语法糖


# Future
* `Future` 是异步编程的核心
* `Future` 是由异步计算或函数产生的单一最终值，即最终会产生一个值
* Rust 的异步函数都会返回 `Future`，`Future` 基本上就代表着延迟计算


```rust
// 这个写法就是下边写法的语法糖
async fn read_from_file1() -> String {
    sleep(Duration::new(4, 0));
    println!("{:?}", "Processing file 1");
    String::from("Hello, there from file 1")
}

// 等价于下边这样定义

use std::future::Future;

fn read_from_file1() -> impl Future<Output = String> {
    async {
        sleep(Duration::new(4, 0));
        println!("{:?}", "Processing file 1");
        String::from("Hello, there from file 1")
    }
}

```


## `Future` 的定义

* `type Output`：就是 `Future` 完成时的返回的数据类型
* `poll` 方法：对于异步运行时是非常重要的，是被异步运行时调用的，用来检查异步任务是否已经完成，它返回一个枚举
  * `Poll::Pending`：表示还没有完成
  * `Poll::Ready(val)`：表示已经完成，`val 就是 Output 类型的`

```rust
pub trait Future {
    /// The type of value produced on completion.
    #[stable(feature = "futures_api", since = "1.36.0")]
    #[rustc_diagnostic_item = "FutureOutput"]
    type Output; // 就是完成时的返回的数据类型

    /// Attempt to resolve the future to a final value, registering
    /// the current task for wakeup if the value is not yet available.
    ///
    /// # Return value
    ///
    /// This function returns:
    ///
    /// - [`Poll::Pending`] if the future is not ready yet
    /// - [`Poll::Ready(val)`] with the result `val` of this future if it
    ///   finished successfully.
    ///
    /// Once a future has finished, clients should not `poll` it again.
    ///
    /// When a future is not ready yet, `poll` returns `Poll::Pending` and
    /// stores a clone of the [`Waker`] copied from the current [`Context`].
    /// This [`Waker`] is then woken once the future can make progress.
    /// For example, a future waiting for a socket to become
    /// readable would call `.clone()` on the [`Waker`] and store it.
    /// When a signal arrives elsewhere indicating that the socket is readable,
    /// [`Waker::wake`] is called and the socket future's task is awoken.
    /// Once a task has been woken up, it should attempt to `poll` the future
    /// again, which may or may not produce a final value.
    ///
    /// Note that on multiple calls to `poll`, only the [`Waker`] from the
    /// [`Context`] passed to the most recent call should be scheduled to
    /// receive a wakeup.
    ///
    /// # Runtime characteristics
    ///
    /// Futures alone are *inert*; they must be *actively* `poll`ed to make
    /// progress, meaning that each time the current task is woken up, it should
    /// actively re-`poll` pending futures that it still has an interest in.
    ///
    /// The `poll` function is not called repeatedly in a tight loop -- instead,
    /// it should only be called when the future indicates that it is ready to
    /// make progress (by calling `wake()`). If you're familiar with the
    /// `poll(2)` or `select(2)` syscalls on Unix it's worth noting that futures
    /// typically do *not* suffer the same problems of "all wakeups must poll
    /// all events"; they are more like `epoll(4)`.
    ///
    /// An implementation of `poll` should strive to return quickly, and should
    /// not block. Returning quickly prevents unnecessarily clogging up
    /// threads or event loops. If it is known ahead of time that a call to
    /// `poll` may end up taking awhile, the work should be offloaded to a
    /// thread pool (or something similar) to ensure that `poll` can return
    /// quickly.
    ///
    /// # Panics
    ///
    /// Once a future has completed (returned `Ready` from `poll`), calling its
    /// `poll` method again may panic, block forever, or cause other kinds of
    /// problems; the `Future` trait places no requirements on the effects of
    /// such a call. However, as the `poll` method is not marked `unsafe`,
    /// Rust's usual rules apply: calls must never cause undefined behavior
    /// (memory corruption, incorrect use of `unsafe` functions, or the like),
    /// regardless of the future's state.
    ///
    /// [`Poll::Ready(val)`]: Poll::Ready
    /// [`Waker`]: crate::task::Waker
    /// [`Waker::wake`]: crate::task::Waker::wake
    #[lang = "poll"]
    #[stable(feature = "futures_api", since = "1.36.0")]
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

## 谁来调用 `poll` 方法呢？
* 异步执行器：
  * 它来调用 `poll` 方法
  * 是异步运行时的一部分
  * 会管理一个 `Future` 的集合，并通过调用 `Future` 上的 `poll` 方法来驱动它们的完成。所以函数或代码块前面加上 `async` 后，就相当于告诉异步执行器他会返回 `Future`，
  这个 `Future` 需要被驱动直到完成
* 但是异步执行器如何知道 `Future` 已经准备好可以取得进展了呢？


## 理解 Future

* 之所以使用 Pin 是因为 `Future` 会被异步运行时反复的 `poll`，所以把它 Pin 住，固定到内存的某一位置，对这个异步代码的安全性是必要的。
* `poll` 方法会被异步执行器调用（例如 tokio 的执行器），执行器会尝试解析 `Future` 来得到最终的结果（ 也就是 `Future` 的 `Output` 类型）
* 如果 `Future` 的值不可用即 `poll` 方法返回 `Poll::Pending`，那么当前任务就会被注册到 `Waker` 组件，以便当 `Future` 的变得可用的时候 `Waker` 组件就可以通知异步运行时再次调用 `Future` 里的 `poll` 方法


```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::thread::sleep;
use std::time::Duration;

struct ReadFileFuture {}

impl Future for ReadFileFuture {
    type Output = String;

    fn poll(self: std::pin::Pin<&mut Self>, cx: &mut std::task::Context<'_>) -> std::task::Poll<Self::Output> {
        println!("Tokio! Stop polling me");
        Poll::Pending // Note：这里例子因为这里返回 Poll::Pending 所以 main.rs 中永远不会结束
    }
}

async fn read_from_file2() -> String {
    sleep(Duration::new(2, 0));
    println!("{:?}", "Processing file 2");
    String::from("Hi, there from file 2")
}

#[tokio::main]
async fn main() {
    println!("Hello before reading file!");

    // 生成异步任务
    let h1 = tokio::spawn(async {
        let future1 = ReadFileFuture {};
        future1.await;
    });

    let h2 = tokio::spawn(async {
        let file2_contents = read_from_file2().await;
        println!("{:?}", file2_contents);
    });

    let _ = tokio::join!(h1, h2);
}
```

### 它会一直对它进行 `poll` 吗？
* 不会
* `Tokio` （Rust 的异步设计）是使用一个 `Waker` 组件来处理这件事的
* 当被异步执行器 `poll` 过的任务还没有准备好产生值时，这个任务就被注册到一个 `Waker`。`Waker` 会有一个处理程序（handle），它会被存储在任务关联的 Context 对象中。
* `Waker` 上还有一个 `wake()` 方法，可以用来告诉异步执行器关联的任务应该被唤醒了。当 `wake()` 方法被调用了，Tokio 执行器就会被通知是时候再次 `poll` 这个异步任务了，具体方式就是
调用任务上的 `poll` 函数


```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::thread::sleep;
use std::time::Duration;

struct ReadFileFuture {}

impl Future for ReadFileFuture {
    type Output = String;

    fn poll(self: std::pin::Pin<&mut Self>, cx: &mut std::task::Context<'_>) -> std::task::Poll<Self::Output> {
        println!("Tokio! Stop polling me");
        /*
            通知 tokio 运行时，异步任务已经准备好了，
            可以被 poll 了

            这里因为下边返回的还是 Poll::Pending，
            所以这个方法会被不断地执行
        */
        cx.waker().wake_by_ref();
        Poll::Pending
    }
}

async fn read_from_file2() -> String {
    sleep(Duration::new(2, 0));
    println!("{:?}", "Processing file 2");
    String::from("Hi, there from file 2")
}

#[tokio::main]
async fn main() {
    println!("Hello before reading file!");

    // 生成异步任务
    let h1 = tokio::spawn(async {
        let future1 = ReadFileFuture {};
        future1.await;
    });

    let h2 = tokio::spawn(async {
        let file2_contents = read_from_file2().await;
        println!("{:?}", file2_contents);
    });

    let _ = tokio::join!(h1, h2);
}
```


# Tokio 组件的组成

![image](/assets/images/rust/tokio.png)

* `Tokio` 运行时需要理解操作系统（内核）的方法（epoll）来开启 `I/O 操作`（例如读取网络数据，读写文件等）
* `Tokio` 运行时会注册异步的处理程序，以便在事件发生时作为 `I/O 操作`的一部分进行调用。而在 `Tokio` 运行时里，从内核监听这些事件并与 `Tokio` 其他部分通信的组件就是`反应器（reactor）`
* `Tokio` 执行器，他会把一个 `Future` 当其可取得更多进展时，通过调用 `Future` 的 `poll()` 方法来驱动其完成

* 那么 `Future` 使如何告诉执行器他们准备好取得进展了呢 ？
    * 就是 `Future` 调用 `Waker` 组件上的 `wake()` 方法
    * `Waker` 组件就会通知执行器，然后把 `Future` 放回队列，并再次调用 `poll()` 方法，直到 `Future` 完成
    

## Tokio 组件的简化工作流程
1. `Main` 函数在 `Tokio` 运行时上生成任务 1
2. 任务 1 有一个 `Future`，会从一个大文件中读取内容
3. 从文件读取内容请求交到系统内核的文件子系统
4. 与此同时，任务 2 也被 `Tokio` 运行时安排进行处理
5. 当任务 1 的文件操作结束时，文件子系统会触发一个系统中断，他会被翻译成 `Tokio` 响应器可识别的一个事件
6. `Tokio` 响应器会通知任务 1，文件操作的数据已经准备好
7. 任务 1 通知它注册的 Waker 组件，说明它可以产生一个值了
8. Waker 组件通知 Tokio 执行器来调用任务 1 关联的 poll() 方法
9. `Tokio` 执行器安排任务 1 进行处理，并调用 `poll()` 方法
10. 任务 1 产生一个值


```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::thread::sleep;
use std::time::Duration;

struct ReadFileFuture {}

impl Future for ReadFileFuture {
    type Output = String;

    fn poll(self: std::pin::Pin<&mut Self>, cx: &mut std::task::Context<'_>) -> std::task::Poll<Self::Output> {
        println!("Tokio! Stop polling me");
        /*
            通知 tokio 运行时，异步任务已经准备好了，
            可以被 poll 了

            这里因为下边返回的还是 Poll::Pending，
            所以这个方法会被不断地执行
        */
        cx.waker().wake_by_ref();
        // Poll::Pending
        Poll::Ready(String::from("Hello, from file 1!!!")); // 这里返回 Poll::Ready，这个 Future 可结束了
    }
}

async fn read_from_file2() -> String {
    sleep(Duration::new(2, 0));
    // println!("{:?}", "Processing file 2");
    String::from("Hi, there from file 2")
}

#[tokio::main]
async fn main() {
    println!("Hello before reading file!");

    // 生成异步任务
    let h1 = tokio::spawn(async {
        let future1 = ReadFileFuture {};
        let val = future1.await;
        println!("{:?}", val);
    });

    let h2 = tokio::spawn(async {
        let file2_contents = read_from_file2().await;
        println!("{:?}", file2_contents);
    });

    let _ = tokio::join!(h1, h2);
}
```

* 上边 `ReadFileFuture` 返回 `Poll::Ready` 导致能取得一个值，最终结束


# 最后例子
## 创建一个自定义 Future，它是一个定时器，具有以下功能：
* 可设定超时时间
* 当它被异步运行时的执行器 poll 的时候，它会检查
  1. 如果当前时间大于等于超时时间，则返回 Poll::Ready，里面带着一个 String 值
  2. 如果当前时间小于超时时间，则它会睡觉直至超时为止。然后它会触发 Waker 上的 wake() 方法，这就会通知异步运行时的执行器来安排再次执行该任务


```rust
use std::future::{Future, self};
use std::pin::Pin;
use std::task::{Context, Poll};
use std::thread::sleep;
use std::time::{Duration, Instant};

struct AsyncTimer {
    expiration_time: Instant, // 超时时间, Instant 定义于标准库
}

impl Future for AsyncTimer {
    type Output = String;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if Instant::now() >= self.expiration_time {
            println!("Hello, it's time for Future 1");
            Poll::Ready(String::from("Future 1 has completed"))
        } else {
            println!("Hello, it's not yet time for Future 1. Going to sleep");
            let waker = cx.waker().clone();
            let expiration_time = self.expiration_time;
            std::thread::spawn(move || {
                let current_time = Instant::now();
                if current_time < expiration_time {
                    std::thread::sleep(expiration_time - current_time);
                }
                waker.wake();
            });
            Poll::Pending
        }
    }
}

async fn read_from_file2() -> String {
    sleep(Duration::new(2, 0));
    String::from("Future 2 has completed")
}

#[tokio::main]
async fn main() {
    let h1 = tokio::spawn(async {
        let future1 = AsyncTimer {
            expiration_time: Instant::now() + Duration::from_millis(3000),
        };
        println!("{:?}", future1.await);
    });

    let h2 = tokio::spawn(async {
        let file2_contents = read_from_file2().await;
        println!("{:?}", file2_contents);
    });

    let _ = tokio::join!(h1, h2);
}

// Hello, it's not yet time for Future 1. Going to sleep
// "Future 2 has completed"
// Hello, it's time for Future 1
// "Future 1 has completed"
```



