---
layout: post
title: RustGuide-07-Trait
date: 2023-03-20 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 10 泛型、Trait、生命周期

## 提取函数消除重复代码

```rust
// 使用 &
fn largest(list: &[i32]) -> i32 {
    let mut largest = list[0];
    for &item in list {        // &item 类型是 i32
        if item > largest {
            largest = item;
        }
    }
    largest
}

// 使用 *
fn largest2(list: &[i32]) -> i32 {
    let mut largest = list[0];
    for item in list {          // item 类型是 &i32
        if *item > largest {
            largest = *item;
        }
    }
    largest
}
```

## 泛型
* 可以提高代码复用能力
* 泛型是具体类型或其他属性的抽象代替
1. 编写代码时，泛型是一个占位符
2. 编译器在编译时将 “占位符” 替换为具体类型
* 例如 `fn largest<T>(list: &[T]) -> T { ... }`
1. T 叫类型参数。通常很短一般是一个字母


### 函数定义中使用泛型

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list {
        if item > largest { // 这里有问题，之后讲
            largest = item;
        }
    }
    largest
}
```

### struct 中定义泛型
* 可以使用多个泛型类型参数

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn test_struct() {
    let p = Point { x: 1, y: 2};
    let p2 = Point { x: 1.0, y: 2.0};

}
```

### enum 中定义泛型
* 让枚举的变体持有泛型的数据类型

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T,E> {
    Ok(T),
    Err(E),
}
```

### 方法中定义泛型

```rust
struct Point<T> {
    x: T,
    y: T,
}

// 泛型方法
impl<T> Point<T>  {
    fn getX(&self) -> &T {
        &self.x
    }
}

// 针对特定类型的方法
// 只有 Point<i32> 有 getX1 方法，其他 Point<T> 没有 getX1 方法
impl Point<i32>  {
    fn getX1(&self) -> &i32 {
        &self.x
    }
}
```


> 把 T 放在 impl 关键字后，表示在类型 T 上实现方法，例如 `impl<T> Point<T>`
{: .prompt-info }


> struct 里的泛型参数可以和方法里的泛型参数不同
{: .prompt-info }

```rust
struct Origin<T, U> {
    x: T,
    y: U,
}

impl<T, U> Origin<T, U> {
    fn mixup<V, W>(self, other: Origin<V, W>) -> Origin<T, W> {
        Origin { x: self.x, y: other.y }
    }
}

fn test_origin() {
    let p1 = Origin{ x: 1, y: 4};
    let p2 = Origin{ x: "hello", y: "world"};
    let p3 = p1.mixup(p2);
    println!("p3.x={}, p3.y={}", p3.x, p3.y);  // p3.x=1, p3.y=world
}
```

### 泛型代码的性能
* 使用泛型的代码和使用具体类型的代码运行速度是一样的
* 单态化（monomorphization）：即在编译时将泛型替换为具体类型的过程

```rust
fn main() {
    let integer = Some(5);
    let float = Some(5.0);
}

// 单态化编译时展开
enum Option_i32 {
    Some(i32),
    None,
}
enum Option_f64 {
    Some(f64),
    None,
}

// 展开后的 main 函数
fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

## Trait
* 告诉 Rust 编译器
1. 某种类型具有哪些并且可以与其他类型共享的功能
* Trait 可以以抽象的方式定义共享的行为
* Trait bounds（约束）：泛型类型参数指定为实现了特定行为的类型，即泛型类型参数实现了某些 Trait
* Trait 与其他语言的接口（interface）有些类似，但又有区别

### 定义一个 Trait
* 把方法签名放在一起，来定义实现某种目的所必需的一组行为
* 只有方法签名，没有方法的具体实现
* Trait 里可以写多个方法，每个方法签名占一行，以 `;` 结尾
* 实现该 Trait 的类型必须提供具体方法的实现

```rust
// 定义一个 trait
pub trait Summary {
    fn summarize(&self) -> String;
}
```

### 在类型上实现 trait
* 与为类型实现方法类似

```rust
// lib.rs
// 定义一个 trait
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {

    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{} -> {}", self.username, self.content)
    }
}


// main.rs
// traits 即 Cargo.toml 里的 package name
use traits::NewsArticle;
use traits::Summary;
use traits::Tweet;

fn main() {
    println!("Hello, world!");
    let tweet = Tweet {
        username: String::from("hourse_ebook"),
        content: String::from("of course, as you probably already know, people"),
        reply: false,
        retweet: false,
    };
    println!("1 new tweet: {}", tweet.summarize());

}
```

