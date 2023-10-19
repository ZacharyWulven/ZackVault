---
layout: post
title: RustGuide-17-Practice
date: 2023-05-16 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 17 Rust 项目练习

## 项目需求
* 在 socket 上监听 TCP 的链接
* 解析少量的 HTTP 请求
* 创建一个合适的 HTTP 响应
* 使用线程池改进服务器的吞吐量

> 本项目并不是最佳实践，而是将前边章节的内容复习一下
{: .prompt-info }

* 本项目会处理 TCP 里的原始字节，并与 HTTP 响应打交道


## 单线程 Web 服务器

* hello.html

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="utf-8">
    <title>Hello!</title>
</head>

<body>
    <h1>Hello!</h1>
    <p>Hi from Rust</p>
</body>

</html>
```

* 404.html

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="utf-8">
    <title>Hello!</title>
</head>

<body>
    <h1>Oops!</h1>
    <p>Sorry, I don't know what you're asking for.</p>
</body>

</html>
```

* 单线程 Web 服务器代码 

```rust
// 监听 TCP 需要用到
use std::net::TcpListener;
use std::io::prelude::*;
use std::net::TcpStream;
use std::fs;
use std::time::Duration;
use std::thread;

fn main() {
    /*
        bind 函数会监听传入的地址，这里是 127.0.0.1:7878，返回值是 Result<T, E>

     */
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    /*
        incoming 产生一个 TCP Stream 的流序列的迭代器，
        而单个流就表示客户端和服务器之间打开了一个连接
        而 for 循环就会依次处理每一个连接，并生成一系列的流进行处理
     */
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        handle_connection(stream);
    }
}
/*
    参数为可变的
    因为 TcpStream 实例内部记录了返回给我们的数据
    它可能会返回多于我们请求的数据，并将这些数据保存下来
    以备下次请求使用，因为 TcpStream 的内部状态可能会改变，所以标记为 mut

 */
fn handle_connection(mut stream: TcpStream) {
    // 缓存 512 字节
    let mut buffer = [0; 512];
    /*
        read 方法会从 TcpStream 读取数据
        并把放到 buffer 中

     */
    stream.read(&mut buffer).unwrap();

    // 请求(CRLF 是换行回车符号)
    // Method Request-URI HTTP-Version CRLF
    // headers CRLF
    // message-body

    // 响应
    // HTTP-Version Status-Code Reason-Phrase CRLF
    // headers CRLF
    // message-body

    //let response = "HTTP/1.1 200 OK\r\n\r\n";

    /*
        b"" 是字节字符串的语法
        将 get 里的文本转化为字节字符串，后边就可以进行比较了

     */
    let get = b"GET / HTTP/1.1\r\n";

    let sleep = b"GET /sleep HTTP/1.1\r\n";

    // 看看 buffer 是否以 GET 开头
    // 这里返回一个元组
    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else if buffer.starts_with(sleep) {
        // 如果以 http://127.0.0.1/sleep 开头就休眠 5s
        thread::sleep(Duration::from_secs(5));
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();
    let response = format!("{}{}", status_line, contents);

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();


    // 将 response 写回去
    //stream.write(response.as_bytes()).unwrap();
    // flush 会等待并阻止程序的运行, 直到所有的字节都被写入到连接中
    //stream.flush().unwrap();
    
    // 然后把 buffer 中的字节转为字符串打印出来
    //println!("Request: {}", String::from_utf8_lossy(&buffer[..]));
}
```

## 改为多线程 Web 服务器
* 使用线程池实现
* 线程池就是一组预分配出来的线程，它们被用于等待并随时处理可能的任务，当程序接收到一个新的任务时，
它会给线程池里的线程分配任务，然后这个线程就会处理这个任务
* 线程池里其他的线程在第一个线程处理任务的同时还可以接收其他的任务，而当线程处理完它的任务后，我们就把它放回线程池，
此时它就变成空闲的状态了，就可以处理新的任务了
* 所以线程池允许你并发的处理连接，从而就增加了服务器的吞吐量


* 为每个连接都创建一个线程

