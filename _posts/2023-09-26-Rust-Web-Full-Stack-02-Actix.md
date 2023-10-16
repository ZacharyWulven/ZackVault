---
layout: post
title: Rust Web 全栈开发教程-Actix 和 REST API
date: 2023-09-26 16:45:30.000000000 +09:00
categories: [Rust]
tags: [Rust]
---


# 2 Actix 框架尝鲜入门

## 2.1 添加 crate 依赖
* `actix-web`：即 `Actix` web 框架，版本是 `v3`
* `actix-rt`：即 `Actix` 异步的运行时，`actix-web v3 依赖 actix-rt 的版本是 1.1.1`


## 2.2 操作
* Step 1：创建 `workspace`，编辑 `workspace 的 Cargo.toml`

```rust
[workspace]
members = ["webservice"]
```

* Step 2：创建 `webservice` 
```shell
$ cargo new webservice
```

* Step 3：打开 `webservice` 中的 `Cargo.toml` 添加依赖

```rust
[package]
name = "webservice"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "3"
actix-rt = "1.1.1"

# 指定二进制名称
[[bin]]
name = "server1"

# [[bin]] 类似一个数组，可指定多个区域，这里还可以有个 server2
#[[bin]]
#name = "server2"
```

* Step 4：在 `webservice` 的 `src` 目录中添加 `bin` 目录，在 `bin` 目录中添加 `server1.rs` 文件，
1. `server1.rs` 中包含 `server1 二进制的 main 函数`

```
目录结构
webservice
 |-src
  |-bin
   |-server1.rs
```


* server1.rs

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Responder};
use std::io;

// 配置 route
pub fn general_routes(cfg: &mut web::ServiceConfig) {
    /*
        设置路由 path 为 /health
        web::get() 表示使用 http get，
        to 中函数 health_check_handler 就是对应的 Handler
     */
    cfg.route("/health", web::get().to(health_check_handler));
}

// 配置 Handler

pub async fn health_check_handler() -> impl Responder {
    /*
        返回 Ok Response，并返回 json 为 Actix Web Service is running
        要求返回值实现 Responder trait
     */
    HttpResponse::Ok().json("Actix Web Service is running!")
}
 
// 实例化 HTTP server 并运行
#[actix_rt::main] // 用到 actix_rt 运行时
async fn main() -> io::Result<()> {
    // 构建 app，配置 route 路由，传入 general_routes 函数
    let app = move || App::new().configure(general_routes);

    // 运行 HTTP server
    // new 函数初始化一个 HttpServer，然后绑定到 3000 地址并运行
    HttpServer::new(app).bind("127.0.0.1:3000")?.run().await
}
```

* Step 5： 运行程序

在 `workspace` 根目录运行 

```shell
$ cargo run -p webservice --bin server1
```

* 或在 `webservice` 目录中

```shell
$ cargo run --bin server1
```

* 然后在浏览器中访问 `http://localhost:3000/health` 即可


## 2.3 基本组件

* `Actix HTTP Server` 它实现了 HTTP 协议，它负责应对请求的，它会开启多个线程来应对请求

![image](/assets/images/rust/web_server/actix.png)


## 2.4 `Actix` 的并发

### `Actix` 支持两类并发：
1. `异步 I/O`：给定的 `OS 原生线程` 在等待 `I/O` 时可以执行其他任务（例如监听网络连接）
2. `多线程并行`：默认情况下启动 `OS 原生线程` 的数量与系统逻辑 `CPU` 的数量是相同的



# 3 构建 REST API

## 3.1 添加 crate 依赖
* `serde`：即序列化和反序列化的库，版本是 `v1`
* `chrono`：用于使用时间相关的字段，版本是 `v0.4`


## 3.2 构建内容

![image](/assets/images/rust/web_server/rest_api.png)

1. 能添加课程，`POST:/courses`
2. 能获取课程，`GET:/courses/teacher_id/course_id`
3. 能获取老师教的所有课程，`GET:/courses/teacher_id`
4. 存储使用内存

* 相关文件
1. bin/teacher-service.rs
2. models.rs
3. state.rs：应用程序状态
4. routers.rs
5. handlers.rs


## 3.3  

* Step 1：修改 `webservice` 的 `toml`，添加 `teacher-service`，并添加 `default-run` 配置项

```rust
# 这是 webservice 的 toml
 
[package]
name = "webservice"
version = "0.1.0"
edition = "2021"

# 当运行 webservice 时，如果不指名二进制文件，那么首先运行 teacher-service
default-run = "teacher-service"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
actix-web = "3"
actix-rt = "1.1.1"

# 指定二进制名称
[[bin]]
name = "server1"

# [[bin]] 类似一个数组，可指定多个区域，这里还可以有个 teacher-service
#[[bin]]
#name = "teacher-service"
```
* Step 2：在 `bin` 目录创建 `teacher-service.rs`，在 `src` 目录下创建 `models.rs、state.rs、routers.rs、handlers.rs`


> 使用 `Actix` 框架，`state`（应用程序状态） 就可以被注入到请求它的 `Handler` 中，这些 `Handler` 就可以根据方法签名中的参数来访问这些 `state`
{: .prompt-info }


* Step 3：运行

```shell
// 运行 teacher-service，因为配置了 default-run 所以 cargo run 不指定二进制默认运行 teacher-service
$ cargo run -p webservice
```

可以再开一个命令行用 `curl` 进行验证，可 `curl` 多次，以便验证次数

```shell
$ curl localhost:3000/health
```

