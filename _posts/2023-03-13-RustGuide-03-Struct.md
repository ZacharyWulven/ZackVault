---
layout: post
title: RustGuide-03-Struct
date: 2023-03-13 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 5 Struct

## 定义并实例化 struct
* 使用 struct 关键字
* 花括号内，为所有字段（Field）定义名称和类型
* 实例化
1. 为每个字段指定具体的值
2. 无需按声明的顺序指定

```rust
struct User {
    name: String,
    email: String,
    sign_in_count: u64,
    active: bool, 
}

// 字段顺序可以不一样
// 这种方式实例化，如果少字段没有赋值 会报错
let user = User {
    email: String::from("tom@gmail.com"),
    sign_in_count: 42,
    name: String::from("tom"),
    active: true,
};
```

* 获取值与赋值

```rust
    let mut user = User {
        email: String::from("tom@gmail.com"),
        sign_in_count: 42,
        name: String::from("tom"),
        active: true,
    };
    println!("{}", user.email);
    
    // 这里赋值必须 user 是可变的，要加 mut
    user.email = String::from("zack@gmail.com");

    println!("{}", user.email);
```

> 一旦一个 struct 实例时可变的，那么它的所有字段就都是可变的
{: .prompt-info }


* struct 作为返回值

```rust
fn build_user(name: String, email: String) -> User {
    User {
        email: email,
        sign_in_count: 42,
        name: name,
        active: true,
    }
}
```

* 字段初始化简写
1. 当字段名称与字段值对应的变量名相同时，就可以使用字段初始化简写方式

```rust
fn easy_build_user(name: String, email: String) -> User {
    User {
        email,  // 简写
        sign_in_count: 42,
        name,
        active: true,
    }
}
```

* struct 更新语法
1. 即你想基于某个 struct 实例来创建一个新实例的时候，可以使用 struct 更新语法
2. 下例 u2 的除 name 和 email 字段，其他字段都使用 u1 的值

```rust
    let u1 = easy_build_user(String::from("jack"), String::from("jack@gmail.com"));
    println!("{}", u1.email);

    let u2 = User {
        email: String::from("u2@gmail.com"),
        name: String::from("u2"),
        ..u1    // 注意这里没有逗号 ,
    };
    println!("{}", u2.email);
```

## Tuple Struct
* 类似 Tuple 的 Struct
* Tuple Struct 整体有个名，但里边的元素没有名
* 适用场景：想给一个 Tuple 整个起个名称，并让他不同于其他 Tuple，而且又不需要给每个元素起名
* 定义 Tuple Struct

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

fn test_tuple_sturct() {
    let black = Color(0, 0, 0);

    let origin = Point(0,0,0);

    // Note：black 与 origin 是不同类型，因为是不同的 tuple struct 的实例
    println!("{}", black.0);
}
```
## Unit-Like Struct（没有任何字段）
* 在 Rust 里可以定义没有任何字段的 struct，叫做 Unit-Like Struct 
* 适用场景：需要在某个类型上实现某个 trait，但是在里面又没有想要存储的数据


```rust
struct AlwaysEqual;

fn main() {
    let subject = AlwaysEqual;
}
```


## struct 数据的所有权
* 以 User 为例
1. name 和 email 的类型是 String 而不是 &str，相当于持有对 name 和 email 的所有权
2. 而其他俩字段是标量类型，所以 User 就拥有对其所有字段的所有权，即 User 的实例拥有其所有数据
3. 只有 User 实例有效，那么里边的字段数据也是有效的

* 但 struct 里的字段也可以是引用，这需要生命周期
1. 生命周期可以保证只有 struct 实例时有效的，那么里边的引用也是有效的
2. 否则，如果 struct 里存储的是引用，而不是使用生命周期的话，就行报错


```rust
struct User {
    name: String,
    email: String,
    sign_in_count: u64,
    active: bool, 
}

// 使用 slice（引用）定义 User
struct User {
    name: &str,    // 这样会报错，因为没有指定生命周期，生命周期后边讲
    email: &str,
    sign_in_count: u64,
    active: bool, 
}
```

## struct 例子
```rust

