---
layout: post
title: RustGuide-10-Closure
date: 2023-03-26 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 13 闭包

## 函数式语言特性：闭包（closure）
* 闭包：就是可以捕获其所在环境的匿名函数
1. 闭包是一个匿名函数
2. 保存为变量或作为参数传给另一个函数或作为函数的返回值
3. 可以在某一个地方创建闭包，然后在另一个上下文中调用闭包来完成运算
4. 可以从其定义的作用域内捕获值

### 例子 生成自定义运行计划程序
* 目标：不让用户发生不必要的等待
1. 仅在必要时调用该算法
2. 只调用一次


* 定义闭包
```rust
    // 定义闭包，num 为参数，将闭包赋值给 closure
    let closure = |num: u32| -> u32 {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };
```

## 闭包的类型推断
* 闭包不强制要求标注参数和返回值的类型
* 闭包通常很短小，通常只在狭小的上下文中工作，编译器通常能推断出闭包的参数和返回值类型
* 也可以手动添加类型标注

### 函数与闭包的语法比较

```rust
// 定义方法
fn add_one_v1(x: u32) -> u32 {
    x + 1
}

// 定义闭包
let add_one_v2 = |x: u32| -> u32 {
    x + 1
};

let add_one_v3 = |x| {
    x + 1
};

// 由于函数体只有一行，因此 {} 可省略
let add_one_v4 = |x| x + 1;
```

* 闭包的类型推断

```rust
    // 在未使用闭包时，因为无法推断其类型，所以报错
    let example_closure = |x| x;
    // 一旦使用了，就能推断出 x 的类型了，就不报错了
    let s = example_closure(String::from("hello"));
    // 一旦类型确定了，就不能再改了, 这里就不能传 5 了
    let s1 = example_closure(5);
```

> 闭包的定义最终只会为参数/返回值推断出唯一的具体的类型。可以通过 `()` 调用闭包，像上边的 `example_closure(5)`。
{: .prompt-info }


## 闭包使用泛型参数和 `fn trait` 来存储闭包
### 如何解决多次调用闭包？
* 创建一个 `struct`，它持有闭包及其调用结果
1. 只在需要结果时才执行该闭包
2. 执行完就把结果缓存了
* 这种模式通常叫记忆化（memorization）或延迟计算（lazy evaluation）
* 如何让 `struct` 持有闭包
1. `struct` 的定义需要知道所有字段的类型，需要指明闭包的类型

> 每个闭包实例都有自己唯一的匿名类型，即使两个闭包签名完全一样
{: .prompt-info }

* 所以需要使用泛型和 `Trait Bound`

### Fn Trait
* Fn Trait 由标准库提供
* 所有的闭包都至少实现了以下 trait 之一
1. Fn
2. FnMut
3. FnOnce


```rust
use std::thread;
use std::time::Duration;
use core::cmp::PartialEq;

/*
    T 就是闭包的类型，使用 Fn trait
    它接收 u32 类型，返回一个 u32 类型
    闭包在运行前 value 的值是 None
    闭包运行后，会把值存放在 value 中
 */
struct Cacher<T> where T: Fn(u32) -> u32 {
    calculation: T,
    value: Option<u32>,
}

impl<T> Cacher<T> where T: Fn(u32) -> u32 {
    fn new(calculation: T) -> Cacher<T> {
        Cacher { calculation, value: None }
    }

    fn value(&mut self, arg: u32) -> u32 {
        match self.value {
            Some(v) => v,
            None => {
                // 调用闭包 参数为 arg
                let v = (self.calculation)(arg);
                self.value = Some(v);
                v
            }
        }
    }
}

fn generate_workout(intensity: u32, random_number: u32) {
    let mut cacher = Cacher::new(|num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    });

    if intensity < 25 {
        println!("Today, do {} pushups!", cacher.value(intensity));
        println!("Next, do {} situps!", cacher.value(intensity));
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!("Today, run for {} minutes!", cacher.value(intensity));
        }
    }
}
```

