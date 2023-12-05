---
layout: post
title: RustGuide-13-Concurrent
date: 2023-04-03 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 16 无畏并发
## 并发
* Concurrent：指程序的不同部分之间独立执行
* Parallel：指程序的不同部分同时运行


* 在 Rust 里号称是无畏并发：允许你编写没有细微（诡异） bug 的代码，并且在不引入新 bug 情况下易于重构
* 本课程中的 “并发” 是泛指 Concurrent 和 Parallel，即包括并发和并行


## 使用线程同时运行代码 

### 进程和线程
* 在大部分 OS（操作系统）里，代码运行在进程（process）中，OS 同时管理多个进程
* 在你的程序里，各个独立部分可以同时运行，运行这些独立部分的就是线程（thread）
* 多线程运行
1. 提升性能表现
2. 增加复杂度，无法保障各个线程的执行顺序

### 多线程可导致的问题
* 竞争状态，线程以不一致的顺序访问数据或资源
* 死锁，两个线程彼此等待对方使用完所持有的资源，线程无法继续
* 只在某些情况下发生 Bug，很难可靠的复制和修复

### 实现线程的方式
* 通过调用 OS 的 API 来创建线程：1:1 的模型（一个 OS 的线程对应一个语言的线程），需要较小的运行时
* 语言自己可以实现线程（绿色线程）：M:N 的模型（M 个绿色线程，N 个系统线程），需要更大的运行时
* Rust 标准库仅提供了 1:1 模型的线程，保持尽量小的运行时

### 通过 spawn 创建新线程
* 通过 `thread::spawn` 函数可以创建新线程
1. 参数：一个闭包（在新线程里运行的代码）


```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            // 让当前线程 sleep 
            thread::sleep(Duration::from_millis(1));
        }
    }); 
    for i in 1..5 {
        println!("hi number {} from the main thread!", i);            
        thread::sleep(Duration::from_millis(1));
    }
    // 主线程结束，上边的异步线程也结束了
}
```

* 上边代码，主线程结束，异步线程也结束了，这样不好，如何能保证异步线程任务完成呢？可使用 JoinHandle

### 通过 `JoinHandle` 来等待所有线程的完成
* `thread::spawn` 函数的返回值类型是 `JoinHandle`
* `JoinHandle` 是持有值的所有权的
1. 调用 `join` 方法，就可以等待对应其他线程的完成

* `join` 方法：调用 `handle` 的 `join` 方法会阻止当前运行线程的执行，直到 `handle` 所表示的这个些线程终结


> 类似 `swift semaphore 的 wait`
{: .prompt-info }


```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            // 让当前线程 sleep 
            thread::sleep(Duration::from_millis(1));
        }
    }); 
    for i in 1..5 {
        println!("hi number {} from the main thread!", i);           
        thread::sleep(Duration::from_millis(1));
    }
    // 主线程结束，上边的异步线程也结束了, 可以使用 joinHandle 解决

    /*
        join(）会阻塞当前线程的执行，直到 handle 对应的线程结束
        类似 swift semaphore 的 wait
     */
    handle.join().unwrap();
    println!("thread done");
}
```

### 使用 move 闭包
* move 闭包通常和 `thread::spawn` 函数一起使用，它允许你使用其他线程的数据
* 创建线程时，把值的所有权从一个线程转移到另一个线程
* 使用 `move` 关键字

```rust
use std::thread;

fn moved() {
    let v = vec![1, 2, 3];
    /*
        未加 move 时报错，因为 v 的在线程内的生命周期可能长于外部
        强制异步线程闭包获得 v 的所有权可以加上 move 关键字
     */
    let handle = thread::spawn(move || {
       println!("Here's a vector: {:?}", v); 
    });

    handle.join().unwrap();
}
```

## 使用消息传递来跨线程传递数据

### 消息传递
* 一种很流行并且能保证安全并发的技术就是：消息传递
1. 在这种机制里线程（或 Actor）通过彼此发送消息（数据）来进行通信

> Go 语言的名言：不要用共享内存来通信，要用通信来共享内存
{: .prompt-info }

* Rust 提供了基于消息传递的并发方式： Channel（标准库提供）

### Channel（管道）
* 一个 Channel 就是一个管道
* 一个 Channel 包含：发送端、接收端
* 调用发送端的方法，发送数据
* 接收端会检查和接收到达的数据
* 如果发送端或接收端中任意一端被丢弃了，那么 Channel 就`关闭`了

### 创建 Channel
* 使用 `mpsc::channel` 函数来创建 Channel
1. mpsc 表示 multiple producer，single consumer（多个生成者，一个消费者，即多个发送端，一个接收端）
2. 返回一个 tuple，里边的元素分别是发送端、接收端