### RustRover Tips

![image](/assets/images/rust/rover_tips.png)


### 实现 trait 的约束
* 想要在某个类型上实现某些 `Trait` 它有一些前提条件
1. 这个`类型`或这个 `Trait` 其中之一，是在本地 `crate` 里定义的, 就可以实现该 `Trait`
* 无法为外部类型来实现外部 `Trait`，例如在本地项目为标准库里 Vector 实现标准库里的 Display Trait 这是不行的
1. 这个限制即程序属性的一部分（也就是一致性）
2. 更具体的说：它叫`孤儿原则`，之所以这么命名是因为它的父类型并没有定义在当前库里。
3. 此规则可以保证其他人的代码不能破坏你写的代码，vice verse
4. 如果没有这个规则，那么两个 crate 就可以为同一类型实现同一 trait，Rust 就不知道该使用哪个实现了


> 这里在 OC 里是相当于会覆盖，OC 本质是放到数组前边了
{: .prompt-info }


> 孤儿原则要求一个 `类型`或一个 `Trait` 其中之一，是在本地 `crate` 里定义的, 就可以实现该 `Trait`
{: .prompt-info }


### 默认实现
* 通常为 trait 提供默认行为是有用的
* 如果有默认实现，类型就不用再实现对应的方法也能调用了

```rust
// 定义一个 trait
pub trait Summary {
    //fn summarize(&self) -> String;
    // 默认实现
    fn summarize(&self) -> String {
        String::from("Read more...")
    }
}
```


* Note：默认实现的方法也可以调用 trait 中的其他方法，即使这个方法没有默认实现

```rust
// 定义一个 trait
pub trait Summary {
    fn summarize_author(&self) -> String;

    //fn summarize(&self) -> String;
    // 默认实现中调用 trait 中的其他方法
    fn summarize(&self) -> String {
        format!("Read more by {}", self.summarize_author())
    }
}

pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {

    fn summarize_author(&self) -> String {
        format!("@{}", self.author)
    }

    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {

    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }

    // 注释掉，就用默认实现了
    // fn summarize(&self) -> String {
    //     format!("{} -> {}", self.username, self.content)
    // }
}
```

> 无法从方法重新实现里调用该方法的默认实现
{: .prompt-info }


### Trait 作为参数
* 1 impl trait 语法，适用于简单 case


```rust
pub fn notify(item: impl Summary) {
    println!("Breaking news {}", item.summarize());
}

// call
traits::notify(tweet);
```

* 2 trait bound 语法，适用于复杂 case
1. impl trait 其实就是 trait bound 语法的语法糖 


```rust
// trait bound 写法
pub fn notify2<T: Summary>(item: T) {
    println!("Breaking news {}", item.summarize());
}
```

### 使用 + 号指定多个 trait bound

```rust
// impl trait 写法, 使用 + 号指定多个 trait bound
pub fn notify(item: impl Summary + Display) {
    println!("Breaking news {}", item.summarize());
}

// trait bound 写法，使用 + 号指定多个 trait bound
pub fn notify2<T: Summary + Display>(item: T) {
    println!("Breaking news {}", item.summarize());
}
```

### trait bound 使用 where 语句来指定 trait 的约束
* 在方法签名的后边指定 where 语句


```rust
pub fn notify_no_where<T: Summary + Display, U: Clone + Debug>(a: T, b: U) -> String {
    format!("where {}", a.summarize())
}

pub fn notify_where<T, U>(a: T, b: U) -> String
    where T: Summary + Display, 
          U: Clone + Debug  {
    format!("where {}", a.summarize())
}
```

### 使用 trait 作为返回类型


```rust
pub fn notify_return2(flag: bool) -> impl Summary {
    // 只能返回一个实现了 Summary 的类型
    NewsArticle {
        headline: String::from("headline"),
        location: String::from("location"),
        author: String::from("author"),
        content: String::from("content"),
    }

    // 如果实现返回多个类型，会报错
    /* 
    if flag {
        Tweet {
            username: String::from("username"),
            content: String::from("of course, as you probably already know, people"),
            reply: false,
            retweet: false,
        }
    } else {
        NewsArticle {
            headline: String::from("headline"),
            location: String::from("location"),
            author: String::from("author"),
            content: String::from("content"),
        }
    }
    */
}
```

> 在使用 impl trait 返回类型时，必须确认这个方法只能返回同一种类型，返回可能的多个类型会报错
{: .prompt-info }