### 使用 Cacher 实现的限制
* 针对不同参数 arg，value 方法总返回同样的值
1. 可进行测试是不通过的，因为闭包只执行一次

```rust
#[cfg(test)]
mod tests {

    #[test]
    fn call_diff_value() {
        let mut c = super::Cacher::new(|x| x);
        let v1 = c.value(1);
        let v2 = c.value(2);
        assert_eq!(v2, 2);
    }
}
```

* 使用 HashMap 代替单个值
1. key 就是 arg 参数
2. value 就是执行闭包的结果
* 另一个限制是只能接收一个 u32 类型参数和 u32 类型的返回值，可通过引入多个泛型参数进行解决


## 使用闭包捕获上下文环境
* 闭包可以捕获他们所在的环境

> 闭包可以访问定义它的作用域内的变量，而普通函数不行。捕获环境会产生内存开销。
{: .prompt-info }


### 捕获值的方式(与函数获得参数的方式一样，3 种)
#### 1 取得所有权，对应 `FnOnce trait`:
* 捕获时将变量的所有权 `move` 到自己的作用域内。`适用于只能被调用一次的闭包`
* 所有闭包都至少实现了 `FnOnce`，因为所有闭包都可至少调用一次
* 适用 case：如果一个闭包将捕获的值移出其主体（例如，通过将值返回或传递给其他函数），那么它只能实现 `FnOnce`，
因为一旦值被移出，闭包就不能再次调用


```rust
fn main() {
    println!("capture_three begin");

    let mut x = vec![1, 2, 3];
    println!("Before defining closure: {x:?}");

    // 新线程获得了 x 的所有权
    thread::spawn(move || println!("From thread: {x:?}"))
        .join()
        .unwrap();
}
```

#### 2 可变借用，对应 `FnMut`:
* 捕获可变借用，并对其值进行修改
* 适用于不会移出捕获的值，但是可能会修改捕获的值的闭包
* 可以被多次调用

```rust
fn main() {

    let mut x = vec![1, 2, 3];
    println!("Before defining closure: {x:?}");

    let mut borrows_mutablely = || x.push(5);
    
    // println!("Before calling closure x is: {x:?}");  不能打印，因此此时有一个可变的引用
    borrows_mutablely();
    println!("After calling closure x is: {x:?}"); // [1, 2, 3, 5]
}
```

#### 3 不可变借用，对应 `Fn`:
* 捕获不可变借用, 适用于既不移出捕获的值，也不修改捕获的值的闭包
* 也适用于完全不捕获值的闭包
* 这种闭包可以被多次调用并且不会修改其环境
* 特别适用于需要并发调用闭包的场景


```rust
fn main() {
    let mut x = vec![1, 2, 3];
    println!("Before defining closure: {x:?}");

    let only_borrows = || println!("From closure: {x:?}");
    println!("Before calling closure x is: {x:?}");
    only_borrows();
    println!("After calling closure x is: {x:?}");
}
```

* 闭包必须命名捕获的生命周期
    - 我们需要告诉 Rust，从 `make_a_cloner` 返回的闭包的生命周期不能比 `s_ref` 更长
    
    
```rust
// fn make_a_cloner<'a>(s_ref: &'a str) -> impl Fn() -> String + 'a {

// '_ 表示返回的闭包依赖于某个生命周期
fn make_a_cloner(s_ref: &str) -> impl Fn() -> String + '_ { 

    move || s_ref.to_string()
}
```


* 在创建闭包时，通过闭包对环境值的使用，Rust 推断出具体使用哪个 trait
1. 所有闭包都实现了 `FnOnce`，因为所有闭包都可至少调用一次
2. 而那些不需要移动被捕获的变量的闭包实现了 `FnMut`
3. 那些不需要对捕获变量进行修改的闭包实现了 `Fn`

 
* 闭包体可以对捕获的值进行的操作
1. 将捕获的值移出闭包
2. 修改捕获的值
3. 即不移动，也不修改值
4. 完全不从环境中捕获值