```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // tx 是 sender，rx 是 receiver
    let (tx, rx) = mpsc::channel();

    // 创建一个线程，并使用 move 让线程有 tx 的所有权
    // 新的线程必须拥有发送端的所有权才能往通道里发消息
    thread::spawn(move || { // 加 move，获得 tx 的所有权
        let val = String::from("hi");
        // 发送消息，如果接收到已经被丢弃，这时会产生一个错误，这里用 unwrap 简单处理
        tx.send(val).unwrap();
    });

    /*
        recv 会阻塞当前线程，直到有消息被传入
        有消息就返回 Ok，否则返回 Err
     */
    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

### 发送端 tx 的 send 方法
* 参数：就是想要发送的数据
* 返回：`Result<T,E>`，如果有问题（例如接收端已经被丢弃），就返回一个错误

### 接收端的方法
* recv 方法：阻塞当前线程的执行，直到 Channel 中有值被送过来
1. 一旦收到值就会返回 `Result<T,E>`
2. 如果这个 Channel 的所有发送端都关闭了，就会收到一个错误

* try_recv 方法：它不会阻塞当前线程，会立即返回 `Result<T,E>`
1. 这时如果有数据到达就返回 Ok，里边包裹着数据
2. 否则返回一个错误
* 通常会使用循环调用来检查 `try_recv` 的结果，一旦有消息就进行处理，如果没来也可做其他操作

### Channel 和所有权转移
* 所有权在消息传递中非常重要，能帮你编写安全、并发的代码


```rust
fn main() {
    // tx 是 sender，rx 是 receiver
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || { // 加 move，获得 tx 的所有权
        let val = String::from("hi");
         
        tx.send(val).unwrap(); // val 在这里发生了 move
        
        // Error! 这里报错因为 val 已经 move 了
        println!("val is {}", val);
    });

    /*
        recv 会阻塞当前线程，直到有消息被传入
        有消息就返回 Ok，否则返回 Err
     */
    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

### 发送多个值，看到接收者在等待

```rust
fn wait() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_millis(200));
        }
    });

    /*
        把接收端当成迭代器，这样就不需要调用 recv 函数了
        每收到一个值就将其打印
        当 Channel 关闭时就会退出这个循环
        这也是一种常用的用法
     */
    for received in rx {
        println!("Got: {}", received);
    }
}
```

### 通过 clone 创建多个发送者

```rust
fn clone_tx() {
    let (tx, rx) = mpsc::channel();

    // 这里通过 clone 创建一个新的发送端 tx1
    let tx1 = mpsc::Sender::clone(&tx);

    // 用 tx1 进行 send
    thread::spawn(move || {
        let vals = vec![
            String::from("1: hi"),
            String::from("1: from"),
            String::from("1: the"),
            String::from("1: thread"),
        ];

        for val in vals {
            tx1.send(val).unwrap();
            thread::sleep(Duration::from_millis(200));
        }
    });

    // 用 tx 进行 send
    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_millis(200));
        }
    });
    
    for received in rx {
        println!("Got: {}", received);
    }
}
```


> 以上讲的都是用通信来共享内存，也就是 Go 语言推崇的方式
{: .prompt-info }



## 共享状态的并发
下面讲一下 go 语言不推荐的用共享来进行并发（通信）
 
* Rust 支持通过共享状态来实现并发
* Channel 类似`单所有权`：`即一旦将值的所有权转移至 Channel，在原来的地方就无法再继续使用它了`
* 而共享内存并发类似`多所有权`，`即多个线程可以同时访问同一块内存`

### 使用 `Mutex` 来保证每次只允许一个线程来访问数据
* `Mutex` 是 `mutual exclusion（互斥锁）`的简写
* 在同一时刻，`Mutex` 只允许一个线程来访问某些数据

* 想要访问数据
1. 线程必须先获取互斥锁，在 Rust 里就是调用 `lock` 方法获得一个 `lock`
2. `lock` 数据结构是 `Mutex` 的一部分，它能跟踪谁对数据拥有独占访问权
3. `Mutex` 通常被描述为：通过锁定系统来保护它所持有的数据

### `Mutex` 的两条规则
1. 在使用数据之前，必须尝试获取锁（lock）
2. 使用完 `Mutex` 所保护的数据，必须对数据进行解锁，以便其他线程可以获取锁

### `Mutex<T>` 的 API
* 通过 `Mutex::new(数据)` 来创建 `Mutex<T>`，`Mutex<T>` 也是一个智能指针
* 访问数据前，通过 `lock 方法`来获取锁
1. lock 方法会阻塞当前线程
2. lock 方法也可能会失败
3. 返回的是 MutexGuard（也是一个智能指针，实现了 Deref 和 Drop）

