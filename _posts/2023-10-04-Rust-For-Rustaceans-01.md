---
layout: post
title: Rust 中级教程-接口设计建议-01-Unsurprising
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


* 注意：`派生的 Trait（这样即派生 #[derive(Debug)]）` 会为任意泛型参数添加相同的约束（bound），请看下边代码例子

```rust
use std::fmt::Debug;

/*
    Pair 通过 derive 派生的方式实现 trait，所以 T 要实现 Debug trait
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

* 如果你的类型不是 `Send` 的类型那么就无法放到 `Mutex`（互斥锁）中，也不能在包含线程池的应用程序中传递使用，请看下边例子

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

* 如果你的类型不是 `Sync` 的类型那么就无法通过 `Arc` 进行共享，也不能被放置在静态变量中，请看下边例子


```rust
use std::cell::RefCell;
use std::env::consts::ARCH;
use std::sync::Arc;

fn main() {
    let x = Arc::new(RefCell::new(42));
    std::thread::spawn(move || {
        let mut x = x.borrow_mut();
        // 因为 RefCell 没有实现 Sync 所以下边报错
        *x += 1; // error `RefCell<i32>` cannot be shared between threads safely
    });
}
```

> 如果你的类型没有实现上述 `trait`，建议在文档中说明
{: .prompt-info }


### 2.3.2.3 建议实现 `Clone Trait` 和 `Default Trait`

* 实现 `Clone Trait` 的例子

```rust
#[derive(Debug, Clone)]
struct Person {
    name: String,
    age: u32,
}

impl Person {
    fn new(name: String, age: u32) -> Person {
        Person { name, age }
    }
}

fn main() {
    let p1 = Person::new("Alice".to_owned(), 22);
    let p2 = p1.clone();

    println!("p1: {:?}", p1);
    println!("p2: {:?}", p2);
}
```


* 实现 `Default Trait` 的例子，`Default` 即能够提供一个默认的初始值


```rust
#[derive(Default)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let point = Point::default();
    println!("Point: ({}, {})", point.x, point.y);
}
```




> 如果你的类型没有实现上述 `trait`，建议在文档中说明
{: .prompt-info }


### 2.3.2.4 建议实现 `PartialEq、PartialOrd、Hash、Eq、Ord`

* `PartialEq` 特别有用
1. 因为用户会希望使用 `==` 或 `assert_eq!` 比较你的类型的两个实例

```rust
// 实现 PartialEq
#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2};
    let p2 = Point { x: 1, y: 2};
    let p3 = Point { x: 3, y: 4};

    // 使用 == 比较它们是否相等
    println!("Point1 == Point2: {}", p1 == p2);
    println!("Point1 == Point3: {}", p1 == p3);
}
```

#### `PartialOrd、Hash` 相对更专门化一些

* 如果需要将类型作为 `Map` 中的 `Key` 使用的时候，就必须要求其类型实现 `PartialOrd`，以便 `Key` 可以进行比较

```rust
// PartialOrd 例子
use std::collections::BTreeMap;

// 实现了这些 trait
// Ord 需要实现 PartialOrd
#[derive(Debug, PartialEq, Eq, Clone)]
struct Person {
    name: String,
    age: u32,
}

fn main() {
    let mut ages = BTreeMap::new();

    let person1 = Person {
        name: "Alice".to_owned(),
        age: 25,
    };

    let person2 = Person {
        name: "Bob".to_owned(),
        age: 23,
    };

    let person3 = Person {
        name: "Cook".to_owned(),
        age: 31,
    };

    // 去掉实现 PartialOrd，这里报错
    ages.insert(person1.clone(), "Alice's age");
    ages.insert(person2.clone(), "Bob's age");
    ages.insert(person3.clone(), "Cook's age");
 
    for (person, desc) in &ages {
        println!("{}: {} - {:?}", person.name, person.age, desc);
    }
}
```

* 使用 `std::collection` 的集合类型进行去重的类型，就必须要求其类型实现 `Hash`，以便进行哈希计算

```rust
use std::collections::HashSet;
use std::hash::{Hash, Hasher};

#[derive(Debug, PartialEq, Eq, Clone)]
struct Person {
    name: String,
    age: u32,
}

impl Hash for Person {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.name.hash(state);
        self.age.hash(state);
    }
}