### 使用 trait bound 定义 largest 函数


```rust
// std::cmp::PartialOrd 在 prelude 模块里，所以不需要导入
pub fn largest<T: PartialOrd + Clone>(list: &[T]) -> T {
    // 这里报错需要加上 Copy trait，但字符串无法使用所以可以用 Clone trait
    let mut largest = list[0].clone(); 
    for item in list.iter() {
        // 只要实现了 std::cmp::PartialOrd 这个 trait，那么 T 才能使用 > 进行比较
        if item > &largest { 
            largest = item.clone();
        }
    }
    largest
}


// 或者 T 不实现 Clone trait，而是返回 T 的引用
pub fn largest_one<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0]; 
    for item in list.iter() {
        // 只要实现了 std::cmp::PartialOrd 这个 trait，那么 T 才能使用 > 进行比较
        if item > &largest { 
            largest = item;
        }
    }
    largest
}

// 调用
let words = vec![String::from("hello"), String::from("world")];
let max_word = traits::largest(&words);
println!("max_word: {}", max_word);

let max_word = traits::largest_one(&words);
println!("max_word one: {}", max_word);
```


### 使用 trait bound 有条件的实现方法
* 在使用泛型参数的 impl 块上使用 trait bound，我们可以有条件为实现了特定 trait 的类型来实现方法

```rust

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    // 每个类型都有一个叫 new 的构造方法
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

/*
    要求 T 必须实现 Display + PartialOrd 这俩 trait
    才能有 cmp_display 这个方法
 */
impl<T: Display + PartialOrd> Pair<T>  {
    fn cmp_display(&self) {
        if self.x > self.y {
            println!("largest member is x = {}", self.x);
        } else {
            println!("largest member is y = {}", self.y);
        }
    }
}
```


### blanket implementations

* 也可以为实现了其他 trait 的任意类型有条件的实现某个 trait
* 为满足 trait bound 的所有的类型上实现 trait 叫覆盖实现（blanket implementations）

* 下边代码：对所有满足实现了 fmt::Display + ?Sized 的类型 T 都实现了 ToString 这个 trait


```rust
impl<T: fmt::Display + ?Sized> ToString for T {
    // A common guideline is to not inline generic functions. However,
    // removing `#[inline]` from this method causes non-negligible regressions.
    // See <https://github.com/rust-lang/rust/pull/74852>, the last attempt
    // to try to remove it.
    #[inline]
    default fn to_string(&self) -> String {
        let mut buf = String::new();
        let mut formatter = core::fmt::Formatter::new(&mut buf);
        // Bypass format_args!() to avoid write_str with zero-length strs
        fmt::Display::fmt(self, &mut formatter)
            .expect("a Display implementation returned an error unexpectedly");
        buf
    }
}
```

## 生命周期
* Rust 里`每个引用都有自己的生命周期`
* 生命周期：即让引用保持有效的作用域
* 大多数情况下，生命周期是隐式的、可以被推断的
* 当引用的生命周期可能以不同的方式相互关联时，就必须手动显式的标注生命周期
* 生命周期存在的主要目标是避免悬垂引用（dangling reference）

### Rust 编译器的借用检查器
* 通过借用检查器，比较作用域来判断所有的借用是否合法

```rust
fn main() {
    println!("Hello, world!");

    let r;
    {
        let x = 5;
        r = &x;
    }
    println!("{}", r); // error，因为 x 已走出作用域被释放了 
    // 检查器检查到 r 的生命周期比 x 长，而 r 有指向 x 的引用所以 error
}
```

### 函数中的泛型生命周期

```rust
// 这个方法签名报错，因为没有说明 x 和 y 的生命周期
//fn large(x: &str, y: &str) -> &str {

// 加 'a 说明生命周期，表示 x、y、返回值的生命周期一样 
fn large<'a>(x: &'a str, y: &'a str) -> &'a str {

    if x.len() >= y.len() {
        x
    } else {
        y
    }
}
```


### 生命周期的标注
* 生命周期的标注不会改变引用的生命周期长度
* 如果某个函数指定了泛型生命周期参数，那么函数就可以接收带有任何生命周期的引用

> Note：生命周期的标注：用于描述多个引用的生命周期的关系，但不会影响生命周期
{: .prompt-info }


### 生命周期的标注语法
* 生命周期的参数名称
1. 必须以单引号 `'` 开头
2. 通常是全小写字母，而且非常短
3. 很多开发者都是使用 `'a`，来作为生命周期参数的名称