```rust
use std::sync::Mutex;

fn main() {
    /*
        创建 Mutex，这个 5 就是要保护的数据
     */
    let m = Mutex::new(5);

    {   // 内部作用域
        /*
            通过 lock 获取锁，这个方法会阻塞当前线程
            直到获取到锁，它有可能发生错误，这里用 unwrap 进行处理
            返回 MutexGuard，它实现了 Deref 所以可以指向它内部
            的数据，从而我们可以获得其内部数据的一个引用
         */
        let mut num = m.lock().unwrap();

        // 这个可以修改其值是因为 MutexGuard，它实现了 Deref trait
        *num = 6;
        
    } // MutexGuard，它实现了 Drop trait，所以这里会释放，也就解锁了
    println!("m = {:?}", m);
}
```

### 多线程共享 `Mutex<T>`

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);

    let mut handles = vec![];

    for _ in 0..10 {
        // Error 这里 move 报错，因为第一次开线程已经获取了 counter 的所有权
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle); 
    } 

    for handle in handles {
        handle.join().unwrap();
    }
    println!("Result: {}", *counter.lock().unwrap());
}
```


> 上边代码由于前一次循环开线程已经获取了 counter 的所有权，所以报错，下边我们通过`多线程的多重所有权` 解决
{: .prompt-info }


### 多线程的多重所有权
* 是否可以使用 `Rc<T>` 解决上例 counter 所以权的问题？
1. 答案是不行的，因为 `Rc<T>` 不可以在线程间安全的被 `Send`，因为 `Rc<T>` 没有实现 `Send trait`
2. 因为：只有实现了 `Send trait` 的类型，才可以在线程间安全的传递
3. `Rc<T>` 只适合单线程的场景


### 使用 `Arc<T>` 来进行原子引用计数
* `Arc<T>` 和 `Rc<T>` 类似，但是它可以用于并发的场景
1. `A` 即 `atomic`，原子性
2. 所以 `Arc<T>` 就是原子性的 `Rc<T>`

* 为什么所有的基础类型都不是原子的？为什么标准库类型不默认使用 `Arc<T>` 呢？
1. 原因是需要性能作为代价
* `Arc<T>` 和 `Rc<T>` 的 API 是相同的


```rust
use std::sync::{ Mutex, Arc };
use std::thread;

fn main() {
    // 通过 Arc 原子性解决多线程共享问题
    let counter: Arc<Mutex<i32>> = Arc::new(Mutex::new(0));

    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle); 
    } 

    for handle in handles {
        handle.join().unwrap();
    }
    println!("Result: {}", *counter.lock().unwrap());
}
```

### `RefCell<T>/Rc<T>` VS `Mutex<T>/Arc<T>`
* `Mutex<T>` 提供了内部可变性，和 Cell 家族一样
* 我们可以使用 `RefCell<T>` 来改变 `Rc<T>` 里的内容
* 我们可以使用 `Mutex<T>` 来改变 `Arc<T>` 里的内容

> 使用 `Rc<T>` 可能造成循环引用，而使用 `Mutex<T>` 可能导致死锁
{: .prompt-info }


## 通过 Send 和 Sync trait 来扩展并发

### Send 和 Sync trait
* Rust 语言的并发特性较少，目前将的并发特性都来自标准库（而不是语言本身）
* 无需局限于标准库的并发，可以自己实现并发
* 但在 Rust 语言中有两个并发概念，相关的是两个 `trait`，他们是`标签 trait`，因为没有定义任何方法
1. `std::marker::Sync`
2. `std::marker::Send` 

### Send trait
* 允许线程间转移所有权

> 实现了 `Send trait` 的类型就可以在线程间转移所有权。Rust 中几乎所有类型都实现了 `Send trait`，但 `Rc<T>` 没有实现 `Send trait`，它只被用于单线程的场景。
{: .prompt-info }

* 任何完全由 `Send trait` 类型组成的类型也被标记为 `Send trait`
* 除了原始指针之外，几乎所有的基础类型都实现了 `Send trait`

### Sync
* 允许从多线程进行访问，或叫允许多线程同时访问

> 实现 `Sync` 的类型可以安全的被多个线程引用。即如果 `T` 实现了 `Sync`，也就是说 `&T` 实现了 `Send`，也就是说 `&T` 可以被安全的送往另一个线程。
{: .prompt-info }

* 基础类型都实现了 `Sync`
* 完全由 `Sync` 类型组成的类型也相当于实现了 `Sync`
1. 但 `Rc<T>` 不是 `Sync` 的
2. `RefCell<T>` 和 `Cell<T>` 家族也不是 `Sync` 的
3. `Mutex<T>` 是 `Sync` 的

> 记住：手动来实现 `Send` 和 `Sync` 是不安全的，因为手动实现这些 `trait` 需要使用 Rust 的不安全的特殊的代码，你需要非常谨慎的设计才能保证线程间的安全性要求
{: .prompt-info }
