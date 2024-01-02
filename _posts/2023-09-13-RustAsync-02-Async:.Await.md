---
layout: post
title: RustAsync-02-Async/.Await
date: 2023-09-13 16:45:30.000000000 +09:00
categories: [Rust, Rust Async]
tags: [Rust, Rust Async]
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
/// 1. 该类型没有名称
/// 2. 它实现了 Future<Output=R>，这个函数 R 就是 Result<String>
/// 
/// 
/// 第一次对 cheapo_request 进行 poll 时：
/// 从函数体顶部开始执行，直到第一个 await（针对 TcpStream::connect 返回 Future 进行 await），
/// 这个 await 就会对 TcpStream::connect 的 Future 进行 poll
/// 1. 如果没有完成就返回 Pending
/// 2. 只要没有完成， cheapo_request 函数就没法继续
/// 3. main 函数中对 cheapo_request 也无法继续 poll
/// 4. 直到 TcpStream::connect 返回 Ready 后才能继续往后执行
/// 
/// 
/// await 能干什么？：
/// 1. 获得 Future 的所有权，并对其进行 poll
/// 2. 对 Future 进行 poll 时，如果 Future 返回 Ready，其最终值就是 await 表达式的值，这时就继续执行后续代码，
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
/// 这种途中能暂停执行，然后恢复执行的能力是 async 函数所独有的，由于 await 表达式依赖于“可恢复执行”这个特性，
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



# 4 Pinning

## 什么是 `Pin`
* `Pin` 是与 `Unpin` 标记一起工作的 
* `Pin` 会保证实现了 `!Unpin` 的对象永远不会被移动


## 为什么需要 `Pin`

```rust
let fut_one = /* ... */;
let fut_two = /* ... */;
async move {
    fut_one.await;
    fut_two.await;
}


/// 上边 async 块会创建一个实现了 `Future trait` 的匿名类型，而这个类型会提供一个与下边类似的 poll 方法

// The `Future` type generated by our `async { ... }` block
// 上边 async 块生成的类型
struct AsyncFuture {
    fut_one: FutOne,
    fut_two: FutTwo,
    state: State,
}

// List of states our `async` block can be in
enum State {
    AwaitingFutOne,
    AwaitingFutTwo,
    Done,
}

impl Future for AsyncFuture {
    type Output = ();
    
    // 串行完成 FutureOne 和 FutureTwo，都完成后返回 Ready
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match self.state {
                State::AwaitingFutOne => match self.fut_one.poll(..) {
                    Poll::Ready(()) => self.state = State::AwaitingFutTwo,
                    Poll::Pending => return Poll::Pending,
                }
                State::AwaitingFutTwo => match self.fut_two.poll(..) {
                    Poll::Ready(()) => self.state = State::Done,
                    Poll::Pending => return Poll::Pending,
                }
                State::Done => return Poll::Ready(()),
            }
        }
    }
}
```

* 上边 `async 块`没有使用引用，而如果在 `async 块` 中使用引用会是什么情况呢？

```rust
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);  // 返回 Future
    read_into_buf_fut.await;
    println!("{:?}", x);
}


/// 上边块编译后的结构
struct ReadIntoBuf<'a> {
    buf: &'a mut [u8], // points to `x` below，就是对 x 的引用
}

// 上边 async 块生成的类型
struct AsyncFuture {
    x: [u8; 128], 
    read_into_buf_fut: ReadIntoBuf<'what_lifetime?>,
}

```

* 上边 如果 x 移动了，那么 `ReadIntoBuf` 中还是指向 `x` 原来的地址（数据已经走了，但指针还指向原来的地址，这样指针就失效了）为了解决这个问题就需要
把 `Future Pin` 到内存中的特定位置防止该问题发生
1. 可以在 `async` 块里安全的创建到值的引用


* 先看一下 swap 的例子

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();


    println!("1: a: {}, b: {}", test1.a(), test1.b()); // 1: a: test1, b: test1
    std::mem::swap(&mut test1, &mut test2); 

    // 交换它们地址的值，移动数据
    // 交换之后地址没有变，但里边内容变了
    // test1 地址没有变，但内部变成了 test2 内容
    println!("1: a: {}, b: {}", test1.a(), test1.b()); // 1: a: test2, b: test1
    println!("2: a: {}, b: {}", test2.a(), test2.b()); // 2: a: test1, b: test2
}
#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
        }
    }

    // We need an `init` method to actually set our self-reference
    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }

    fn a(&self) -> &str {
        &self.a
    }

    fn b(&self) -> &String {
        assert!(!self.b.is_null(), "Test::b called without Test::init being called first");
        unsafe { &*(self.b) }
    }
}

