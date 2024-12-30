---
layout: post
title: RustGuide-06-Error
date: 2023-03-19 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 9 错误处理

### panic! 宏，不可恢复的错误
* 错误分类
1. 可恢复的错误，比如文件未找到，可再次尝试
2. 不可恢复的错误，就是 bug，比如索引超出了范围

* Rust 里没有异常机制
1. 针对可恢复错误，Rust 提供了 `Result<T,E>` 类型
2. 针对不可恢复错误，Rust 提供了 panic! 宏

### 不可恢复错误与 panic! 宏
* 当 panic! 宏执行时
1. 你的程序会打印一个错误信息
2. 展开、清理调用栈
3. 最后退出程序

* 为了应对 panic 
1. 可以展开
2. 或终止（abort）调用栈

* 默认情况下，当 panic 发生时，有两种操作可选择
1. 程序会展开调用栈（工作量大），这意味着程序会沿着调用栈往回走，在往回走的过程中，每遇到一个函数就会把这个函数中的数据清理掉
2. 或理解终止调用栈，这会不进行清理工作，直接终止程序，所使用的内存需要由操作系统来清理


* 如果你想让你的二进制文件更小，我们就可以把 “展开” 改为 “终止”
1. 在 Cargo.toml 文件中适当的 profile 部分设置 panic = "abort"


```
[profile.release]
panic = "abort"
```


```rust
    let list = vec![1,2,3];
    list[9]; // 数组越界系统会调用 panic!
```

* panic! 可能调用在我们的代码中，也可能调用在依赖库的代码中
1. 通过 panic! 回溯信息来定位引起问题的代码
2. 设置 RUST_BACKTRACE=1 可以获得回溯信息

```shell
// 执行这个命令查看调试信息
$ RUST_BACKTRACE=1 cargo run

// 执行这个命令查看更丰富的调试信息
$ RUST_BACKTRACE=full cargo run
```

> 为了获取有调试信息的回溯，必须 cargo run 时 不能带 `--release`
{: .prompt-info }

## Result 枚举与可恢复的错误

* `Result<T,E>`


```rust
enum Result<T,E> {
  Ok(T),  // T 即操作成功情况下，Ok 变体里返回的数据类型
  Err(E), // E 即操作失败情况下，Err 变体里返回的错误类型
}

```


### 使用 Match 表达式处理 Result
* 和 Option 枚举一样，Result 及其变体也是由 prelude 模块导入的

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => {
            panic!("open hello txt failed, {:#?}", error);
        }
    };  // match 表达式作为语句必须以 ; 结尾

}
```

* 匹配不同的错误

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            // 创建文件也可能导致 panic，所以这里也 match 下
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("error create file, {:?}", e),
            },
            // other_error 是自己起的名字
            other_error => panic!("error open file failed, {:#?}", other_error),
        },
    };
}
```

* `Result<T,E>` 由很多方法
1. 这些方法接收闭包作为参数
2. 这些方法都是使用 match 来实现的

```rust
// 这样写等于上边的多 match 写法
    let f = File::open("hello.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound {
            File::create("hello.txt").unwrap_or_else(|e| {
                panic!("error create file, {:?}", e)
            })
        } else {
            panic!("error open file failed, {:#?}", error);
        }
    });
```

### 出现错误时使程序产生 `panic` 的常用快捷方式


#### 1. unwrap 方法
* 是 match 的一个快捷方法。用于提取 `Option` 或 `Result` 类型内部的值
  - 如果 Result 的结果是 Ok，则返回 Ok 里的值
  - 如果 Result 的结果是 Err，则调研 panic! 这个宏
* 缺点是错误信息不能自定义

```rust
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => {
            panic!("open hello txt failed, {:#?}", error);
        }
    };

    // 这么写类似上边 match
    let f = File::open("hello.txt").unwrap();
```

#### 2. expect 方法
* 与 unwrap 方法类似，但是可以指定错误信息