> 3 个 `trait` 有层级关系，所有实现了 `Fn` 的闭包都实现了 `FnMut`，而所有实现了 `FnMut` 的闭包都实现了 `FnOnce`
{: .prompt-info }

## Move 关键字
* 在参数列表前使用 `move` 关键字，可以强制闭包取得它所使用的环境值的所有权
1. 用于当将闭包传递给新线程以移动数据使其归新线程所有时，此技术最为有用

```rust
    let x = vec![1, 2, 3];
    /*
        声明闭包，使用 move 关键字
        将 x 的所有权移动到了闭包里
     */
    let equal_to_x = move |z| z == x;
    // Note：此处打印不能访问 x，因为 x 的所有权已经移动到了闭包中
    println!("can not use x here, because x did move: {:?}", x);
    let y = vec![1, 2, 3];
    assert!(equal_to_x(y));
```

* 使用 `mut` 的例子


```rust
fn capture2() {
    let mut x = vec![1, 2, 3];
    /*
        声明闭包，使用 move 关键字
        将 x 的所有权移动到了闭包里
     */
    let mut equal_to_x = move |z| -> bool {
        x.push(4);
        println!("test x is {:?}", x);

        x == z
    };
    let y = vec![1, 2, 3];
    assert!(equal_to_x(y) == false);
}
```

> 最佳实践：当指定 `Fn trait bound 之一时`，首先用 `Fn`，基于闭包体里的情况，如果需要 `FnOnce 或 FnMut`，编译器会再告诉你
{: .prompt-info }


# 13.2 迭代器
* 迭代器模式：允许你依次为一系列项里元素执行某些任务
* 在这个过程中，迭代器负责，遍历每个元素，并且确定这个序列何时完成
* Rust 里的迭代器
1. 它是惰性的：除非你调用消费这个迭代器的方法，否则迭代器本身没有任何效果。
2. 可以理解为这个迭代器如果不用它就什么也没干，而当你使用了某些可以消耗迭代器方法的时候，这时迭代器才起到其作用

```rust
    let v1 = vec![1, 2, 3];
    // 这返回一个迭代器，没有使用所以没有任何效果
    let v1_iter = v1.iter();

    // 使用迭代器
    // for 取得了 v1_iter 的所有权，在其内部已经将其变为 mut
    for val in v1_iter {
        println!("Got: {}", val);
    }
    // 这里不能打印，因为已经没有了所有权
    //println!(": {:#?}", v1_iter);
}
```

## Iterator trait 和 next 方法
* 所有的迭代器都实现了 `Iterator trait`
* `Iterator trait` 定义于标准库

```rust
// 大致是这样的
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

> `type Item` 和 `Self::Item` 定义了与此该 `trait` 关联的类型。实现 `Iterator trait` 需要你定义一个 `Item` 类型，它用于 `next` 方法的返回类型（即迭代器的返回元素的类型）
{: .prompt-info }


### Iterator trait
* `Iterator trait` 仅要求实现一个方法，就是 `next` 
* `next`
1. 每次调用都返回迭代器中的一项（一个元素）
2. 返回结果包裹在 Some(v) 里
3. 迭代结束，返回 None

* 可直接在迭代器上调用 `next` 方法

```rust
#[cfg(test)]

mod tests {

