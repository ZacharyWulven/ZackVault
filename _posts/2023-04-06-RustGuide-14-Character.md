---
layout: post
title: RustGuide-14-Character
date: 2023-04-06 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 17 Rust 的面向对象编程特性

## 面向对象语言的特性


## 封装

```rust
pub struct AveragedCollection {
    list: Vec<i32>,  // 私有
    average: f64,    // 私有
}

impl AveragedCollection {
    pub fn add(&mut self, value: i32) {
        self.list.push(value);
        self.update_average();
    }

    pub fn remove(&mut self) -> Option<i32> {
        let result = self.list.pop();
        match result {
            Some(value) => {
                self.update_average();
                Some(value)
            },
            None => None,
        }
    }

    pub fn average(&self) -> f64 {
        self.average
    }
    
    // 私有方法
    fn update_average(&mut self) {
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }

}
```


### 继承
* Rust 里没有继承
* 使用继承的原因：代码复用
1. Rust 中默认使用 `trait` 方法来进行代码共享，因为如果`某 trait` 有默认实现那么任何实现了这个 `trait` 的类型就自动拥有那些方法
2. 实现了`某 trait` 的类型也可以覆盖 `trait` 的默认实现

> 即 Rust 实现继承是通过面向协议的方式
{: .prompt-info }


* 多态
1. Rust 中通过`泛型`和 `trait 约束（限定参数化多态 bounded parametric）`来实现多态，即通过类型抽象来实现

> 现在很多新语言都不使用继承作为内置的设计方案了
{: .prompt-info }

## 使用 trait 对象来存储不同类型的值
有这样一个需求，创建一个 GUI 工具，它会遍历某个元素列表，依次调用元素的 draw 方法进行绘制，例如 Button、TextField 等元素

* 在面向对象语言里，一般是定义一个 Component 的父类，里面定义了 draw 方法，定义 Button、TextField 等类，继承于 Component 类
* Rust 中通过定义一个 trait 实现

> Rust 避免将 `struct` 或 `enum` 称为对象，因为它们与 `impl` 块是分开的
{: .prompt-info }

### `trait` 对象
* `trait` 对象有些类似于其他语言中的对象，某种程度上是组合了数据与行为
* `trait` 对象与传统对象不同的地方：无法为 `trait` 对象添加数据
* `trait` 对象实际是被专门用于抽象某些共有行为的，它没有其他语言中的对象那么通用


```rust
// lib.rs
pub trait Draw {
    fn draw(&self);
}

pub struct Screen {
    // Vec 里的元素是实现了 Draw trait 的类型
    pub components: Vec<Box<dyn Draw>>,
}

impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}

pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        println!("绘制一个 Button");
    }
}

// main.rs
use GUI_17::{ Draw, Button, Screen };

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        println!("绘制一个选择框");
    }
}

fn main() {
    demo_gui();
}

fn demo_gui() {
    let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No"),
                ],
            }), 
            Box::new(Button {
                width: 50,
                height: 20,
                label: String::from("Ok"),
            }),
        ]
    };
    screen.run();
}
```

### Trait 对象执行的是动态派发
* 将 `trait` 约束作用于泛型时，Rust 编译器会执行单态化操作
1. 即编译器会为我们用来替换泛型类型参数的每一个具体类型生成对应类型的函数和方法的非泛型实现
2. 通过单态化生成的代码会执行静态派发（static dispatch），在编译过程中确定调用的具体方法

* 动态派发（dynamic dispatch）
1. 无法在编译过程中确定你调用的究竟是哪个方法
2. 编译器会产生额外的代码以便于在运行时找出希望调用的方法
* 如果使用 `trait` 对象，就会执行动态派发
1. 这就会导致产生运行时的开销
2. 并且阻止编译器内联方法代码，使得部分优化操作无法进行

### Trait 对象必须保证对象安全
* 只能把满足对象安全（object-safe）的 `trait` 转化为 `trait` 对象
* Rust 采用一系列规则来判定某个对象是否安全，只需记住两条：满足这两条这个对象就是安全的
1. `方法的返回类型不是 Self`
2. `方法中不包含任何泛型类型参数`

* 标准库 Clone trait 就是一个不符合安全规则的例子