```

上边交换变量的内存展示

![image](/assets/images/rust/pin_swap.png)


## Pin 实践
* Pin 类型会包裹指针类型，保证指指向的值不被移动
1. 例如：`Pin<&mut T>`、`Pin<&T>`、`Pin<Box<T>>`，即使 `T` 是 `!Unpin` 标记的，也能保证 `T` 不被移动

## Unpin trait（可以被移动）
* 大多数类型如果被移动了，不会造成问题，它们实现了 `Unpin trait`
* 指向 `Unpin` 类型的指针，可以自由的放入或从 `Pin` 中取出
1. 例如：`u8` 是 `Unpin` 的，`Pin<&mut u8>` 和普通的 `&mut u8` 是一样的，都是可以移动的


> 如果类型拥有 `!Unpin`（固定） 标记，那么在 `Pin` 之后它们就无法移动了
{: .prompt-info }


### `Pin` 到 `Stack` 上

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}


impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned, // This makes our type `!Unpin`, 使类型变为 `!Unpin`
        }
    }

    fn init(self: Pin<&mut Self>) {
        let self_ptr: *const String = &self.a;
        // 如果对象实现了 `!Unpin`，则把这个对象 Pin 到 Stack 上就是 unsafe 的
        // 这时对象的数据就固定到栈上了，不会移动
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
    }

    fn a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }

    fn b(self: Pin<&Self>) -> &String {
        assert!(!self.b.is_null(), "Test::b called without Test::init being called first");
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    // test1 is safe to move before we initialize it
    let mut test1 = Test::new("test1");
    // Notice how we shadow `test1` to prevent it from being accessed again
    // 这里需要把 test1 shadowing 一下，以之前的 test1 再次被访问
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    // 这里交换就会报错，因为传入 swap 的参数应该是 Unpin 的，而此时 test1，tetst2 是 Pin 的
    std::mem::swap(test1.get_mut(), &mut test2.get_mut()); // Error, required by this bound in `Pin::<&'a mut T>::get_mut

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
```

> 如果对象实现了 `!Unpin`，则把这个对象 `Pin` 到 `Stack` 上就是 `unsafe` 的，这时对象的数据就固定到栈上了，不会移动
{: .prompt-info }


### `Pin` 到 `Heap` 上

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> { // 使用 Box 说明 Pin 到 Heap 上
        let t = Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned,
        };
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }

    fn b(self: Pin<&Self>) -> &String {
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    let test1 = Test::new("test1");
    let test2 = Test::new("test2");

    println!("a: {}, b: {}",test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}",test2.as_ref().a(), test2.as_ref().b());
}
```

## 小结
* 如果 `T` 实现了 `Unpin（默认情况）`，则 `Pin<'a, T>` 与 `&'a mut T` 完全等价 
1. `Pin` 不 `Pin` 它没有卵用
2. `Unpin` 意味着该类型如果被 `Pin` 了，那么它也是可以移动的，所以 `Pin` 对这种类型不起作用
* 如果 `T` 实现了 `!Unpin`（就是 Pin 的），那么把 `&mut T` 变成 `Pin` 的 `T`，就需要 `unsafe` 操作
* 大部分标准库类型都实现了 `Unpin`，Rust 里大部分正常类型也是。但由 `async/await` 生成的 `Future` 是个例外
* 可以使用特性标记为类型添加一个 `!Unpin` 绑定（最新版），或者通过添加 `std::marker:PhantomPinned` 到类型上（稳定版）

* 可以将数据 `Pin` 到 `Stack` 或 `Heap` 上
* 把 `!Unpin` 对象 `Pin` 到 `Stack` 上需要 `unsafe` 操作
* 把 `!Unpin` 对象 `Pin` 到 `Heap` 上是`不`需要 `unsafe` 操作
1. 它有个快捷操作：即使用 `Box::pin`

> 重要：针对已经 `Pin` 的数据，如果 `T` 是 `!Unpin` 的，则需要保证它从被 `Pin` 后，内存一直有效且不会调整其用途，直到 `drop` 被调用
{: .prompt-info }