    #[test]
    fn iterator_demonstration() {
        let v1 = vec![1, 2, 3];
        // 这里需要加 mut，因为调用 next 方法会改变迭代器里记录序列的变量
        let mut v1_iter = v1.iter();

        assert_eq!(v1_iter.next(), Some(&1));
        assert_eq!(v1_iter.next(), Some(&2));
        assert_eq!(v1_iter.next(), Some(&3));
    }

}
```

### Note：几个迭代方法
* `iter()` 返回不可变引用的迭代器，这时调用 next 方法返回的值是指向集合中的不可变引用，即元素的不可变引用
* `into_iter` 方法：创建的迭代器会获得所有权，会把元素 `move` 到新的作用域内，并取得它的所有权
* `iter_mut` 方法：就是在遍历元素时使用的是可变的引用


## 消耗迭代器的方法
* 在标准库中 `Iterator trait` 有一些默认实现的方法。有一些方法会调用 `next` 方法，所以手动实现 `Iterator trait` 时就必须实现 `next` 方法，调用 `next` 方法的方法叫`消耗型适配器`。因为调用他们会把迭代器耗尽
* 例如，`sum` 方法就会耗尽迭代器，取得迭代器的所有权，`sum` 通过反复调用 `next` 来遍历所有的元素，每次调用就会把当前元素添加到一个总和里，迭代结束，返回总和

```rust
    #[test]
    fn iter_sum() {
        let v1 = vec![1, 2, 3];
        let v1_iter = v1.iter();
        let v2 = v1_iter.sum();
        assert_eq!(6, v2);
    }
```

## 产生其他迭代器的方法
* 在 `Iterator trait` 上还定义了另一种方法叫`迭代器适配器`
1. 不会消耗原始迭代器
2. 通过修改原始迭代器的某些方面来产生新的迭代器
* 可以通过链式调用使用多个迭代器适配器来执行复杂操作，这种调用可读性较高
* 例如：`map` 就是一个 `迭代器适配器`，它接收一个闭包作为参数，闭包作用与迭代器的每个元素，产生新的迭代器
* 例如：`collect()` 方法就是一个消耗型适配器，把结果收集到一个集合类型中


```rust
    #[test]
    fn iter_sum_2() {
        let v1 = vec![1, 2, 3];

        // v1.iter().map(|x| x + 1) 并不会对每个元素进行加 1，因为没有消耗迭代器
        // v1.iter().map(|x| x + 1);
        // 调用 collect() 进行消耗性的操作

        // Vec<_> 由编译器自行推断
        let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();
        assert_eq!(v2, vec![2, 3, 4]);
    }
```

## 迭代器：使用闭包捕获环境
* `filter` 方法，接收一个闭包，这个闭包在遍历迭代器的每个元素时，返回 `bool` 类型
* 如果返回 true，当前元素会包含在 `filter` 产生的新迭代器中，否则返回 `false` 则不会


```rust
#[derive(PartialEq, Debug)]
struct Shoe {
    size: u32,
    style: String,
}

fn shoe_in_my_size(shoes: Vec<Shoe>, size: u32) -> Vec<Shoe> {
    // 捕获 size 
    shoes.into_iter().filter(|shoe| shoe.size == size).collect()
}


#[cfg(test)]

mod tests {
    use crate::{Shoe, shoe_in_my_size};

    #[test]
    fn iter_filter_my_shoes() {
        let shoes = vec![
            Shoe { size: 10, style: String::from("sneaker")},
            Shoe { size: 12, style: String::from("sandal")},
            Shoe { size: 10, style: String::from("boot")},
        ];

        let my_shoes = shoe_in_my_size(shoes, 10);

        assert_eq!(my_shoes, vec![
            Shoe { size: 10, style: String::from("sneaker")},
            Shoe { size: 10, style: String::from("boot")},
        ]);
    }
}
```

## 使用 `Iterator trait` 创建自定义的迭代器
* 即你需要提供一个 next 方法的实现

```rust
// 这是一个从 1 遍历到 5 的计数器
struct Counter {
    count: u32,
} 

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

    #[test]
    fn using_iterator_methods() {
        /*
            zip 即把两个迭代器合并在一起，里边的元素分别来自那两个迭代器，相当于生成一个元组
            Counter::new().skip(1) 略过第一个元素，即从 2 遍历到 5
         */
        let sum: u32 = Counter::new()
            .zip(Counter::new().skip(1))
            .map(|(a, b)| a * b)     // 两个迭代器元素相乘
            .filter(|x| x % 3 == 0)  // 过滤相乘后能被 3 整除的元素
            .sum();
        /*
           map:
           1 * 2 = 2
           2 * 3 = 6
           3 * 4 = 12
           4 * 5 = 20
           filter:
           6 + 12 = 18
        */
        assert_eq!(18, sum);
    }

    #[test]
    fn calling_next_for_counter() {
        let mut counter = Counter::new();
        assert_eq!(counter.next(), Some(1));
        assert_eq!(counter.next(), Some(2));
        assert_eq!(counter.next(), Some(3));
        assert_eq!(counter.next(), Some(4));
        assert_eq!(counter.next(), Some(5));
        assert_eq!(counter.next(), None);
    }
