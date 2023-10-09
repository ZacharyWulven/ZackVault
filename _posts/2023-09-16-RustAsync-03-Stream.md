---
layout: post
title: RustAsync-03-Stream
date: 2023-09-16 16:45:30.000000000 +09:00
categories: [Rust]
tags: [Rust]
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
      这时整个函数也叫完成，它并不会等待另一个任务的完成
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