* 生命周期标注的位置
1. 在引用符号 `&` 的后边
2. 使用空格将引用和类型分开
3. 例：`&i32`，一个普通的引用
4. 例：`&'a i32`，带有显式生命周期的引用
5. 例：`&'a mut i32`，带有显式生命周期的可变引用


> 单个生命周期的标注本身没有意义。因为生命周期标注是描述多个引用之间的关系
{: .prompt-info }


### 函数签名中的生命周期标注
* 泛型生命周期参数声明在，函数名和参数列表之间的 `<>` 里
1. 例 `fn large<'a>(x: &'a str, y: &'a str) -> &'a str`，
2. 上例表示参数 x、y 和返回值必须拥有相同的生命周期，而这个生命周期就是 `'a`


> 上例的函数签名告诉 Rust，有一个生命周期 `'a`，其参数和返回值的存活时间必须不短于 `'a`。当我们在函数签名指明生命周期参数时，我们并没有改变传入的参数和返回值的生命周期，只是告诉检查器，一些非法的调用约束而已
{: .prompt-info }


* 如果函数被外部代码引用时，想单靠 Rust 确定参数和返回值的生命周期，这是不可能的

```rust
// Note：把参数传入函数时，被用于代替 'a 的生命周期作用域就是 x、y 所重叠的那部分作用域
// 即 x、y 生命周期比较短的那个
fn large<'a>(x: &'a str, y: &'a str) -> &'a str {

    if x.len() >= y.len() {
        x
    } else {
        y
    }
}
```

* 这段可以编译过


```rust
    let str1 = String::from("abc");
    {
        // str2 是静态的全程序作用域有效
        let str2 = "xyz";
        let result = large(str1.as_str(), str2);
        println!("{}", result);
    }
```

* 有问题的代码

```rust
    let str1 = String::from("abcd");
    let result;
    {
        let str2 = String::from("xyz");
        
        // large 返回生命周期是 str1 与 str2 较短的那个
        // 所以这里用的是 str2 的生命周期, 赋给了 result 
        // as_str() 将 String 转为 &str
        result = large(str1.as_str(), str2.as_str()); // error
    } // 这里 str2 离开了作用域，但依然在借用的状态

    // str2 必须在外部作用域结束前一直有效才行
    // result 是 str2 的生命周期，str2 已经释放了 所以有问题
    println!("{}", result);
```


> 上例中 `'a` 就是 x、y 生命周期比较小的那个。虽然 `str1` 长度比 `str2` 长，会返回给 `result`，但 Rust 编译器不管，它只看生命周期，这里生命周期短的那个是 `str2`，所以后边使用 `result` 必须保证 `str2` 依然有效才行。因为 `str2` 提前走出了作用域所以有问题。
{: .prompt-info }


### 深入理解生命周期
* 指定生命周期参数的方式取决于函数所做的事情

```rust
// 这样函数只返回 x 也就是只需要 x 的生命周期，所以 y 就不需声明生命周期了
fn large2<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

* 从函数返回引用时，返回类型的生命周期参数必须要与其中一个参数的生命周期相匹配
* 如果返回的引用没有指向任何参数，那么它只能引用函数内创建的值
1. 这时就是悬垂引用，该值在函数结束时就走出了作用域

```rust
fn large3<'a>(x: &'a str, y: &str) -> &'a str {
    let str = String::from("abc");
    str.as_str() // 这是悬垂引用，这里报错
}

// 改成返回 String 就可以，这样相当于把 str 所有权返回给调用者
fn large3_3<'a>(x: &'a str, y: &str) -> String {
    let str = String::from("abc");
    str
}
```

> 生命周期语法就是来确定关联每个参数和返回值之间的生命周期关系的
{: .prompt-info }


### struct 中的生命周期标注
* struct 里可以包括
1. 自持有的类型
2. 引用类型：需要在每个引用上添加生命周期标注


```rust
struct Expert<'a> {
    part: &'a str, // 引用类型，即 part 这个引用要比 struct Expert 实例的生命周期要长
}