```

## 使用迭代器和闭包改进 12 章的项目

```rust
// lib.rs
impl Config {
    // 参数为 vec 的切片
    // pub fn new(args: &[String]) -> Result<Config, &'static str>  {
    // std::env::Args 就是一个迭代器，加 mut 因为，迭代器会修改自身的状态
    pub fn new(mut args: std::env::Args) -> Result<Config, &'static str>  {

        if args.len() < 3 {
            return Err("not enough arguments");
        }

        // 优化点
        // 跳过第一个元素，因为第一个元素没有用
        args.next();
        // 使用 clone() 将 &str 转为 String
        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };
        let filename = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file string"),
        };

        // 
        // 使用 std::env 模块
        // 只要 CASE_INSENSITIVE 变量出现 就是不区分大小写的 
        /*
            insensitive 取自环境变量，使用 std::env 模块 
            只要 CASE_INSENSITIVE 变量出现 就是不区分大小写的
            var 返回 Result 对象
            如果设置了环境变量 is_ok 返回 true
            没设置环境变量就是区分大小写的
         */
        let case_insensitive = env::var("CASE_INSENSITIVE").is_ok();
        println!("case_insensitive: {}", case_insensitive);
        Ok(Config { query , filename, case_insensitive })
    }
}

pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents.lines()
            .filter(|line| line.contains(query))
            .collect()
}


//main.rs
    let args = env::args();

    eprintln!("{:?}", args);

    /*
        如果 new 返回的是 Ok，unwrap_or_else 会将 Ok 的值取出并返回
        如果 new 返回的是 Err，就会调用一个闭包(匿名函数)
        unwrap 解包，提取值
     */
    let config = Config::new(args).unwrap_or_else(|err| {
        // |err| 是闭包的参数
        eprintln!("Problem parsing arguments: {}", err);
        /*
            调用 exit，程序会立即终止
            参数 1 即状态码
            可以使用 cargo run 试试
         */
        process::exit(1);
    });
```

## 性能比较：循环 VS 迭代器

* 零开销抽象（Zero-Cost Abstraction）
1. 使用抽象时不会引入额外的运行时开销
2. 迭代器使用了零开销抽象，所以不会引入额外的运行时开销

### 例子

```rust
fn main() {
    let buffer: &mut [i32];
    let coefficients: [i64; 12];
    let qlp_shift: i16;

    for i in 12..buffer.len()  {
        let prediction = coefficients.iter()
            .zip(&buffer[i - 12..i])
            .map(|(&c, &s)| c * s as i64)
            .sum::<i64>() >> qlp_shift;
        let delta = buffer[i];
        buffer[i] = prediction as i32 + delta;
    }
}
```
#### 上边代码
* 编译优化：编译后，这段代码与手写汇编代码几乎一致。由于迭代次数固定为 12 次，编译器进行循环展开，消除了循环控制的开销
* 快速访问：所有系数都存储在寄存器中，访问速度极快
* 无运行时开销：没有运行时的边界检查额外开销

> 结论：在 Rust 中可放心使用迭代器和闭包等高级特性，它们提供了更高层次的代码抽象，同时保持着极高的运行时性能，不会带来性能损失
{: .prompt-info }