fn main() {
    let w = 30;
    let l = 50;
    println!("area = {}", area(w, l));

    // 使用元组
    let rect = (30, 50);
    println!("area = {}", area_tuple(rect));

    // 使用 struct，最优方案，语义明确
    let rectangle = Rectangle{width: 30,  length: 50};
    println!("area = {}", area_struct(&rectangle));

    // 
    /*
        {} 告诉 println! 使用 Display trait 格式化
        而有些类型没有实现 Display trait 就会报错 
        如果修复
        方式一：可以使用 {:?}, 但需要在 struct 定义上一行声明 #[derive(Debug)]
        #[derive(Debug)] 是让 struct 选择 Debug 功能
        方式二：使用 {:#?} 
         
     */
    println!("{:#?}", rectangle);
    
    // 或这样打印
    println!("{rectangle:#?}");
}

fn area(width: u32, length: u32) -> u32 {
    width * length
}

fn area_tuple(dim: (u32, u32)) -> u32 {
    dim.0 * dim.1
}

// derive 是派生的意思
#[derive(Debug)] // 意思是让 Rectangle 派生于 Debug 这个 trait
struct Rectangle {
    width: u32,
    length: u32, 
}
// 传递引用，不获取其所有权
fn area_struct(rect: &Rectangle) -> u32 {
    rect.width * rect.length
}

```


> Tips：`println!("{rectangle:#?}")`
{: .prompt-info }


### println! 这宏的格式化方法有多种
* std::fmt::Display 即 `println!("{}", black.0)`
* std::fmt::Debug 即 #[derive(Debug)]，`println!("{:#?}", rectangle)`
* #[derive(Debug)] 模式下格式化方式 `"{:?}"` 和 `"{:#?}"`（这样会格式化一下可读性更高）

## struct 方法 
* 方法与函数不同之处
1. 方法是在 struct 或 enum 或 trait 对象的上下文中定义的
2. 方法的第一个参数总是 `self`，表示方法被调用的 struct 实例，即正在调用 struct 这个方法的实例

### 如何定义方法
* 定义方法需要在 impl 块里定义
* 方法的第一个参数可以是 &self，也可以 self（获得其所有权）或 &mut self（可变借用），和其他参数一样

## 方法调用的运算符
* Rust 没有 ->(指针指向) 运算符
* Rust 会自动引用或解引用，在调用方法时就会发生这种行为
* 在调用方法时，Rust 根据情况自动添加 &、&mut 或 *，以便 object 可以匹配方法签名
* 下边代码效果相同
1. p1.distance(&p2)
2. (&p1).distance(&p2)

* 方法可以有多个参数

## 关联函数
* 可以在 impl 块里定义不把 self 作为第一个参数的函数，它们叫关联函数（不是方法），类似静态方法或叫类方法
1. 例如 String::from("hello")

* 关联函数通常用于构造器
* :: 符号
1. 用于关联函数
2. 用于 Module 创建的命名空间


## 多个 impl 块
* 每个 struct 允许拥有多个 impl 块

```rust
fn main() {
    let rectangle = Rectangle{width: 30,  length: 50};

    println!("rectangle.area {}", rectangle.area());

    // 调用关联函数
    println!("{:#?}", Rectangle::square(30));
}

// derive 是派生的意思
#[derive(Debug)] // 意思是让 Rectangle 派生于 Debug 这个 trait
struct Rectangle {
    width: u32,
    length: u32, 
}

// 为 struct 定义方法需要 impl 块
impl  Rectangle  {
    // self 就是 Rectangle 类型
    fn area(&self) -> u32 {
        self.width * self.length
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.length > other.length
    }

}

impl Rectangle {
    // 这是一个关联函数
    fn square(width: u32) -> Rectangle {
        Rectangle { width, length: width }
    }

}
```

## 所有权例子
* 1 


```rust
#[derive(Copy, Clone)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {

    fn area(&self) -> u32 {
        self.width * self.height
    }

    /*
        &mut self 可变的
        如果想要调用下边方法，Rectangle 创建完必须赋值给一个 mut 的变量
     */
    fn set_width(&mut self, width: u32) {
        self.width = width;
    }

    /*
        参数为 self，此方法会获取所有权
     */
    fn max(self, other: Self) -> Self {
        let w = self.width.max(other.width);
        let h = self.height.max(other.height);
        Self {
            width: w,
            height: h
        }
    }

    fn set_to_max(&mut self, other: Self) {
        /*
            因为 max 方法是会获得所有权的，而 参数 &mut self 是没有所有权的, 所以报错
            修复方案：让 Rectangle 实现 #[derive(Copy, Clone)] 后，max 方法就不再
                    需要所有权了
                    实际调用时是 Rectangle::max(*self, other); 这样的
                    *self 解引用不会发生所有权的移动，而会发生复制，因为 Rectangle 实现了
                    Copy, Clone Trait, 所以可以复制了

         */
        *self = self.max(other);
    }

}
```

* 2 借用 struct 字段


```rust
struct Point {
    x: i32,
    y: i32
}