fn test() {
    let novel = String::from("call me jack. Some years ago ...");
    // &str
    let first = novel.split(".").next().expect("could not find a .");
    // 因为 first 生命周期 比 i 长，所以这样编译可通过
    let i = Expert { part: first };
}
```

> struct 中的引用字段要比 struct 实例的生命周期要长
{: .prompt-info }


### 生命周期的省略

> 每个引用都有生命周期，需要为使用生命周期函数或 struct 来指定生命周期参数
{: .prompt-info }

* 在 Rust 引用分析中所编入的模式称为生命周期省略规则
1. 生命周期省略规则是为了提高开发者的开发效率，而自动推断的
2. 这些规则无需开发者遵守
3. 它们是一些特殊的情况由编译器来考虑
4. 如果你的代码符合这些情况，那么就无需显式的标注生命周期

* 但生命周期规则不会提供完整的推断
1. 如果应用这个规则后，引用依然是模糊不清的，那么会编译失败
2. 解决办法就是手动的添加生命周期标注，来表明引用之间的关系

### 输入、输出生命周期
* 如果生命周期
1. 出现在函数/方法的参数中，那么就叫输入生命周期
2. 出现在函数/方法的返回值中，那么就叫输出生命周期


### 生命周期的省略的 3 个规则
* 编译器使用 3 个规则在没有显式标注生命周期情况下，来确定引用的生命周期
1. 规则 1 应用于输入生命周期
2. 规则 2、3 应用于输出生命周期
3. 如果编译器应用完这个 3 个规则后，依然有无法确定生命周期的引用，这时就会报错
4. 这些规则不仅适用于 fn 方法的定义，也适用于 impl 块

* 规则一：每个引用类型的参数都有自己的生命周期
* 规则二：如果只有一个输入生命周期参数，那么该生命周期被赋给所有的输出生命周期参数
* 规则三（Only for method）：如果有多个输入生命周期参数，而其中一个是 &self 或 &mut self（是方法），那么 self 的生命周期会被赋给所有的输出生命周期参数

* 例子 1，假设我们是编译器对 `fn first_word(s: &str) -> &str` 这个进行推导
1. 应用规则一后，`fn first_word<'a>(s: &'a str) -> &str`
2. 应用规则二后，`fn first_word<'a>(s: &'a str) -> &'a str`
3. 因此所有引用都有了生命周期，不需要开发者自己手动设置生命周期

* 例子 2，假设我们是编译器对 `fn bigger(a: &str, b: &str) -> &str` 这个进行推导
1. 应用规则一后，`fn bigger<'a, 'b>(a: &'a str, b: &'b str) -> &str`
2. 规则二、三不适用，这时还无法推断出返回值的生命周期，无法确定所有引用的生命周期，这时就会报错
3. 当编译器应用三个规则后依然无法确定所有参数和返回值的生命周期时就会报错


### 方法中的生命周期标注
* 在 struct 上使用生命周期实现方法，语法和泛型参数的语法一样
* 在哪声明和使用生命周期参数，依赖于生命周期参数是否和字段、方法的参数或返回值有关
* struct 字段的生命周期名
1. 在 impl 的后边声明
2. 在 struct 名后边进行使用
3. 这些生命周期是 struct 的一部分

* 在 impl 块内的方法签名中
1. `引用`必须绑定于 `struct` 字段引用的生命周期，或者`引用`是独立的
2. 生命周期省略规则经常使方法中的生命周期标注不是必须的

```rust
struct Expert<'a> {
    part: &'a str, // 引用类型，即 part 这个引用要比 struct Expert 实例的生命周期要长
}

// impl<'a> 字段生命周期名字
// impl<'a> 与 Expert<'a> 不能忽略
impl<'a> Expert<'a> {
    // 根据规则一，就不用为 &self 添加生命周期了
    fn level(&self) -> i32 {
        3
    }
    // 根据规则一为参数 &self 和 announcement: &str 添加生命周期
    // 根据规则三 返回值被赋予了 self 的生命周期
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("announcement is {}", announcement);
        self.part
    }
}
```

### 静态生命周期
* `'static` 是一个特殊的生命周期，即表示整个程序的执行时间（整个程序的执行期）
* 例如所有的字符串字面量都拥有 `'static` 生命周期
* 字符串字面量是被存储在二进制程序里，所以他总是可用的

```rust
// 字符串字面量
let s: &'static str = "I'm a static value";
```

* 为引用指定 `'static` 前一定要三思
1. 要考虑这个引用是否需要在整个程序生命周期都存活
2. 因为大部分情况下，错误是尝试创建一个悬垂引用或可用生命周期不匹配，这时应该尝试去解决这个问题，而不是指定 `'static` 


### 泛型参数类型、Trait Bound、生命周期

```rust
fn longest_with_an_announcement<'a, T>
(x: &'a str, y: &'a str, ann: T) -> &'a str 
where T: Display {
    println!("{}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
