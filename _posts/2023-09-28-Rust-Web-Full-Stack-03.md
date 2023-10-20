---
layout: post
title: Rust Web 全栈开发教程-03
date: 2023-09-28 16:45:30.000000000 +09:00
categories: [Rust, Rust Web]
tags: [Rust, Rust Web]
---


# 6 Web Service 中的错误处理

* 在 `Actix` 中有统一的错误处理机制 
1. 即将数据库错误、序列化错误等错误转为自定义的错误类型，再转为有意义的 `HTTP` 响应信息发给客户端

![image](/assets/images/rust/web_server/actix_err.png)


## `Actix-Web` 的错误处理

* 编程语言常用的两种错误处理方式：
1. 异常（比如 JAVA）
2. 返回值（比如 Go、Rust）


* Rust 希望开发者能显示的处理错误，因此可能出错的函数返回 `Result` 这个枚举类型

```rust
enum Result<T,E> {
  Ok(T), // 如果这个函数没有错误就返回 Ok，正确要返回的值就是 T
  Err(E),
}
```

* 例子：

```rust
use core::num;
use std::num::ParseIntError;

fn main() {
    // let result = square("25");
    let result = square("Rt");

    println!("{:?}", result);
}

// 这就是一个可能出错的函数
fn square(val: &str) -> Result<i32, ParseIntError> {
    match val.parse::<i32>() {
        // 能成功解析成一个整数，就返回这个整数的平方
        Ok(num) => Ok(num.pow(2)),
        Err(e) => Err(e),
    }
}
```


## `?` 运算符
* 在某个函数中使用 `?` 运算符，该运算符就会尝试从 `Result` 中获取值：
1. 如果没有获取成功，它就会接收 Error，中止函数执行，并把错误传播到调用该函数的函数


```rust
fn square(val: &str) -> Result<i32, ParseIntError> {
    /*
        如果成功获取 i32，则 num 的值就是其值，
        否则此函数直接返回了，返回的就是 Err(e)
     */
    let num = val.parse::<i32>()?;
    Ok(num ^ 2)
}
```


> 把 `?` 作用于 `Result`。如果结果是 `Ok`，`Ok` 中的值就是表达式的结果，然后继续执行。如果结果是 `Err`，则 `Err` 就是整个函数的返回值，就类似使用了 `return`
{: .prompt-info }


## 自定义错误类型
* 项目中产生的错误肯定不是同一类型的错误，是多种类型的，这里解决思路就是创建一个自定义错误类型，例如枚举或结构体

```rust
#[derive(Debug)]
pub enum MyError {
  ParseError,
  IOError,
}
```

### `Actix-Web` 也使用同样思路，把错误转化为 `HTTP Response`
* `Actix-Web` 定义了一个通用的错误类型 `actix_web::error::Error`（是个 `struct`）
1. 它实现了标准库的 `std::error::Error` 这个 trait


> 任何实现了标准库 `Error trait` 的类型，都可以通过 `?` 运算符转化为 `Actix 的 Error` 类型，即转化为 `actix_web::error::Error` 类型，`Actix 的 Error` 类型会自动转化为 `HTTP Response` 返回给客户端
{: .prompt-info }


* `Actix-Web` 中定义了 `ResponseError trait`：即任何实现了该 `trait` 的错误均可以转化为 `HTTP Response` 消息
* `Actix-Web` 框架中对一些常见的错误有一些内置的实现，这些错误会自动转为 `HTTP Response`，例如
1. Rust 标准 `I/O` 错误
2. `Serde` 错误（序列化、反序列化错误）
3. `Web` 错误：例如 `ProtocolError、Utf8Error、ParseError` 等等

* 而其他错误类型（内置实现不包含的类型）：就需要自定义实现错误到 `HTTP Response` 的转换