```rust
// 监听 TCP 需要用到
use std::net::TcpListener;
use std::io::prelude::*;
use std::net::TcpStream;
use std::fs;
use std::time::Duration;
use std::thread;

fn main() {
    /*
        bind 函数会监听传入的地址，这里是 127.0.0.1:7878，返回值是 Result<T, E>

     */
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    /*
        incoming 产生一个 TCP Stream 的流序列的迭代器，
        而单个流就表示客户端和服务器之间打开了一个连接
        而 for 循环就会依次处理每一个连接，并生成一系列的流进行处理
     */
    for stream in listener.incoming() {
        let stream = stream.unwrap();

        // 
        /*
            为每一个连接都创建一个新的线程
            但这样来一个连接就创建一个线程，容易被 DDoS 攻击
         */
        thread::spawn(|| {
            handle_connection(stream);
        });
    }
}
```

* lib.rs

```rust
use std::thread;
use std::sync::{Arc, mpsc::{self, Receiver}, Mutex};

type Job = Box<dyn FnBox + Send + 'static>;

// JoinHandle 来源参考 thread 的 spawn 函数
pub struct ThreadPool {
    // 这里与 spawn 函数不同，这里不需要返回值，所以写 ()
    //threads: Vec<thread::JoinHandle<()>>,
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

impl ThreadPool {
    /// Create a new ThreadPool
    /// The size is the number of threads in the pool.
    /// 
    /// # Panics
    /// 
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);
        let (sender, receiver) = mpsc::channel();
        // 使用 Arc 解决所有权共享问题
        let receiver = Arc::new(Mutex::new(receiver));
        let mut workers = Vec::with_capacity(size);
        for id in 0..size {
            /*
                这里我们系统线程创建后处于等待状态，然后有代码传给他们时再执行
                所以我们使用一个叫 Worker 的数据结构
                Worker 是线程池里常用的术语
                然后让 Worker 来管理上述行为
             */
            workers.push(Worker::new(id, Arc::clone(&receiver)));

        }
        // 线程池持有通道的发送端
        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F) where F: FnOnce() + Send + 'static, {
        let job = Box::new(f);
        self.sender.send(job).unwrap();
    }
}


// 使得 self 可以在 Box 上调用
trait FnBox {
    fn call_box(self: Box<Self>);
}
// 所有实现 FnOnce 的类型都给他实现 FnBox 上调用

impl<F: FnOnce()> FnBox for F {
    fn call_box(self: Box<F>) {
        (*self)();
    }
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        // 传入 id 创建一个线程
        let thread = thread::spawn(move || loop {
            // 这块的问题是：慢的请求还是会阻塞其他线程
            while let Ok(job) = receiver.lock().unwrap().recv() {
                println!("Worker {} got a job: executing.", id);

                // (*job)(); 无法直接调用，换成 job.call_box()
                job.call_box();
            }
        });
        Worker { id, thread }
    }
}

```

* 查看文档

```shell
$ cargo doc --open
```

## 停机和清理
* 实现 `Drop Trait` 后的全部项目代码

### lib.rs

```rust
use std::thread;
use std::sync::{Arc, mpsc::{self, Receiver}, Mutex};

type Job = Box<dyn FnBox + Send + 'static>;


enum Message {
    NewJob(Job),
    Terminate,
}

// JoinHandle 来源参考 thread 的 spawn 函数
pub struct ThreadPool {
    // 这里与 spawn 函数不同，这里不需要返回值，所以写 ()
    //threads: Vec<thread::JoinHandle<()>>,
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

impl ThreadPool {
    /// Create a new ThreadPool
    /// The size is the number of threads in the pool.
    /// 
    /// # Panics
    /// 
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);
        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));
        let mut workers = Vec::with_capacity(size);
        for id in 0..size {
            /*
                这里我们系统线程创建后处于等待状态，然后有代码传给他们时再执行
                所以我们使用一个叫 Worker 的数据结构
                Worker 是线程池里常用的术语
                然后让 Worker 来管理上述行为
             */
            workers.push(Worker::new(id, Arc::clone(&receiver)));

        }
        // 线程池持有通道的发送端
        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F) where F: FnOnce() + Send + 'static, {
        let job = Box::new(f);
        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        println!("Sending terminate message to all works");

        for _ in &mut self.workers {
            self.sender.send(Message::Terminate).unwrap();
        }

        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);
            
            // worker.thread.join() 这里不能直接这样
            // 因为 join 要获取所有权，所以将 Worker 的 thread 变为 Option<> 类型解决
            if let Some(thread) = worker.thread.take() {
                // 这里依然不会释放 因为 join 还在等待线程，并且获取接收的内容
                // 需要通过其他信号解决
                thread.join().unwrap();
            }
        }
    }
}


// 使得 self 可以在 Box 上调用
trait FnBox {
    fn call_box(self: Box<Self>);
}
// 所有实现 FnOnce 的类型都给他实现 FnBox 上调用

impl<F: FnOnce()> FnBox for F {
    fn call_box(self: Box<F>) {
        (*self)();
    }
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
        // 传入 id 创建一个线程
        // let thread = thread::spawn(move || {
        //     // 这块的问题是：慢的请求还是会阻塞其他线程
        //     while let Ok(job) = receiver.lock().unwrap().recv() {
        //         println!("Worker {} got a job: executing.", id);

        //         // (*job)(); 无法直接调用，换成 job.call_box()
        //         job.call_box();
        //     }
        // });
        
        // 从上边 while let 换成 loop
        let thread = thread::spawn(move || loop {
            // 这块的问题是：慢的请求还是会阻塞其他线程
            let message = receiver.lock().unwrap().recv().unwrap();
            match message {
                Message::NewJob(job) => {
                    println!("Worker {} got a job: executing.", id);
                    job.call_box();
                },
                Message::Terminate => {
                    break;
                },
            }
        });


        Worker { id, thread: Some(thread) }
    }
}
```

