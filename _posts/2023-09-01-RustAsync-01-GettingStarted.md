---
layout: post
title: RustAsync-01-GettingStarted
date: 2023-09-01 16:45:30.000000000 +09:00
categories: [Rust, Rust Async]
tags: [Rust, Rust Async]
---


[配套官方教程](https://rust-lang.github.io/async-book/)


# 1 GettingStarted

## 为什么使用 `async`
* `async` 是一种并发（concurrent）的编程模型
1. 允许你在少数系统线程上运行大量并发任务
2. 通过 `async/await` 语法，看起来和同步编程差不多

## 其他并发的模型

### OS 线程
* 无需改变编程模型，线程间同步困难，性能开销大
* 使用线程池可以降低一些开销成本，但依然无法支持大量的 IO 绑定工作

### Event-driven 编程（事件驱动的编程模型）
* 与回调函数一起使用，有可能是比较高效的
* 但其控制流是非线性的，数据流和错误传播是难以追踪的

### Coroutines
* 类似线程，无需改变编程模型
* 类似 `async`，可以支持大量的任务
* 但是是抽象掉了大量底层的细节（而底层的细节对系统编程，自定义运行时实现很重要）
* 面向系统编程的语言就不太合适了

### Actor 模型
* 即将所有并发计算划分为 `actor`，其消息通信比较容易出错
* 此模型实现起来可以比较高效，但很多实际问题没有解决（例如控制流、逻辑重试）


## Rust 中的 async
* `Future` 是惰性的，`lazy` 的
1. 只有在调用 `poll` 函数时才能取得进展，而被丢弃的 `Future` 就无法取得进展了

* `Async` 是零成本的
1. 使用 `Async` 可以无需堆内存的分配（heap allocation）和动态调度（dynamic dispatch），函数是静态分配，并且没有创建额外的线程
2. 对性能比较好，而且允许在受限环境使用 `Async`

* Rust 不提供内置的运行时，运行时都是由社区提供的

* 单线程、多线程都支持，但优缺点不同


## Rust 中的 async 和线程（thread）
* OS 线程
1. 适用于少量的任务，有内存和 CPU 开销，而且线程生成和线程间的切换非常昂贵
2. 线程池可以减低一些成本
3. 允许重用同步代码，代码无需大改，无需特定的编程模型
4. 有些系统支持修改线程优先级


* Async
1. 显著降低内存和 CPU 开销
2. 同等条件下，支持比线程多几个数量级的任务（少数线程支撑大量任务）
3. 可执行文件大（需要生成状态机，每个可执行文件捆绑一个异步运行时）


* 使用 OS 线程下载网页，然而下载网页是一项小任务；为如此少量的工作创建一个线程是相当浪费的。对于更大的应用程序，它很容易成为瓶颈。

```rust
fn get_two_sites() {
    // Spawn two threads to do work.
    let thread_one = thread::spawn(|| download("https://www.foo.com"));
    let thread_two = thread::spawn(|| download("https://www.bar.com"));

    // Wait for both threads to complete.
    thread_one.join().expect("thread one panicked");
    thread_two.join().expect("thread two panicked");
}
```

* 在异步Rust中，我们可以同时运行这些任务，而无需额外的线程，在这里，没有创建额外的线程。此外，所有函数调用都是静态调度的，并且没有堆分配！

```rust
async fn get_two_sites_async() {
    // Create two different "futures" which, when run to completion,
    // will asynchronously download the webpages.
    let future_one = download_async("https://www.foo.com");
    let future_two = download_async("https://www.bar.com");

    // Run both futures to completion at the same time.
    join!(future_one, future_two);
}
```


> Note：`async` 并不是比线程好，只是与线程不同而已
{: .prompt-info }



## Rust Async 目前状态
* 部分稳定，部分仍在变化
* 特点
1. 针对典型并发任务，性能出色
2. 与高级语言特性频繁交互（例如生命周期，pinning）
3. 同步和异步代码间、不同运行时的异步代码间存在兼容性约束（`因为官方没有运行时，运行时是社区提供的`，所以可能导致有兼容问题）
4. 由于不断进化，未来维护代码负担可能会重一些

## 语言和库的支持
* 虽然 Rust 本身就支持 `async` 编程，但很多应用依赖于社区的库：
1. 标准库提供了最基本的特性、类型和功能，例如 `Future trait`
2. `async/await` 语法直接被 Rust 编译器支持
3. `Futures crate` 提供了许多实用类型、宏和函数。它们可以用于任何异步应用程序。
4. 异步代码、IO 和任务生成的执行由 `async runtimes` 提供，例如 `Tokio` 和 `async-std`（这俩目前比较流行）。大多数 `async` 应用程序和一些 `async crate` 都依赖于特定的运行时。


> Rust 不允许你在 `trait` 中声明 `async` 函数。
{: .prompt-info }


## 编译和调试

### 编译错误
* 由于 `async Rust` 通常依赖于更复杂的语言功能，例如生命周期和 `Pinning`，因此可能会更频繁的遇到这些类型的错误。

### 运行时错误
* 每当运行时遇到异步函数，编译器会在后台生成一个状态机，`Stack traces` 里有其明细，以及运行时调用的函数。因此解释起来更复杂。

### 新的失效模式
* 可能出现一些新的故障，他们可以通过编译，甚至可以通过单元测试


## 兼容性考虑

### `async`（异步） 和同步代码不能总是自由组合
* 例如不能直接在同步函数中来调用异步函数

### `async`（异步）代码间也不总是能自由组合
* 例如一些 `crate` 依赖与特定的 `async` 运行时，就可能有兼容性约束的问题


> So，在做项目中应该尽早的确定使用哪个 `async` 运行时，尽早确定下来
{: .prompt-info }

## 性能特征
* `async` 的性能依赖于运行时的实现（通常是比较出色的）

## `async/await` 入门

### `async`
* `async` 就是异步的意思，它把一段代码转化为一个实现了 `Future trait` 的状态机
* 虽然在同步方法中调用了阻塞函数会阻塞整个线程，但阻塞的 `Future` 将放弃对线程的控制，从而允许其他 `Future` 来运行

### `async fn`

* 异步函数语法
* `async fn` 返回的类型是 `Future`，`Future` 需要由一个执行者来运行

```rust
#![allow(unused)]
fn main() {
    println!("Hello, world!");

    async fn do_something() {

    }
}

```

* `futures::executor::block_on`（这是一个执行者）
1. `block_on` 会阻塞当前线程，直到提供的 `Future` 运行完成
2. 其他执行者提供更复杂的行为，例如：将多个 `Future` 安排到同一个线程中

```rust
use futures::executor::block_on;

async fn hello_world() {
    println!("hello, world!");
}

fn main() {
    let future = hello_world(); // 未打印任何东西

    // 将 future 传给执行者 block_on
    block_on(future);  // future 才运行，并打印出 "hello, world!"
}
```

### `await`
* 在 `async fn` 中，可以使用 `.await` 来等待另一个实现 `Future trait` 的类型的完成
* `await` 与 `block_on` 不同，`.await` 不会阻塞当前线程，而是异步的等待这个 `Future trait` 的完成
1. 即如果该 `Future` 目前无法取得进展，就允许其他任务运行

```rust
use futures::executor::block_on;

struct Song {

}

async fn learn_song() -> Song {
    Song {}
}

async fn sing_song(song: Song) {}

async fn dance() {}

async fn learn_and_sing() {
    let song = learn_song().await; // .await 后执行下边的 sing_song
    sing_song(song).await;
}

async fn async_main() {
    let f1 = learn_and_sing(); // 返回 future
    let f2 = dance();          // 返回 future

    // join! 这个宏类似 await，它可以等待多个 future
    // 如果阻塞在了 f1 这个线程，那么 f2 线程就会接管当前线程，反之亦然
    // 如果 f1 和 f2 都阻塞了，那么就说这个函数阻塞了，那就需要交给其执行者，这里是 block_on
    futures::join!(f1, f2);
}

fn main() {
    // 下面 3 行是串行的，没有达到更好的性能要求
    // let song = block_on(learn_song());
    // block_on(sing_song(song));
    // block_on(dance());
    
    // 改为使用 await
    block_on(async_main());
}

````

# 2.1 幕后原理：执行 Future 和任务

### Future Trait
* `Future Trait` 是 `Rust Async` 编程的核心
* `Future Trait` 是一种异步计算，它可以产生一个值
* 实现了 `Future Trait` 的类型表示`目前可能还不可用的值，未来可能可用`

* 下面是 `Future Trait` 简单版本

```rust
enum Poll<T> {
    Ready(T),
    Pending,
}

trait SimpleFuture {
    // Output 即未来要返回的值的类型
    type Output; 


    // poll 类似轮询，调用 poll 方法就会驱动 SimpleFuture 向着完成继续前进;
    // 参数 wake 是函数指针;
    // 返回值 Poll：
    // 1 如果是 Ready 就说明这个 Future 结束了，并且 Ready 的值的类型就是 Output 类型
    // 2 如果是 Pending 就说明这个 Future 还没有结束，未来至少还有 poll 一次，看看到时候进展
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output>;
}
```

### 另一个角度解释 `Future Trait`

* `Future Trait` 可以表示：
1. 下一次网络数据包的到来
2. 下一次鼠标的移动
3. 或者仅仅是经过一段时间的时间点，例如：过了 5 秒后的时间点

* `Future Trait` 代表着一种你可以`检验其是否完成的操作`
* `Future Trait` 就是来检验的，你可以通过 `poll` 函数来检验（取得进展）
1. `poll` 函数会驱动 `Future Trait` 尽可能接近完成
2. 如果 `Future Trait` 返回的是 `poll::Ready(result)`（`result` 就是最终的结果（那个值））这个变体就说明 `Future Trait` 完成了
3. 如果 `Future Trait` 返回的是 `poll::Pending` 这个变体就说明这个 `Future Trait` 还没有完成，未来当 `Future Trait` 准备好取得
更多进展时调就用一个 `waker 的 wake() 函数`


> 小结：针对 `Future` 你唯一能做的就是调用 `poll` 来敲它，直到出现一个`值`或一个 `Error`
{: .prompt-info }

### wake() 函数
* 当一个 `Future` 将要取得更多进展时候，就会调用 `wake() 函数`
* 当 `wake() 函数` 被调用时：
1. 执行器（executer）就会取得 `Future` 再次调用 `poll` 函数，以便 `Future` 能取得更多进展，最大进展就是 `Future` 的完成，把值取出来
* 如果没有 `wake() 函数`，执行器就不知道特定的 `Future` 何时能取得进展，就只能不断地调用 `poll`，这样是低效率的
* 通过 `wake() 函数`，执行器就能确切的知道哪些 `Future` 已准备好进行 `poll` 的调用


#### 例子一

```rust
pub struct SocketRead<'a> {
    socket: &'a Socket,
}

