---
layout: post
title: Rust 中级教程-接口设计建议-01
date: 2023-10-04 16:45:30.000000000 +09:00
categories: [Rust, Rustaceans]
tags: [Rust, Rustaceans]
---


# 1 简介

[rust-for-rustaceans](https://rust-for-rustaceans.com/)

<!--![image](/assets/images/rust/web_server/teacher_aim.png)-->

## 本教程 4 个原则
1. 不意外（unsurprising）
2. 灵活（flexible）
3. 显而易见（obvious）
4. 受约束（constrained）


[Rust API 指南 ](https://rust-lang.github.io/api-guidelines/)


# 2 不意外（unsurprising）

## 2.1 也叫最少意外原则
1. 意思是你写的接口应该尽可能的直观
2. 可预测，用户一看到这个接口应该能大概才出来是干什么用的
3. 至少应该不让人感到意外

## 2.2 核心思想
* 贴近用户已经知道的东西，用户不必重新学习一些概念
1. 比如你定义一个 `xxxError trait` 那么应该就是类似标准库中 `Error` 类似功能

## 2.3 让接口可预测
* 命名方面 
* 要实现一些常用的 `Traits`，例如实现 `Debug trait`
* 人体工程学（Ergonomic）Traits
* 包装类型（Wrapper Type）


### 2.3.1 命名
* 接口的名称应该符合惯例（Rust 标准库或社区常用的惯例），便于推断其功能
1. 例如一个叫 `iter 方法`，大概率应该将 `&self` 作为参数，并返回一个迭代器（iterator）
2. 例如一个叫 `into_inner` 的方法，大概率应将 `self` 作为参数，并返回某个包装的类型
3. 例如一个叫 `SomethingError` 的类型，应该实现 `std::error::Error trait`，并出现在各类 `Result` 中


> 小结：功能与命名更语义化，所见即所得
{: .prompt-info }


* 即将通用、常用的名称依然用于相同的目的，让用户好猜、好理解

> 推论：同名的事物应该以相同的方式工作，否则用户大概率会写出错误的代码
{: .prompt-info }


### 2.3.2 实现常用的 `Trait`
* 用户通常会假设接口中的一切均可“正常工作”（按用户预期的工作）
1. 例如使用 `{:?}` 可打印任何类型
2. 可以发送任何东西到另外的线程
3. 用户期望每个类型都是 `Clone` 的

> 建议积极实现大部分标准 `Trait`，即使不会立即用到
{: .prompt-info }

* 由于 Rust 无法为外部类型实现外部的 `Trait`，所以你最好实现这种 `Trait`
1. 例如，`Debug Trait` 是标准库 `Trait`，而你的类型是外部 `Trait`


### 2.3.2.1 建议实现 `Debug Trait`

> 注意：几乎所有的类型都能而且应该实现 `Debug Trait`，最佳实践是 `#[derive(Debug)]`
{: .prompt-info }


* 注意：派生的 `Trait` 会为任意泛型参数添加相同的约束（bound），请看下边代码例子

```rust
use std::fmt::Debug;

/*
    Pair 使用派生的方式实现 trait，所以 T 要实现 Debug trait
*/
#[derive(Debug)]
struct Pair<T> {
    a: T,
    b: T,
}

struct Person {
    name: String,
}

fn main() {
    let pair = Pair { a: 5, b: 10 };
    // i32 已经实现了 Debug trait，所以下边打印没有问题
    println!("Pair: {:?}", pair);

    let pair = Pair {
        a: Person { name: "Dave".to_string() },
        b: Person { name: "Tom".to_string() },
    };
    // Note：由于 Person 没有实现 Debug trait，所以这里打印报错
    println!("Pair: {:?}", pair);

}
```

* 利用 `fmt::Formatter` 提供的各种 `debug_xxx` 辅助方法手段实现 `Debug Trait`，提供的辅助方法有
1. `deubg_struct`，针对 `struct` 的
2. `deubg_tuple`
3. `deubg_list`
4. `deubg_set`
5. `deubg_map`


```rust
use std::fmt;

struct Pair<T> {
    a: T,
    b: T,
}

// 这里即手段实现 Debug trait
impl<T: fmt::Debug> fmt::Debug for Pair<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // 由于 Pair 是 struct，所以这里使用 debug_struct 方法实现
        f.debug_struct("Pair")    // 名称是 Pair
            .field("a", &self.a)
            .field("b", &self.b)
            .finish()
    }
}

fn main() {
    let pair = Pair { a: 5, b: 10};

    println!("Pair: {:?}", pair);
}
```

### 2.3.2.2 建议实现 `Send Trait` 和 `Sync Trait`、最小程度 `unpin`

* 如果你的类型不是 `Send` 的类型那么就无法放到 `Mutex`（互斥锁）中，也不能在包含线程池的应用程序中传递使用
* `Send` 例子

```rust
use std::rc::Rc;

#[derive(Debug)]
struct MyBox(*mut u8);

unsafe impl Send for MyBox {
    
}


fn main() {
    println!("Hello, world!");

    let mb = MyBox(Box::into_raw(Box::new(42)));
    let x = Rc::new(42);

    
    std::thread::spawn(move || {
        /*
            因为 Rc 没有实现 Send trait
            所以下边跨线程使用 x 时候就会报错
         */
        println!("{:?}", x); // error: `Rc<i32>` cannot be sent between threads safely

        // 而 MyBox 实现了 Send，所以这里不会报错
        println!("{:?}", mb);
    });

}
```



#### 小结

> 建议实现： `Trait` 有 `Debug、Send、Sync`，以及最小程度实现 `unpin`
{: .prompt-info }