```rust
// 不安全的因为它返回 Self
pub trait Clone {
    fn clone(&self) -> Self;
}


// 这样就会报错，因为 Clone trait 返回 Self，因此不能称为 trait 对象
pub struct Screen {
    pub components: Vec<Box<dyn Clone>>,
}
```

## 实现面向对象的设计模式

### 状态模式（state pattern）
* 一个值拥有的内部状态是由数个状态对象（state object）表达而成，而值的行为则随着内部状态的改变而改变
* 使用状态模式就意味着：
1. 业务需求变化时，不需要修改持有状态的值的代码，或者使用这个值的代码
2. 只需要更新状态对象内部的代码，以改变其规则。或者增加一些新的状态对象

```rust
//lib.rs

pub trait State {
    /*
        Box<Self> 只能被 Box 包裹的实例调用
        这里会获取 self 的所有权，并将旧值失效
     */
    fn request_review(self: Box<Self>) -> Box<dyn State>;

    fn approve(self: Box<Self>) -> Box<dyn State>;

    fn content<'a>(&self, post: &'a Post) -> &'a str {
        ""
    }

}
pub struct Post {
    state: Option<Box<dyn State>>,
    content: String,
}

impl Post {
    pub fn new() -> Post {
        Post { state: Some(Box::new(Draft {})), content: String::new() }
    }

    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
    pub fn content(&self) -> &str {
        /*
            返回值的引用 Option<&T>
            这里 unwrap 不会 panic 因为 T 总是有值
         */
        self.state.as_ref().unwrap().content(&self)
        
    }

    pub fn request_review(&mut self) {
        // 
        /*
            使用 take 函数取出 some 的值，同时所有权 move 出来了
            self.state.take() 之后 self.state 的值就是 None
         */
        if let Some(s) = self.state.take() {
            self.state = Some(s.request_review());
        }
    }
    
    pub fn approve(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve());
        }
    }
}



struct Draft {

}

impl State for Draft {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        Box::new(PendingReview {})
    }
    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
}

struct PendingReview {

}

impl State for PendingReview {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }
    fn approve(self: Box<Self>) -> Box<dyn State> {
        Box::new(Published {})
    }
}

struct Published {

}

impl State for Published {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }
    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }

    fn content<'a>(&self, post: &'a Post) -> &'a str {
        &post.content
    }
}

//main.rs
use blog_17::Post;

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    assert_eq!("", post.content());

    post.request_review();
    assert_eq!("", post.content());

    post.approve();
    assert_eq!("I ate a salad for lunch today", post.content());


}
```

### 状态模式的权衡取舍
* 缺点：
1. 某些状态之间是相互耦合的，如果新增一个状态与其相关联的状态就需要修改
2. 需要重复实现一些逻辑代码

### 将状态和行为编码为类型（更 Rust 的方式）
* 将状态编码为不同的类型
1. Rust 类型检查系统会通过编译时错误来阻止用户使用无效的状态

* 使用 Rust 方式改进上述 Blog 需求


```rust
//lib.rs
pub struct Post {
    content: String,
}

pub struct DraftPost {
    content: String,
}

impl Post {
    // new 时返回草稿 Post（DraftPost）
    pub fn new() -> DraftPost {
        DraftPost { content: String::new() }
    }

    pub fn content(&self) -> &str {
        &self.content
    }
}

impl DraftPost {
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }

    // 获得所有权，并返回新 struct
    pub fn request_review(self) -> PendingReviewPost {
        PendingReviewPost { content: self.content }
    }
}

pub struct PendingReviewPost {
    content: String,
}

impl PendingReviewPost {
    /*
        只有发布成功返回 Post，Post 才有 content 方法
        达到了未 approve 前不能查看其内容目的
     */
    // 获得所有权，并返回新 struct
    pub fn approve(self) -> Post {
        Post { content: self.content }
    }
}

// main.rs
use blog_17_gai::Post;

fn main() {

    let mut post = Post::new();
    post.add_text("I ate a salad for lunch today");

    let post = post.request_review();
    let post = post.approve();

    assert_eq!("I ate a salad for lunch today", post.content());
}
```

### 总结
* Rust 不仅能够实现面向对象的设计模式，还可以支持更多的模式
1. 例如将状态和行为编码为类型

* 面向对象的经典模式并不总是 Rust 编程实践中的最佳选择，因为 Rust 具有所有权等其他面向对象语言没有的特性