impl SimpleFuture for SocketRead<'_> {
    type Output = Vec<u8>;

    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if self.socket.has_data_to_read() {
            // socket 有数据，读取数据到 buffer 并返回
            Poll::Ready(self.socket.read_buf())
        } else {
            /* 
                socket 还没有数据时，在未来有数据时，或这个 future 准备取得更多进展时候，
                带告诉它，就是通过 wake 这个函数告诉，当未来有数据时候，wake 就会被调用
                wake 被调用后就会再次调用 poll 这个方法来检查数据是否真的有了
                最后返回 Pending 变体
            */
            self.socket.set_readable_callback(wake);
            Poll::Pending
        }
    }
}
```

#### 例子二
* 本例组合多个异步操作，而无需中间分配
* 可以通过无分配的状态机来实现多个 `Future` 同时运行或串联运行

```rust
/*
    Join 中有两个 Future
    它的作用就是并发的让这两个 Future 来完成
*/
pub struct Join<FutureA, FutureB> {
    a: Option<FutureA>,
    b: Option<FutureB>,
}

impl<FutureA, FutureB> SimpleFuture for Join<FutureA,FutureB> 
where FutureA: SimpleFuture<Output = ()>, 
    FutureB:SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {

        // a 如果有值就取出，同时将 a 设置为 None
        if let Some(a) = &mut self.a {
            if let Poll::Ready(()) = a.poll(wake) {
                self.a.take();
            }
        }

        // b 如果有值就取出，同时将 b 设置为 None
        if let Some(b) = &mut self.b {
            if let Poll::Ready(()) = b.poll(wake) {
                self.b.take();
            }
        }

        // 如果 a 和 b 都是 None 就说明这两个 Future 都完成了
        // 否则就返回 Pending，Pending 就表示这里至少还有一个 Future 未完成 
        if self.a.is_none() && self.b.is_none() {
            Poll::Ready(())
        } else {
            Poll::Pending
        }
    }
    
}
```

#### 例子三
* 多个连续的 `Future` 可以一个接一个的运行

```rust
pub struct AndThenFut<FutureA, FutureB> {
    first: Option<FutureA>,
    second: FutureB,
}