fn print_point(p: &Point) {
    println!("Point at ({}, {})", p.x, p.y);
}

fn main() {
    let mut p = Point { x: 0, y: 0 };

    let x = &mut p.x;

    print_point(&p); // Error：这里需要 p 有读的权限，但 p 其实没有读权限

    p.y = *x + 1;  // Ok：这样可以
    *x += 1;
    println!("{}", p.y);
}
```

![image](/assets/images/rust/py.png)


> Tips: `let x = &mut p.x` 后，`p` 和 `p.x` 将失去所有权限，但 `p.y` 仍然是可变的，保持原有权限
{: .prompt-info }


# 6 枚举与模式匹配

## 定义枚举

```rust
fn main() {

    /*
        创建一个枚举值
        枚举的变体（枚举值）都位于标识符的命名空间下，使用两个冒号进行分隔
     */
    let four = IPAddrKind::V4;
    let six = IPAddrKind::V6;
    println!("{:#?}", four);


    let local = IPAddress {
        kind: IPAddrKind::V4,
        address: String::from("127.0.0.1"),
    };
    println!("{:#?}", local);

    let loopback = IPAddress {
        kind: IPAddrKind::V6,
        address: String::from("::1"),
    };
    println!("{:#?}", loopback);

}

// 
fn route(ip: IPAddrKind) {

}

#[derive(Debug)]
enum IPAddrKind {
    V4,
    V6,
}

#[derive(Debug)]
struct IPAddress {
    kind: IPAddrKind,
    address: String,
}
```

### 将数据附加到枚举的变体中
* swift 也有类似的特性
```rust
enum IPAddr {
    V4(String),
    V6(String),
}
```
* 优点
1. 不需要额外使用 struct 来存储相关数据
2. 每个变体可以拥有不同类型以及关联的数据量，例如下列

```rust
enum IPAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

    let home = IPAddr::V4(127, 0, 0, 1);
    let lb = IPAddr::V6(String::from("::1"));

    println!("{:#?}", home);
    println!("{:#?}", lb);
```

## 标准库中的 IpAddr
* 枚举的变体中可以嵌入任意类型的数据

```rust
struct Ipv4Addr {

}

struct Ipv6Addr {

}

enum IpAddr {
  V4(Ipv4Addr),
  V6(Ipv6Addr)
}
```

## 另一个栗子

```rust
#[derive(Debug)]
enum Message {
    Quit,
    Move { x: i32, y: i32}, // 关联的数据类型是一个匿名的 struct
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn test_message() {

    let q = Message::Quit;
    let m = Message::Move { x: 12, y: 12 };
    let w = Message::Write(String::from("Hello"));
    let c = Message::ChangeColor(0, 255, 255);
    
}
```

## 为枚举定义方法
* 也是使用 impl 关键字

```rust
impl Message {
    fn call(&self) { 
        println!("Hello");
    }
}
```

## Option 枚举
* 定义与标准库中
* 在 Prelude（预导入模块）中
* 描述了：某个值可能存在（是某个类型）或不存在的情况

### Rust 没有 Null
* 其他语言中 Null 是一个值，表示“没有值”
* 一个变量可以处于两种状态：空值（null）或非空
* Null 的问题在于：当你尝试像使用非 Null 值那样使用 Null 值的时候，就会引起某种错误，因此 Rust 没有 Null
* 但 Null 的概念还是有用的：因某种原因而变为无效或缺失的值

### Rust 中提供了类似 Null 概念的枚举，即 `Option<T>`
* 在标准库中定义是这样的

```rust
enum Option<T> {
  Some(T),
  None,
}
```

* 可以直接使用 `Option<T>` 和 `Some(T)` 和 None

```rust
    let some_str = Some("A String");
    let some_num =  Some(5);