### main.rs

```rust
// 监听 TCP 需要用到
use std::net::TcpListener;
use std::io::prelude::*;
use std::net::TcpStream;
use std::fs;
use std::time::Duration;
use std::thread;
use hello_web_20::ThreadPool;

fn main() {
    /*
        bind 函数会监听传入的地址，这里是 127.0.0.1:7878，返回值是 Result<T, E>

     */
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    let pool = ThreadPool::new(4);


    /*
        incoming 产生一个 TCP Stream 的流序列的迭代器，
        而单个流就表示客户端和服务器之间打开了一个连接
        而 for 循环就会依次处理每一个连接，并生成一系列的流进行处理

        take(2) 只能处理 2 次请求，第三次就停止了
     */
    
    for stream in listener.incoming().take(2) {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });

        /*
            为每一个连接都创建一个新的线程
            但这样来一个连接就创建一个线程，容易被 DDoS 攻击
         */
        // thread::spawn(|| {
        //     handle_connection(stream);
        // });
    }

    println!("Shutting down!");
}
/*
    参数为可变的
    因为 TcpStream 实例内部记录了返回给我们的数据
    它可能会返回多于我们请求的数据，并将这些数据保存下来
    以备下次请求使用，因为 TcpStream 的内部状态可能会改变，所以标记为 mut

 */
fn handle_connection(mut stream: TcpStream) {
    // 缓存 512 字节
    let mut buffer = [0; 512];
    /*
        read 方法会从 TcpStream 读取数据
        并把放到 buffer 中

     */
    stream.read(&mut buffer).unwrap();

    // 请求(CRLF 是换行回车符号)
    // Method Request-URI HTTP-Version CRLF
    // headers CRLF
    // message-body

    // 响应
    // HTTP-Version Status-Code Reason-Phrase CRLF
    // headers CRLF
    // message-body

    //let response = "HTTP/1.1 200 OK\r\n\r\n";

    /*
        b"" 是字节字符串的语法
        将 get 里的文本转化为字节字符串，后边就可以进行比较了

     */
    let get = b"GET / HTTP/1.1\r\n";

    let sleep = b"GET /sleep HTTP/1.1\r\n";

    // 看看 buffer 是否以 GET 开头
    // 这里返回一个元组
    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else if buffer.starts_with(sleep) {
        // 如果以 http://127.0.0.1/sleep 开头就休眠 5s
        thread::sleep(Duration::from_secs(5));
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();
    let response = format!("{}{}", status_line, contents);

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();


    // 将 response 写回去
    //stream.write(response.as_bytes()).unwrap();
    // flush 会等待并阻止程序的运行, 直到所有的字节都被写入到连接中
    //stream.flush().unwrap();
    
    // 然后把 buffer 中的字节转为字符串打印出来
    //println!("Request: {}", String::from_utf8_lossy(&buffer[..]));
}
```