impl<FutureA, FutureB> SimpleFuture for AndThenFut<FutureA, FutureB>
where FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()> {
    
    type Output = ();

    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        // 如果 first 有值，就调用 first 的 poll 方法
        // 如果 返回 Ready 就说明第一个完成了，就把 first 设置为 None，然后再调用 second poll 进行返回
        // 如果 first 没有完成就返回 Pending
        if let Some(first) = &mut self.first {
            match first.poll(wake) {
                Poll::Ready(()) => self.first.take(),
                Poll::Pending => return Poll::Pending,
            };
        }
        self.second.poll(wake)
    }
}
```

### 真正的 `Future Trait VS SimpleFuture`

* `SimpleFuture` 参数 `self` 类型不再是 `&mut self`，而是 `pin<&mut self>`
1. `pin<&mut self>` 允许我们创建不可移动的 `future`
2. 不可移动的对象可以在它们的字段间存储指针
3. 如果要启用 `async/await` 语法的话，则 `pin` 是必须的

* `SimpleFuture` 的参数 `wake`，从 `fn()` 变为了 `&mut Context<'_>`
1. 在 `SimpleFuture` 里我们通过调用函数指针 `wake: fn()` 来告诉 `Future` 的执行器相关的 `Future` 应该被 `poll` 了
2. 而由于 `wake: fn()` 是一个函数指针，它不能一些上下文数据，即不能存储任何关于哪个 `Future` 调用了 `wake: fn()` 的数据
3. `Context` 类型提供了访问 `Waker` 类型的值的方式，这些值可以被用来 `wake up（唤醒）` 特定任务。例如：实际项目中 `Web Server` 可能有上千个不同的连接，
它们的 `wake up` 应该是单独管理的