    let absent_num: Option<i32> = None; // 这是一个无效的值
```

* `Option<T>` 比 Null 好在哪？
1. `Option<T>` 与 T 是不同类型，不可以把 `Option<T>` 直接当成 T
2. 若想使用 `Option<T>` 中的 T 必须将它转换为 T

```rust
    let x: i8 = 5;
    let y: Option<i8> = Some(5);
    let sum = x + y; // 这里报错，因为 y 不是 i8 类型
```

## match
* match 是一个控制流运算符
* match 允许一个值与一系列模式进行匹配，并执行成功匹配的模式对应的代码
* 这些模式可以是字面量、变量名、通配符


### 绑定值的模式
* 匹配的分支可以绑定到被匹配对象的部分值
1. 因此，可以从 enum 值中提取值

```rust
#[derive(Debug)]
enum USState {
    Alabama,
    Alaska,
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(USState),
}

fn value_in_cents(coin: Coin) -> u8 {
    /*
        match 会把 coin 与里边模式依次进行比较
        如果某匹配成功，那个模式的代码就被执行，模式后边的是一个表达式
        如果匹配失败，就往下继续匹配
        匹配成功代码表达式会作为整个 match 的最终结果返回

        这个函数的最后一个表达式就是 match 表达式
     */
    match coin {
        Coin::Penny => 1, // 这里 Coin::Penny 是一个模式, 简单表达式 => 直接写就行
        Coin::Nickel => { // 复杂表达式需要 => 后使用 {}
            println!("5");
            5
        },
        Coin::Dime => 10,  
        Coin::Quarter(state) => { //绑定值的模式匹配
            println!("State quarter from {:#?}!", state);
            25
        },
    }
}

fn test_match() {

    let cent = value_in_cents(Coin::Quarter(USState::Alabama));
    println!("{}", cent);

}
```

### 匹配 `Option<T>`

```rust
fn main() {
    let five = Some(5);
    let six = plus_one(five);
    let none = plus_one(None);
}

fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}
```


> match 匹配必须穷举所有可能。
{: .prompt-info }


```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        Some(i) => Some(i + 1), // 这样会报错，因为没有穷举所有的模式，缺少 None 的匹配
    }
}
```


> `_` 通配符可以替代其余没有列出的值
{: .prompt-info }


```rust
let v = 0u8; // 定义一个 u8 类型
match v {
    1 => println!("one"),
    2 => println!("two"),
    // other => println!("other {:#?}", other),   // 其他 case 打印 other
    _ => (), // 或者使用 _ 通配符代替。_ 通配符必须在最后一行写。它的代码是()一个空元组，即什么也不做
}
```

### if let
* 处理只关心一种匹配而忽略其他匹配的情况
* 更少的代码，更少的缩进，更少的模板代码
* 放弃了穷举的可能
* 可以用 else 代替 `_ 通配符`


```rust

    let v2 = Some(3u8);
    match v2 {
        Some(3) => println!("three"),
        _ => println!("if let else others"),
    }
等价于这么写
    if let Some(3) = v2 {
        println!("if let three");
    } else {
        println!("if let else others");
    }

```

> 可以把 if let 看作是 match 的语法糖，即只根据某一个模式来运行代码，其他的可能就忽略了
{: .prompt-info }


## 所有权相关

### 1 能通过编译


```rust
fn main() {
    let opt = Some(String::from("Hello world"));

    match opt {
        Some(_) => println!("Some"),
        None => println!("None"),
    }

    println!("{opt:?}");
}
```

### 2 不能通过编译

```rust
fn main() {
    let opt = Some(String::from("Hello world"));

    match opt {
        // 这里发生了移动
        Some(s) => println!("{s}"),
        None => println!("None"),
    }
    println!("{opt:?}"); // 不能再使用 opt 
}
```

* 不能通过编译所有权情况

![image](/assets/images/rust/enum_move.JPG)

> 上边 `Some(s)` 处发生了 `move`, 而后续的打印需要 `opt` 有读权限，但是它没有，所以导致后边不能再使用 `opt` 了
{: .prompt-info }

#### 修复方案：使用引用 `match &opt`

```rust
fn main() {
    let opt = Some(String::from("Hello world"));

    // 这里使用引用解决, 不发生 move
    match &opt { 
        Some(s) => println!("{s}"),
        None => println!("None"),
    }
    println!("{opt:?}");
}
```

* 修复方案的所有权情况

![image](/assets/images/rust/enum_mv_fix.JPG)
