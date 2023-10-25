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