```rust
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>,) -> Poll<Self::Output>;
}
```

# 2.2 使用 `Waker` 唤醒任务

## `Waker` 类型的作用
* `Future` 在第一次 `poll` 时通常无法完成任务，所以 `Future` 需要保证在准备好取得更多进展后（比如说 `socket` 有数据了），`Future` 可以再次被 `poll`
* 而且每次 `Future` 被 `poll` 时，它都是作为一个任务的一部分
* 任务（即 Task）就是被提交给执行者的顶层的 `Future`

* `Waker` 提供了 `wake()` 方法，这个方法可以被用来告诉执行者，相关的任务应该被唤醒了
* 当 `wake()` 方法被调用时候，执行者就知道 `Waker` 所关联的任务已经准备好取得更多进展了，这时 `Future` 就应该被再次 `poll` 一下
* 另外 `Waker` 还实现了 `clone()`，可以用于复制和存储

### 例子
* 使用 `Waker` 实现一个简单的计时器 `Future` 

```rust
use std::{ 
    future::Future, 
    pin:: Pin, 
    sync::{Arc, Mutex},
    task::{Context, Poll, Waker},
    thread,
    time::Duration,
};

/*
    TimerFuture 让线程来传达定时器的时间已经到了，这个 Future 可以完成了
*/
pub struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>,
}

// 在 Future 和等待的线程之间共享的状态
struct SharedState {
    /// 睡眠时间是否已经都过完了
    completed: bool,

    /// Future 运行在任务上（总是属于一个任务），而那个任务有一个 Waker 就是这个 waker
    /// waker 就是 `TimerFuture` 所运行于的任务的 Waker（）
    /// 在设置 `completed = true` 之后，线程可以使用 waker 来告诉 `TimerFuture` 的任务可以唤醒了，并取得进展
    waker: Option<Waker>,
}

impl Future for TimerFuture {
    type Output = ();

    // 查看 shared state，看下 timer 是否已经结束
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(())
        } else {
            /*
                设置 waker 以便当 timer 结束时线程可以唤醒当前任务，
                保证 Future 可以再次被 poll，并看到 `completed = true` 

                每次 Future 被 poll 时,都把 waker clone 一下，
                这是因为 TimerFuture 可在执行者的任务间移动，这会导致过期的 waker
                指向错误的任务，从而阻止了 TimerFuture 正确的唤醒

                Note：可以使用 `Waker::will_wake` 函数来检查这一点，为了简单这里我们就省略了
             */
            shared_state.waker = Some(cx.waker().clone());
            Poll::Pending            
        }
    }

}

impl TimerFuture {
    // 创建一个新的 TimerFuture，它将在提供的时限过后完成
    pub fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        // 生成新线程
        // 这里 clone 一下，因为其是 Arc 类型所以只是增加了引用计数
        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            // 线程休眠对应的时间
            thread::sleep(duration);
            let mut shared_state = thread_shared_state.lock().unwrap();

            // 休眠时间已到了然后发出信号：计时器已停止并唤醒 Future 被 poll 的最后一个任务（如果存在的话）
            shared_state.completed = true;
            if let Some(waker) = shared_state.waker.take() {
                // wake 方法被调用后，相关的任务（或 Future）就可以被唤醒，然后这个 Future 就会被再 poll 一下，
                // 看是否可以取得更多进展或完成
                waker.wake()
            }
        });

        TimerFuture { shared_state }
    }
}
```


