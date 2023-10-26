---
layout: post
title: Rust Web 全栈开发教程-05-features
date: 2023-10-01 16:45:30.000000000 +09:00
categories: [Rust, Rust Web]
tags: [Rust, Rust Web]
---

# 8 添加教师管理功能

## 8.1 目标

![image](/assets/images/rust/web_server/teacher_aim.png)


## 8.2 代码

* sql

```
drop table if exists teacher;

create table teacher(
    id serial primary key,
    name varchar(100) not null,
    picture_url varchar(200) not null,
    profile varchar(2000) not null,
);

insert into teacher(id, name, picture_url, profile) values(1, 'Tom', 'http://first.png', 'first')
```

> 如果遇到端口占用，可执行命令 `$ lsof -i:[端口号]`, 然后 `$ kill -9 [PID]` 即可
{: .prompt-info }

[添加教师功能代码](https://github.com/ZacharyWulven/Rust_Web_Full_Stack_Guide/commit/6e989020863bc4b3bce9b795f8dfdde4d209e2db)


# 9 服务器端 Web 应用（非本课重点）
* 服务器端 Web 应用就是在服务器端把页面渲染好

## 9.1 主要技术
* 模板引擎：`Tera`
1. 模板引擎的作用就是可以把静态的页面进行渲染，把动态的数据渲染到模板里然后一同返回给用户

## 9.2 操作

* Step 1：在 `workspace` 根目录

```shell
$ cargo new webapp
```

* 然后编辑 `workspace` 的 `Cargo.toml` 添加 `webapp`

```rust
# 这是 workspace 的 toml

[workspace]
members = ["webservice", "webapp"]
```

* Step 2：编辑 `webapp` 的 `Cargo.toml`

```rust
[package]
name = "webapp"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
actix-files = "0.6.0-beta.16"  # 处理静态文件
actix-web = "4.0.0-rc.2"
awc = "3.0.0-beta.21" # 相当于一个 http 客户端
dotenv = "0.15.0"
serde = { version = "1.0.136", features = ["derive"]}
serde_json = "1.0.79"
tera = "1.15.0"
```

* Step 3：新增 `webapp/.env` 文件，设置这个 webapp 运行的地址和端口

```
HOST_PORT=127.0.0.1:8080
```

[webapp 代码](https://github.com/ZacharyWulven/Rust_Web_Full_Stack_Guide/commit/46906e5b6a5f395b9375f6c3210d19fc361bdab1)


# 10 使用 WebAssembly 编写客户端应用

## 10.1 项目结构和功能

![image](/assets/images/rust/web_server/assembly.png)

## 10.2 什么是 `WebAssembly`
* `WebAssembly` 是一种新的编码方式，可以运行在现代的浏览器中
1. 它是一种低级的类汇编语言
2. 具有紧凑的二进制格式
3. 可以接近原生的性能运行
4. 并为 `C/C++、C#、Rust` 等语言提供一个编译目标，以便它们可以在 Web 上运行
5. 它也被设计为可以与 `JavaScript` 共存的，允许两者一起工作

### 机器码
* 机器码就是计算机可直接执行的语言
* 汇编语言比较接近机器码

![image](/assets/images/rust/web_server/assembly_01.png)


### 机器码与 CPU 架构
* 不同的 CPU 架构需要不同的机器码和汇编
* 高级语言可以”翻译“成机器码，以便在 CPU 上运行

![image](/assets/images/rust/web_server/assembly_02.png)


### WebAssembly
* WebAssembly 其实不是汇编语言，它不针对特定的机器而是针对浏览器的
* WebAssembly 是中间编译器目标（有点类似 苹果 bitcode 的 IR 中间产物）

![image](/assets/images/rust/web_server/assembly_03.png)


* WebAssembly 有两种格式的文件
1. 文本格式：`.wat`，对应图中的 `TEXTUAL FORMAT`
2. 二进制格式：`.wasm`，对应图中的 `BINARY FORMAT`

![image](/assets/images/rust/web_server/assembly_04.png)


### WebAssembly 能做什么呢？
* 可以把你编写的 `C/C++、C#、Rust` 等语言代码编译成 WebAssembly 模块，这样就可以在 `Web` 应用中加载该模块，并通过 `JavaScript` 调用这些模块
* 它并不是为了替代 `JS`，而是与 JS 一起工作
* 在使用 `WebAssembly` 时，依然要使用 `HTML 和 JS`，因为 `WebAssembly` 无法访问平台 `API`，例如 `DOM，WebGL`


### WebAssembly 如何工作？
* 这个一个 C++ 的例子
* 将源代码 `hello.c` 编译为 `hello.wasm`, `hello.wasm` 就可以在浏览器中配合 `HTML 和 JS` 运行了

![image](/assets/images/rust/web_server/assembly_05.png)


### WebAssembly 优点


![image](/assets/images/rust/web_server/assembly_06.png)


## 10.3 搭建环境

[WebAssembly Build Doc](https://rustwasm.github.io/docs/book/)

可直接看第四章教程先安装

```shell
# 安装 wasm-pack
$ curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh\n


# 安装 cargo-generate
$ cargo install cargo-generate

# (可选) 如果 cargo-generate 安装失败可以升级下 rustc
$ rustup update


# 安装 npm（如果没装的话，就是装 node.js）
$ npm --version # 检查 npm 是否安装
```