fn main() {
    let mut persons = HashSet::new();

    let person1 = Person {
        name: "Alice".to_owned(),
        age: 30,
    };

    let person2 = Person {
        name: "Bob".to_owned(),
        age: 20,
    };

    let person3 = Person {
        name: "Charlie".to_owned(),
        age: 40,
    };

    persons.insert(person1.clone());
    persons.insert(person2.clone());
    persons.insert(person3.clone());

    println!("Person Set {:?}", persons);

}
```


#### `Eq 和 Ord` 相对于 `PartialEq 和 PartialOrd` 有额外的语义要求
1. 只应该在确信这些语义适用于你的类型时才实现它们
2. 相当于 `PartialEq 和 PartialOrd` 的扩展


* `Eq` 的额外要求
1. 反身性（Reflexivity）：对于任何对象 `x`，`x == x 必须为真`
2. 对称性（Symmetry）：对于任何对象 `x 和 y`，`如果 x == y 为真，则 y == x 也必须为真`
3. 传递性（Transitivity）：对于任何对象 `x、y、z`，`如果 x == y 为真，并且 y == z 为真，则 x == z 也必须为真`

* `Ord` 的额外要求
1. 反身性（Reflexivity）：对于任何对象 `x`，`x <= x 和 x >= x 必须为真`
2. 反对称性（Antisymmetry）：对于任何对象 `x 和 y`，`如果 x <= y 和 y <= x 都为真，则 x == y 必须为真`
3. 传递性（Transitivity）：对于任何对象 `x、y、z`，`如果 x <= y 和 y <= z 都为真，则 x <= z 必须为真`

#### 建议实现 `serde` 下的 `Serialize、Deserialize` 
* `serde_derive crate` 提供了一些机制，可以覆盖单个字段或枚举变体的序列化
1. 由于 `serde` 是一个第三方库，你可能不希望强制添加对它的依赖
2. 大多数库选择提供一个 `serde` 功能，只有当用户选择启用该功能时才添加对 `serde` 的支持

```rust
// 你的库的 Cargo.toml  
[dependencies]
serde = { version = "1.0", optional = true }

[features]
serde = ["serde"]


// 别人用时候依赖你的库, features 中有 "serde"，才算启用
[dependencies]
mylib = {version = "0.1", features = ["serde"]}
```

#### 为什么没建议实现 `Copy`
* 用户通常不期望类型是 `Copy` 的，如果想要两个副本，通常希望调用 `Clone`

* `Copy` 改变了移动给定类型值的语义，这点会让用户感到意外

* `Copy` 类型受到很多限制，一个最初简单类型很容易变成不再满足 `Copy` 的要求
1. 例如一个类型原来不持有 String，当需求发送变化时它持有 String 时，就不得不移除 `Copy`

```rust
#[derive(Debug, Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 10, y: 20 };
    let p2 = p1; // Note：这里发送的是 Copy 而不是 Move

    println!("p1: {:?}", p1);
    println!("p2: {:?}", p2);
}
```

#### 小结

> 建议实现： `Trait` 有 `Debug、Send、Sync、Default、Clone、PartialEq、PartialOrd、Hash、Eq、Ord`，以及最小程度实现 `unpin`。如果你的类型没有实现上述 `Trait`，建议在文档中说明。不建议实现 `Copy`。
{: .prompt-info }


### 2.3.3 人体工程学的 Trait 实现
* Rust 不会自动为实现 Trait 的类型的引用提供对应的实现
1. 例如，`Bar` 类型实现了某个 `Trait`，有这样一个方法 `fn foo<T: Trait>(t: T)`，由于 `Bar` 实现了这个 `Trait`, `Bar` 肯定可以传入这个方法中，
但是 `Bar` 的引用（`&Bar`）不能传进去，因为这个 `Trait` 可能包含接受 `&mut self` 或 `self` 的方法，而这些方法无法在 `&Bar` 上进行调用

### 对应看到某个 `Trait` 只有 `&self` 方法，而不接受 `&mut self` 或 `self` 时，就会令人非常惊讶，就要解决这个问题，如果解决？
* 即定义新的 `Trait` 时，通常需要为下列提供相应的全局实现
1. `&T where T: Trait`
2. `&mut T where T: Trait`
3. `Box<T> where T: Trait`

### 另外一个场景迭代器
* 如果你的类型可以迭代的话，那么类型的引用也应该添加相应的 `Trait` 实现
1. 即对于任何可迭代的类型来说，考虑为你类型的引用 `&MyType 和 &mut MyType 来实现 IntoIterator trait`,这样在循环中就可以直接使用借用的实例，符合预期


### 2.3.4 包装类型（Wrapper Types）
* Rust 没有传统意义上的继承
#### 2.3.4.1 `Deref 和 AsRef` 提供了类似继承的东西
1. 假如你有一个类型 `T` 的值，并且 `T 满足 T: Deref<Target = U>`，那么就可以在 `T` 的类型的值上直接调用类型 `U` 的方法

```rust
use std::ops::Deref;