* main.rs

```rust
use std::time::Duration;
use futures::executor::block_on;
use timer_future_02::TimerFuture;

fn main() {
    let future = TimerFuture::new(Duration::new(3, 0));
    block_on(future);
}
```

# 2.3 构建一个执行器（Executor）

## `Future` 的执行者（Executor）

> `Future` 是惰性的，除非驱动它们来完成，否则就什么也不做
{: .prompt-info }


* 其中一种取得方式就是在 `async` 中使用 `.await`，但者只是把问题推到了上一层面，也就是谁来执行顶层 `async` 函数返回的 `Future` 呢？
1. 所以说我们需要一个 `Future` 的执行者
* `Future` 执行者会获取一系列顶层的 `Future`，并通过在 `Future` 可以有进展的时候调用 `poll` 方法，来驱动这些 `Future` 运行直到完成
* 通常执行者会先对 `Future` 进行一次 `poll`，以便开始，而当 `Future` 通过调用 `wake()` 表示它们已经准备好取得进展时，它们就会被放回到一个队列里，
然后 `poll` 方法再次被调用，重复此操作直到 `Future` 完成

## 例子
* 构建简单的执行者，可以运行大量的顶层 `Future` 来并发的将它们完成
* 需要使用 `futures crate 的 ArcWake trait`，它提供了简单的方式用来组建 `Waker`


* main.rs (lib.rs 见上边 TimerFuture 实现)

