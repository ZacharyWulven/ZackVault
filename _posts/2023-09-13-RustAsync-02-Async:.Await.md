---
layout: post
title: RustAsync-02-Async/.Await
date: 2023-09-13 16:45:30.000000000 +09:00
categories: [Rust]
tags: [Rust]
---


# 3 async/.await

## 什么是 `async/.await`
* `async/.await` 是 Rust 特殊语法，就是当执行发生阻塞的时，它让放弃当前线程的控制权成为可能，这就允许在等待操作完成的时候，允许其他代码取得进展
1. 例如：socket 数据还没有来时候，这时就不会干等着


## 使用 async 的两种方式
1. async fn
2. async blocks

```rust

// 方式一：async fn
// `foo()` returns a type that implements `Future<Output = u8>`.
// `foo().await` will result in a value of type `u8`.
async fn foo() -> u8 { 5 }

fn bar() -> impl Future<Output = u8> {
    // This `async` block results in a type that implements
    // `Future<Output = u8>`.
    // 方式二：async blocks
    async {
        let x: u8 = foo().await;
        x + 5
    }
}
```

* 这两种方式都返回实现了 `Future trait` 的值
* async fn 和 async blocks 和其他 `Future trait` 都是惰性的
1. 即在真正运行之前它们什么都不做

### 使用 `.await`
* 使用 `.await` 就是运行 `Future trait` 最常见的方式
1. 对 `Future trait` 使用 `.await` 后它就会尝试驱动其运行直到完成
* 如果 `Future trait` 被阻塞了：
1. 它就会放弃当前线程的控制权，让这个线程干别的
2. 当可取得更多进展时，执行器会捡起这个 `Future trait` 并在原来那个位置恢复执行，最终由 `.await` 完成解析


## async/.await 解释

```rust
use async_std::io::prelude::*;
use async_std::net;
use async_std::task;

/// 异步函数以 async 开头
/// 虽然返回值是 std::io::Result<String>，但无需调整返回值类型，Rust 自动把它当成相应的 Future 类型
/// 返回的 Future 是包含所需相关信息的：包括参数、本地变量空间
/// 
/// Future 的具体类型是由编译器基于函数体和参数自动生成的
/// 1. 改类型没有名称
/// 2. 它实现了 Future<Output=R>，这个函数 R 就是 Result<String>
/// 
/// 
/// 第一次对 cheapo_request 进行 poll 时：
/// 从函数体顶部开始执行，直到第一个 await（针对 TcpStream::connect 返回 Future），
/// 这个 await 就会对 TcpStream::connect 的 Future 进行 poll
/// 1. 如果没有完成就返回 Pending
/// 2. 只要没有完成， cheapo_request 函数就没法继续
/// 3. main 函数中对 cheapo_request 也无法继续 poll
/// 4. 直到 TcpStream::connect 返回 Ready 后才能继续往后执行
/// 
/// 
/// await 能干什么？：
/// 1. 获得 Future 的所有权，并对其进行 poll
/// 2. 对 Future 进行 poll 时，如果 Future 返回 Ready，其最终值就是 await 表达式，这时就继续执行后续代码，
/// 否则就返回 Pending 给调用者
/// 
/// 
/// Note：
/// 下一次 main 函数中对 cheapo_request 的 Future 进行 poll 时（使用执行器 block_on），
/// 并不是从函数体顶部开始执行，而是从上一次暂停的位置开始，直到它变成 Ready，才会继续在函数体往下走
/// 例如：TcpStream::connect 上一次返回 Pending，那么代码就停在这里了，下次会从这里再开始看能否
/// 取得更多进展
/// 小结：可以理解为 await Pending 时候，相当于 suspended 了，下次 poll 从这里恢复直到 Ready，再往下走
/// 
/// 
/// 随着 cheapo_request 的 Future 不断被 poll，其执行就是从一个 await 到下一个 await，而且只有子 Future
/// 的 await 变成 Ready 之后才能继续往下走。
/// cheapo_request 的 Future 会追踪：
/// 1. 下一次 poll 应该恢复继续的那个点
/// 2. 以及所需的本地状态（变量、参数、临时变量等）
/// 
/// 
/// 这种途中能暂停执行，然后恢复执行的能力是 async 所独有的，由于 await 表达式依赖于“可恢复执行”这个特性，
/// 所以 await 只能用在 async 中
/// 而暂停执行时线程在做什么？它不是在干等着，而是在做其他的工作。
///
/// 
async fn cheapo_request(host: &str, port: u16, path: &str) -> std::io::Result<String> {
    /*
        .await 的是异步的
        .await 会等待，直到 Future 变成 ready，ready 后 await 最终会解析出 Future 的值

        Note：当调用 async 函数时，在其函数体执行前，它就会立即返回，即执行到 port)) 函数就返回了

        这里就是获取 TcpStream::connect 返回 Future 的所有权，并对这个 Future 进行 poll，
        但对 Future 进行 poll，await 不是唯一的方式，其他的执行器也可以

     */
    let mut socket = net::TcpStream::connect((host, port)).await?;
    let request = format!("Get {} HTTP/1.1\r\nHost: {}\r\n\r\n", path, host);

    socket.write_all(request.as_bytes()).await?;
    socket.shutdown(net::Shutdown::Write)?;

    let mut response = String::new();
    socket.read_to_string(&mut response).await?;
    Ok(response)
}

fn main() -> std::io::Result<()> {
    // 在非异步函数中调用异步函数 block_on 是一种方式, 它是一个执行器
    let response = task::block_on(cheapo_request("example.com", 80, "/"))?;
    println!("{}", response);
    Ok(())
}
```