struct MyVec(Vec<i32>);

// 为 MyVec 实现 Deref，目标类型是 Vec<i32>
impl Deref for MyVec {
    type Target = Vec<i32>;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
    let my_vec = MyVec(vec![1, 2, 3, 4, 5]);
    
    /*
        由于 MyVec 实现了 Deref 
        所以可以直接在 MyVec 类型的值上直接调用 Vec<i32> 的方法 len()
        也可以通过索引获取第一个元素
     */
    println!("Length: {}", my_vec.len());
    println!("First element: {}", my_vec[0]); 
}
```

#### 2.3.4.2 如果你提供了相对透明的类型，例如 `Arc`
1. 那么实现 `Deref` 就允许你的包装类型在使用电运算符时，就会自动解引用为内部类型，从而可以直接调用内部类型的方法
2. 如果访问内部类型不需要任何复杂或潜在的低效逻辑，应考虑实现 `AsRef`，这样用户可以轻松的将 `&WrapperType`(包装类型) 作为 `&InnerType`（内部类型） 来使用
3. 对于大多数包装类型，还应该在可能得情况下实现 `From<InnerType>` 和 `Into<InnerType>`，以便用户可以轻松添加或移除包装类型

```rust
use std::ops::Deref;

#[derive(Debug)]
struct Wrapper(String);

impl Deref for Wrapper {
    type Target = String;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl AsRef<str> for Wrapper {
    fn as_ref(&self) -> &str {
        &self.0
    }
}

// 为 Wrapper 实现 From，可以将 String 转为 Wrapper
impl From<String> for Wrapper {
    fn from(s: String) -> Self {
        Wrapper(s)
    }
}

impl From<Wrapper> for String {
    fn from(wrapper: Wrapper) -> Self {
        wrapper.0
    }
}

fn main() {
    let wrapper = Wrapper::from("Hello".to_string());

    // 使用 . 运算符调用内部字符串类型的方法
    println!("Length: {}", wrapper.len());

    // 使用 as_ref 方法将 Wrapper 转换为 &str 类型
    let inner_ref: &str = wrapper.as_ref();
    println!("Inner: {}", inner_ref);

    // 将 Wrapper 转换为内部类型 String
    let inner_string: String = wrapper.into();
    println!("Inner String: {}", inner_string);

    // let w2: Wrapper = "World".to_string().into();
    // println!("w2 Wrapper: {:?}", w2);

}
```

#### 2.3.4.3 `Borrow Trait`
* 与 `Deref 和 AsRef` 类似，针对更为狭窄的使用情况进行了定制，它允许调用者提供同一类型的多个本质上相同的变体中的任意一个
1. 可叫做 `Equivalent`
2. 例如对于一个 `HashSet<String>`，`Borrow` 就允许调用者提供 `&str` 或 `&String`
3. 虽然使用 `AsRef` 也可以实现类似的效果，但是如果没有 `Borrow` 的额外要求，这种实现是不安全的，因为 `Borrow` 要求目标类型实现的 `Hash/Eq/Ord` 必然与实现类型完全相同，
4. 并且 `Borrow` 还为 `Borrow<T>/&T/&mut T` 提供了通用实现，这使得在 `Trait 约束` 中使用它来接受给定类型的拥有值或引用值非常方便


> `Borrow` 仅仅适用于当你的类型本质上与另一个类型等价时，才适合使用 `Borrow`，而 `Deref 和 AsRef` 则适用于更广泛的实现你的类型可以”充当“的情况
{: .prompt-info }

* `Borrow` 例子

```rust
use std::borrow::Borrow;

fn print_length<S>(string: S) where S: Borrow<str>, {
    println!("Length: {}", string.borrow().len());
}

fn main() {
    let str1: &str = "Hello";
    let string1: String = String::from("World");

    print_length(str1);
    print_length(string1);
}
```