```rust
use futures::executor::block_on;
use futures::{
    future::{BoxFuture, FutureExt},
    task::{waker_ref, ArcWake},
};
use std::{
    future::Future,
    sync::mpsc::{sync_channel, Receiver, SyncSender},
    sync::{Arc, Mutex},
    task::Context,
    thread,
    time::Duration,
};

// lib.rs 中的 TimerFuture
use timer_future_02::TimerFuture;


/// 任务执行者，它会从 channel 收到任务并运行它们
struct Executor {
    ready_queue: Receiver<Arc<Task>>,
}

/// `Spawner` 产生新的 Futures 任务，并把任务放到 channel 中
#[derive(Clone)]
struct Spawner {
    task_sender: SyncSender<Arc<Task>>,
}

/// 一个任务可以重新安排自己，以便被一个 `Executor` 来进行 poll
struct Task {
    /// In-progress future that should be pushed to completion.
    ///
    /// The `Mutex` is not necessary for correctness, since we only have
    /// one thread executing tasks at once. However, Rust isn't smart
    /// enough to know that `future` is only mutated from one thread,
    /// so we need to use the `Mutex` to prove thread-safety. A production
    /// executor would not need this, and could use `UnsafeCell` instead.
    /// 
    /// 正在进行中的 Future，它应该被推向完成
    /// `Mutex` 对应正确性来说不是必要的，因为我们同时只有一个线程在执行任务
    /// 尽管如此，Rust 不够聪明，它无法指定 future 只由一个线程来修改
    /// 所以我们需要使用 `Mutex` 来保证线程的安全
    /// 生成版本的执行者不需要这个，可以使用 `UnsafeCell` 来代替
    future: Mutex<Option<BoxFuture<'static, ()>>>,

    /// Handle to place the task itself back onto the task queue.
    /// 能把任务本身放回任务队列的处理器
    task_sender: SyncSender<Arc<Task>>,
}

/// 最开始会调用这个函数，返回一个执行者、一个任务生成器、和一个管道，
/// 执行者就是在管道的接收端
/// 任务生成器就是在管道的发送端，往通道里发送任务
fn new_executor_and_spawner() -> (Executor, Spawner) {
    // 在 channel 中允许同时排队的最大任务数
    // 这只是让 sync_channel 激活，并不会出现在真实的执行者中
    const MAX_QUEUED_TASKS: usize = 10_000;
    let (task_sender, ready_queue) = sync_channel(MAX_QUEUED_TASKS);
    println!("[{:?}] 生成 Executor 和 Spawner（含发送端、接收端）...", thread::current().id());
    (Executor { ready_queue }, Spawner { task_sender })
}

impl Spawner {
    fn spawn(&self, future: impl Future<Output = ()> + 'static + Send) {
        let future = future.boxed();
        /// 将 future 包装成 任务
        let task = Arc::new(Task {
            future: Mutex::new(Some(future)),
            task_sender: self.task_sender.clone(),
        });
        println!("[{:?}] 将 Future 组成 Task，放入 Channel ...", thread::current().id());
        /// 发送到通道
        self.task_sender.send(task).expect("too many tasks queued");
    }
}

impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        // Implement `wake` by sending this task back onto the task channel
        // so that it will be polled again by the executor.
        // 通过将该任务发送回任务 Channel 来实现 `wake`
        // 以便他将会被执行者再次进行 poll
        println!("[{:?}] call wake_by_ref ...", thread::current().id());

        let cloned = arc_self.clone();
        arc_self
            .task_sender
            .send(cloned)
            .expect("too many tasks queued");
    }
}

impl Executor {
    fn run(&self) {
        println!("[{:?}] Executor running...", thread::current().id());
        // 从通道不断地接收任务，直到没有任务再继续往下走
        while let Ok(task) = self.ready_queue.recv() {
            println!("[{:?}] 接收到任务...", thread::current().id());
            // Take the future, and if it has not yet completed (is still Some),
            // poll it in an attempt to complete it.
            // 获得 future，如果它还没有完成（仍然是 Some），
            // 对它进行 poll，以尝试完成它
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                
                println!("[{:?}] 从任务中取得 Future...", thread::current().id());
                // Create a `LocalWaker` from the task itself
                // 从任务本身创建一个 `LocalWaker`
                
                let waker = waker_ref(&task); 
                println!("[{:?}] 获得 waker by ref ...", thread::current().id());

                let context = &mut Context::from_waker(&waker);
                println!("[{:?}] 获得 context 准备进行 poll() ...", thread::current().id());
                // `BoxFuture<T>` is a type alias for
                // `Pin<Box<dyn Future<Output = T> + Send + 'static>>`.
                // We can get a `Pin<&mut dyn Future + Send + 'static>`

                // from it by calling the `Pin::as_mut` method.
                // `BoxFuture<T>` 是 `Pin<Box<dyn Future<Output = T> + Send + 'static>>` 的类型别名
                // 我们可以通过调用 `Pin::as_mut` 从它获得 `Pin<&mut dyn Future + Send + 'static>`
                if future.as_mut().poll(context).is_pending() {
                    // We're not done processing the future, so put it
                    // back in its task to be run again in the future.
                    // 还没有对 Future 完成处理，所以把它放回它的任务
                    // 以便在未来再次运行
                    *future_slot = Some(future);
                    println!("[{:?}] Poll::Pending ====", thread::current().id());
                } else {
                    // 当返回 ready 后，这个通道就不会再有任务了，因为 main 函数中 spawner drop 操作
                    // 这时，这个 while 循环就停止了
                    println!("[{:?}] Poll::Ready....", thread::current().id());
                }
            }
        }
        println!("[{:?}] Executor run 结束", thread::current().id());
    }
}