```rust
    let f = File::open("hello.txt").expect("文件不存在");
```

### 传播错误
* 将错误返回给调用者

```rust
fn main() {
    let name = read_username_from_file();
    match name {
        Ok(s) => println!("name = {}", s),
        Err(e) => println!("e is {}", e),
    };
}

fn read_username_from_file() -> Result<String, io::Error> {
    // 尝试打开文件，返回 Result<File, Error> 类型
    let f = File::open("hello.txt");

    // 对 Result<File, Error> 进行类型匹配
    let mut f = match f {
        Ok(file) => file, // 如果返回 File 将其赋值给 f
        Err(err) => return Err(err), // 否则此方法返回 Error
    };

    let mut s = String::new();

    // 把文件内容读取到 s 字符串
    // 这个 match 表达式最后没有 ; ，即它是这个函数最后一个表达式，即函数结果
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s), // 如果成功将 s 字符串返回
        Err(err) => Err(err), // read_to_string 失败返回 err        
    }
}
```

### ? 运算符
1. 它是传播错误的快捷方式，它与 `match` 作用是类似的


```rust
fn read_username_from_file_fast() -> Result<String, io::Error> {

    let mut f = File::open("hello.txt")?;
    // 加问号等同于与下边 match
    // let mut f = match f {
    //     Ok(file) => file, // 如果返回 File 将其赋值给 f
    //     Err(err) => return Err(err), // 否则方法返回 Error
    // };

    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

> 把 ? 作用于 Result。如果结果是 Ok，Ok 中的值就是表达式的结果，然后继续执行。如果结果是 Err，则 Err 就是整个函数的返回值，就像使用了 return
{: .prompt-info }


### ? 与 from 函数
* from 来自与标准库里 `std::convert::From` 函数
* from 的作用就是在错误之间进行转换
* 被 ? 所应用的错误，会隐式的被 from 函数处理
* 当 ? 调用 from 函数时
1. 它所接收的错误类型会被转换为当前函数的返回类型所定义的错误类型，下边例子返回的是 `io::Error`
2. 如果发生错误，? 会调用 from 函数，from 函数会将不是 `io::Error` 类型的错误转为 `io::Error` 类型（因为这里方法返回的是 `io::Error`）。但如果 ErrA 转为 ErrB，需要 ErrA 实现 from -> ErrB 函数才行。
3. 这常用于：针对不同的错误原因，返回同一种错误类型。只要每个错误类型实现了转换为所返回的错误类型的 from 函数就可以


```rust
fn read_username_from_file_fast() -> Result<String, io::Error> {
    // 如果发生错误，? 会调用 from 函数，from 函数会将错误转为 io::Error 类型
    // 但如果 ErrA 转为 ErrB，需要 ErrA 实现 from -> ErrB 函数才行
    // 这常用于：针对不同的错误原因，返回同一种错误类型
    
    let mut f = File::open("hello.txt")?;
    // 加问号等同于与下边 match
    // let mut f = match f {
    //     Ok(file) => file, // 如果返回 File 将其赋值给 f
    //     Err(err) => return Err(err), // 否则方法返回 Error
    // };

    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```


### ? 链式调用，继续优化上边的功能


```rust
fn read_username_from_file_chain() -> Result<String, io::Error> {
    let mut s = String::new();
    // 这句意思是，open 和 read_to_string 都成功则函数继续执行
    // 否则哪句失败了，哪句的 Err 就作为函数的结果 return 了
    File::open("hello.txt")?.read_to_string(&mut s)?;
    Ok(s)
}
```


> ? 运算符只能用于返回 Result 的函数
{: .prompt-info }


### 自定义错误例子，实现 `From Trait`

```rust
use std::fs::File;
use std::io::{self, ErrorKind, Error, Read};
use std::num::ParseIntError;

#[derive(Debug)]
pub enum MyError {
    Io(io::Error),
    ParseInt(ParseIntError),
    Other(String),
}

// io::Error 可以通过 ? 转化为 MyError
impl From<io::Error> for MyError {
    fn from(err: io::Error) -> Self {
        MyError::Io(err)
    }
}

impl From<ParseIntError> for MyError {
    fn from(err: ParseIntError) -> Self {
        MyError::ParseInt(err)
    }
}

fn read_username_from_file2() -> Result<String, MyError> {
    let mut name = String::new();
    let file = File::open("hello.txt")?.read_to_string(&mut name)?;
    let num = "55".parse::<i32>()?; // 通过 From<ParseIntError> for MyError 进行转化
    Ok(name)
}
```

### 什么时候可以使用 ? 运算符

> 函数的返回类型与 ? 运算符所作用的值的类型兼容。? 可用于返回类型为 `Result`, `Option` 或实现了 `FromResidual` 的类型的函数内
{: .prompt-info }


### ? 运算符作用与 `Option` 的情况

```rust
fn last_char_of_first_line(text: &str) -> Option<char> {
    // next()? 使用 ? 运算符
    text.lines().next()?.chars().last()
}
```


### ? 运算符与 main 函数
* main 函数的默认返回类型是 `()`，即空元组，什么也不返回
* main 函数的类型也可以是 `Result<T,E>`
* `Box<dyn Error>` 是一个 trait 对象，简单理解即任何可能的错误类型


```rust
fn main() -> Result<(), Box<dyn Error>> {
    let f = File::open("hello.txt")?;
    Ok(()) // 如果 main 函数以这 Ok 结束运行，程序以 0 退出。否则以其他值退出 1 或 -1 等
}
```

> main 函数可返回任何实现了 `std::process::Termination` 这个 Trait 的类型。`Termination` 定义了一个 `report` 函数，这个函数它返回 就是 `ExitCode`
{: .prompt-info }



## 何时应该使用 panic
* 调用 panic 宏就相当于发生一个不可恢复的错误
* 如果使用了 Result 就相当于对错误进行传播，而且这类错误是可恢复的
* 如果你认为自己可以替代调用者处理为一个不可恢复的错误就使用 panic，否则返回 Result 让调用者处理


> 在定义一个可能失败的函数时，优先考虑返回 Result，否则 panic
{: .prompt-info }


### 哪些场景适合 panic
1. 演示某些概念时，unwrap
2. 编写原型代码时：unwrap、expect
3. 测试代码：unwrap、expect

### 你可以确定 Result 就是 Ok 的，那么就可以使用 unwrap

```rust
// 这里一定不会出错，我们就使用 unwrap
let home: IpAddr = "127.0.0.1".parse().unwrap();
```

### 错误处理的指导性建议
1 当你的代码有可能处于损坏状态时，最好使用 panic

2 损坏状态（Bad State）：即某些假设、保证、约定、或不可变性被打破
* 例如非法的值、矛盾的值、或空缺的值被传入代码
* 以及下列中的一条：
1. 这种损坏状态并不是预期能偶尔发生的。
2. 在此之后，你的代码如果处于这种状态就无法运行。
3. 在你使用的类型中，没有一个好的方法来将这些信息（处于损坏状态）进行编码

### 场景建议
1. 别人调用你的代码，传入了无意义的参数，可以使用 panic
2. 你调用外部不可控代码，他们返回了非法状态，你无法修复，可以使用 panic
3. 如果失败是可预期的，这时最好返回 Result
4. 当你代码对某些值进行操作，首先应该验证这些值的合法性，如果不合法可以使用 panic


> 可参考 Rust Book：Error Handling - To panic! or not to panic!
{: .prompt-info }

### 为验证创建自定义类型
* 创建新类型，把验证逻辑放在构造实例的函数里

```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value need be between 1 and 100, got {}", value);
        }
        Guess { value }
    }

    pub fn value(&self) -> i32 {
        self.value
    }
}

fn test_guess() {
    loop {
        let guess = "32";
        let guess: i32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        let guess = Guess::new(guess);
    }
}
```