## async/.await 生命周期
* 与传统函数不同，`async fn` 如果它的参数是引用或是其他非 `'static` 的，那么它返回的 `Future` 就会绑定到参数的生命周期上
* 这意味着 `async fn` 返回的 `Future`，在 `.await` 的同时，这个函数的非 `'static` 的参数必须保持有效


```rust
async fn foo(x: &u8) -> u8 { *x }

// foo 函数扩展后就是 foo_expanded
// 参数和返回值的生命周期都是 'a
fn foo_expanded<'a>(x: &'a u8) -> impl Future<Output = u8> + 'a {
    async move { *x }
}
```


### 存储 `Future` 或传递 `Future`
* 通常 `async` 函数在调用后会立即调用 `.await`，这样就不会有什么问题：
1. 例如 `foo(&x).await`

* 如果你想存储 `async` 返回的 `Future`，或将 `Future` 传递给其他任务或线程，就有问题了，一种变通的解决方法是：
1. 思路：把使用引用作为参数的 `async fn` 转为一个 `'static future`
2. 具体做法：在 `async` 块里，将参数和 `async fn` 的调用绑定到一起（目的就是：延长参数的生命周期来匹配 Future）


```rust
fn bad() -> impl Future<Output = u8> {
    let x = 5;
    // borrow_x 是一个异步函数，这里调用后没有调用 await，x 的生命周期不够长，可能活不到真正运行的时候
    borrow_x(&x) // ERROR: `x` does not live long enough
}

// 解决方案：我们可以在外部加一个 async 块，async 块返回的是 Future
// async 块这个 Future 就会记住 x 这个变量，这样就保证 x 与 borrow_x 生命周期是匹配的了
fn good() -> impl Future<Output = u8> {
    async {
        let x = 5;
        borrow_x(&x).await
    }
}
```


## async move
* `async` 块和闭包都支持 `move` 这个关键字
* `async move` 块（后边对应的代码）会获得其引用变量的所有权：
1. pros：允许这个引用比当前所在的作用域活的长
2. cons：放弃了与其他代码共享这些变量的能力

```rust
/// `async` block:
///
/// Multiple different `async` blocks can access the same local variable
/// so long as they're executed within the variable's scope
async fn blocks() {
    let my_string = "foo".to_string();

    let future_one = async {
        // ...
        println!("{my_string}");
    };

    let future_two = async {
        // ...
        println!("{my_string}");
    };

    // Run both futures to completion, printing "foo" twice:
    let ((), ()) = futures::join!(future_one, future_two);
}

/// `async move` block:
///
/// Only one `async move` block can access the same captured variable, since
/// captures are moved into the `Future` generated by the `async move` block.
/// However, this allows the `Future` to outlive the original scope of the
/// variable:
fn move_block() -> impl Future<Output = ()> {
    let my_string = "foo".to_string();
    async move {
        // ...
        println!("{my_string}");
    }
}
```

## 在多线程执行者上进行 `.await`
* 当使用多线程 `Future` 执行者时，`Future` 就可以在线程间移动
1. 所以 `async` 体里面用的变量必须能够在线程间移动
2. 因为任何的 `.await` 都可能导致切换到一个新的线程

* 这意味着使用以下类型是不安全的：
1. `Rc、&RefCell` 和任何其他没有实现 `Send trait` 的类型，包括没有实现 `Sync trait` 的引用
2. `Note：` 调用 `.await` 时，只要这些类型不在作用域内，你就可以使用它们


* 在跨越一个 `.await` 期间，持有传统的、对 Future 无感知的锁，也不是好主意
1. 因为可能导致线程池锁定
2. 为此，可使用 `futures::lock` 里的 `Mutex` 而不是 `std::sync` 里的 `Mutex`


> 
{: .prompt-info }