fn main() {
    let (executor, spawner) = new_executor_and_spawner();

    // Spawn a task to print before and after waiting on a timer.
    // 生成一个任务，让其等待一个 timer 前后进行打印
    spawner.spawn(async {
        println!("[{:?}] howdy!", thread::current().id());
        // 等待 timer future 在 2s 后完成
        TimerFuture::new(Duration::new(2, 0)).await;
        println!("[{:?}] spawner async done!", thread::current().id());
    });

    // 丢弃生成器以便我们的执行者知道它已经完成了
    drop(spawner);
    println!("[{:?}] Drop Spawner!", thread::current().id());


    // Run the executor until the task queue is empty.
    // This will print "howdy!", pause, and then print "done!".
    // 运行执行者知道任务队列尾空为止
    // 这会打印 “howdy!”, 暂停，然后打印 “done!”.
    executor.run();
}
```

* 大致流程

![image](/assets/images/rust/async_e_01.png)

![image](/assets/images/rust/async_e_02.png)

![image](/assets/images/rust/async_e_03.png)


# 2.4 执行者和系统 IO
* 通过 `SocketRead` 例子讲述，这个 `future` 会读取 `socket` 上可用的数据，如果没有数据，它就屈服于执行者（即它听执行者的）
1. 具体怎么听呢：就是请求当 `socket` 再次可读时，唤醒这个 `Future` 的任务

* 而本例中我们不知道 `Socket` 类型是如何实现的，尤其不知道 `set_readable_callback` 函数如何实现，那么如果在 `socket` 再次可读时安全调用 `wake()` 呢？
1. 一种简单粗暴办法是使用一个线程不断检查 `socket` 是否可读（低效的）

* 而实际中，这个问题是通过与 `IO` 感知的系统阻塞原语（primitive）的集成来解决的
1. 例如 `Linux 的 epoll`，`FreeBSD 和 MacOS 的 kqueue`，`Window 的 IOCP 和 Fuchsia 上的 ports`
2. 所有这些都是通过 Rust 跨平台的 `mio crate` 来暴露的
3. 这些原语（primitive）都允许线程阻塞多个异步 `IO` 事件，并在其中一个事件完成后返回


* `Future` 执行者可以使用这些原语来提供异步 `IO` 对象，
1. 如 `socket`，就可以当特定 `IO` 事件发生时通过配置回调来运行。
2. 具体可参考本章开头的官方文档中的 `2.4 节`

```rust
pub struct SocketRead<'a> {
    socket: &'a Socket,
}

impl SimpleFuture for SocketRead<'_> {
    type Output = Vec<u8>;

    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if self.socket.has_data_to_read() {
            // The socket has data -- read it into a buffer and return it.
            Poll::Ready(self.socket.read_buf())
        } else {
            // The socket does not yet have data.
            //
            // Arrange for `wake` to be called once data is available.
            // When data becomes available, `wake` will be called, and the
            // user of this `Future` will know to call `poll` again and
            // receive data.
            self.socket.set_readable_callback(wake);
            Poll::Pending
        }
    }
}
```
