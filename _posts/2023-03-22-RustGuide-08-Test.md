---
layout: post
title: RustGuide-08-Test
date: 2023-03-22 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 11 编写自动化测试

## 编写和运行测试
* 测试：即是个函数，验证非测试代码的功能是否与预期一致

### 测试函数体内通常执行 3 个操作
1. 准备数据状态
2. 运行被测试的代码
3. Assert（断言）结果


### 测试函数使用 test 属性（attribute）进行标注
* attribute 就是一段 Rust 代码的元数据，不会改变被修饰代码的逻辑，只是对代码进行标注

> 在函数上加 `#[test]` 才可以把函数变为测试函数，不加 `#[test]` 的不是测试函数。
{: .prompt-info }



### 运行测试
* 使用 `cargo test` 命令运行所有的测试函数
1. Rust 会构建一个 Test Runner 可执行文件 
2. Test Runner 就会逐个调用标注了 test attribute 的函数，并报告其运行是否成功

* 当使用 `cargo` 创建 `Library` 项目时，会生成一个 `test module`，里面有一个现成的 `test` 函数，可以参照这个现成的 `test` 函数编写其他 `test` 函数
  * 你可以添加任意数量的 `test module` 或函数

```shell
// 创建一个库项目叫 adder
$ cargo new adder --lib

// 执行测试函数
$ cargo test

执行输出：
running 1 test                // 当前运行一个 test
test tests::it_works ... ok   // 运行 tests::it_works 函数，结果是 ok

// test result: ok 表示项目里所有的测试都通过了
// 0 measured 表示有 0 个性能测试
// 0 filtered out 指我们没有过滤掉任何测试
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

// 这是文档测试的结果，Rust 可以编译测试出现在文档中的代码，这可以保证文档能与实际代码同步
   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

```

### 测试失败
* 测试函数 `panic` 就是失败了
* 每个测试运行在一个新的线程
* 主线程会监听这些测试线程，当主线程看到某个测试线程挂掉嘞，那这个测试就失败了
* 如何让测试线程挂掉呢，就是触发 `panic`

```rust
#[cfg(test)]
mod tests {
    use super::*;  // 使用 * 将外部模块内容全部导入

    #[test] // 标记为：这是一个测试函数
    fn explorer() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
    
    #[test]
    fn make_panic() {
        panic!("make me panic");
    }
}
```

## 断言（Assert）

### 使用 `assert!` 宏检查测试结果
* `assert!` 来自标准库，用来确定某个标准库是否为 true
1. 为 true 测试就通过
2. 为 false 就会调用 `panic!` 宏，测试失败

```rust
    #[test]
    fn another() {
        let a = true;
        assert!(a);
    }
```

### 使用 `assert_eq!` 和 `assert_ne!` 测试相等性
* 都来自标准库
* `assert_eq!（equal）`，判断是否相等，实际就是使用 `==`
* `assert_ne!（not equal）`，判断是否不相等，实际就是使用 `!=`
* 如果断言失败，这俩宏会自动打印出两个参数的值，方便我们查看原因，而 `assert!` 只能指定结果，不能知道原因
1. 失败时使用 debug 格式打印参数，
2. 要求他的参数实现 `PartialEq 和 Debug 这俩 Traits`（所有的基本类型和标准库里大部分类型都实现了）
3. 针对 `struct` 和 `enum` 需要自己自行实现这个俩 `trait`

* 期待的值可以放 `assert_eq!` 或 `assert_ne!` 里的任何参数位置

```rust
fn add_two(a: i32) -> i32 {
    a + 3
}

#[test]
fn it_work_two() {
    assert_eq!(4, add_two(2));
}
```

> 错误信息：thread 'tests::it_work_two' panicked at 'assertion failed: `(left == right)` left: `4`, right: `5`', src/lib.rs:33:9
{: .prompt-info }


## 自定义的错误信息
* 可以向 `assert!`、`assert_eq!`、`assert_ne!` 三个宏添加可选的自定义信息
1. 如果添加了可选的自定义信息，那么他们会和失败信息一起被打印出来

* `assert!` 第一个参数时必填的，自定义消息作为第二个参数
* `assert_eq!`、`assert_ne!` 它们前两个参数是必填的，自定义消息作为第三个参数
* 自定义信息传入 `assert!`、`assert_eq!`、`assert_ne!` 后会被传给 `format!` 宏，可以使用 {} 占位符

```rust
fn greeting(name: &str) -> String {
    format!("Hello {}", name)
}

#[test]
fn it_work_greeting() {
    let result = greeting("Carol");
    assert!(result.contains("Carol"));
}
```


```rust
fn greeting1(name: &str) -> String {
    format!("Hello!")
}

    #[test]
    fn it_work_greeting1() {
        let result = greeting1("Carol");
        assert!(
            result.contains("Carol"), 
            "Greeting did not cntain name {}", result
        );
    }
```

* 上边代码错误信息：thread 'tests::it_work_greeting1' panicked at 'Greeting did not cntain name Hello!', src/lib.rs:53:9


## 用 should_panic 检查 panic
* 测试函数除了检查代码是否返回正确的值外，还需要检查代码是否如预期的处理了发生错误的情况
* 可验证代码在特定情况下是否发生了 `panic`
* 为函数添加 `shoule_panic` 属性
1. 如果标记了 `shoule_panic` 属性的函数里发生了 `panic`，那么测试通过
2. 如果没有发生 `panic`，那么测试失败

```rust
pub struct Guess {
    value: u32,
}

impl Guess {
    pub fn new(value: u32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }
        Guess { value }
    }
}

#[test]
#[should_panic]   // 标记
fn greater_than_100() {
    Guess::new(200); // 测试可以通过，因为发生了 panic
}

```

* 让 `should_panic` 更精确
1. 为 `should_panic` 属性添加一个可选的 `expected` 参数，将检查失败消息中是否包含所指定的文字，如包含 `expected` 的内容则测试通过

```rust
pub struct Guess {
    value: u32,
}

impl Guess {
    pub fn new(value: u32) -> Guess {
        if value < 1 {
            panic!("Guess value must be greater than or equal to 1, got {}.", value);

        } else if value > 100 {
             
        }
        if value < 1 || value > 100 {
            panic!("Guess value must be less than or equal to 100, got {}.", value);
        }
        Guess { value }
    }
}

    #[test]
    // 错误信息要包含 expected 的值，测试才能通过
    #[should_panic(expected = "Guess value must be less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }

```


## 在测试中使用 `Result<T, E>` 枚举
* 在编写测试函数时，无需 `panic`，可使用 `Result<T, E>` 作为返回类型
1. 返回 Ok：测试通过
2. 返回 Err：测试失败


```rust
#[test]
fn it_work_result() -> Result<(), String>{
    if 2 + 2 == 4 {
        Ok(())
    } else {
        Err(String::from("two plus two does not equal four"))
    }
}
```

> 不要在使用 `Result<T, E>` 为返回值的测试函数上使用 `should_panic` 属性，因为测试失败返回 `Err` 不会发生 `panic`
{: .prompt-info }


> 断言某个操作返回 `Err`，不要在那个 `Result` 上使用 `?`, 而应该使用 `assert!(value.is_err())`
{: .prompt-info }


## 控制测试的运行方式
* cargo test 命令，在测试模式下编译代码并生成一个二进制文件
* 可以添加命令行参数来改变 cargo test 的行为
* cargo test 的默认行为
1. 并行运行所有的测试
2. 捕获（不显示）所有的输出（在测试通过情况下），使得读取与测试结果相关的输出更加容易

* 命令行参数
1. 针对 cargo test 的参数，紧跟 cargo test 后边
2. 针对 cargo test 生成的可执行文件，放在 `--` 之后

* `cargo test --help`，显示 `cargo test` 后的参数
* `cargo test -- --help`，显示可以用在 `--` 之后的参数


### 并行运行测试
* 运行多个测试时，默认使用多个线程并行运行
* 确保测试之间
1. 不会相互依赖
2. 不依赖某个共享状态（环境、工作目录、环境变量等等）

### `--test-threads` 参数
* 传给二进制文件的，这种命令要加 `--`，例如 `cargo test -- xxx`
* 用于控制并行线程数
* 可使用 `--test-threads` 参数，后边跟线程数量
1. 例：`cargo test -- --test-threads=1`，串行执行


### 显示函数的输出
* 默认，如果测试通过，Rust 的 test 库会捕获所有打印到标准输出的内容，即不会打印
* 例如 如果被测试代码使用了 `println!`
1. 如果测试通过，那么不会在终端看到 `println!` 的打印内容
2. 如果测试失败，会看到 `println!` 打印内容和失败信息
* 使用 `cargo test -- --show-output`，让测试通过的 case 也能打印 `println!` 内容


### 按测试的名称来运行测试
* 选择运行的测试：将测试的名称（一个或多个）作为参数
* 指定单个测试命令：`cargo test [测试函数名称]`
* 指定多个测试命令
1. 指定测试名的一部分，例如 `cargo test it_work`，将执行 it_work 前缀的所有测试函数
2. 指定模块名称，例如 `cargo test tests`，将执行 tests 模块（mod）里的所有测试函数


### 忽略某些测试
* 使用 `ignore` 属性，对测试函数进行标记，将其忽略在 `cargo test` 之外
* 输出信息中可找到 `ignored` 的测试，完整信息例如：`test tests::it_work_ignore ... ignored`

```rust
    #[test]
    #[ignore]
    fn it_work_ignore() {
        assert_eq!(2, 1 + 1);
    }
```

* 执行 ignore 的测试函数


```shell
$ cargo test -- --ignored
```

* 执行全部的测试函数（包括 ignore 的）

```shell
$ cargo test -- --include-ignored
```


## 测试的组织
* Rust 对测试的分类
1. 单元测试
2. 集成测试

* 单元测试
1. 小、专注
2. 一次对一个模块进行隔离测试
3. 可以测试 `private` 的接口

* 集成测试
1. 在库的外部，和其他外部代码一样使用你的代码
2. 只能使用 `public` 接口
3. 可能在每个测试中使用到多个模块

### 单元测试
* 测试一段代码功能是否符合预期
* 一般把单元测试代码和被测试代码放在 `src` 下同一个文件中
* 同时每个源代码文件都建立一个 `test` 模块来放这些测试函数
* 使用 `#[cfg(test)]` 对测试模块进行标注
1. 标注后，只有运行 `cargo test` 才编译和运行代码。而执行 `cargo build` 时则不会

* 集成测试在不同的目录里，它不需要 `#[cfg(test)]` 标注
* cfg：就是 configuration 的缩写
1. 告诉 Rust 下面的条目只有在指定的配置选项下才被包含
2. 例如 `cfg(test)` 只在 `test` 下才执行
3. 而选项 test 是由 Rust 提供的，用来编译和运行测试


> 只有执行 `cargo test` 才会编译使用 `#[cfg(test)]` 标注的条目，条目包括测试模块中的 `helper` 函数和 `#[test]` 标注的函数
{: .prompt-info }


```rust
#[cfg(test)] // 只有运行 cargo test 会被编译运行
mod tests { 
    // 将外部模块所有内容都导入到这个模块
    use super::*;

    #[test] // 这是一个测试函数，运行 cargo test 会被编译运行
    fn explorer() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
    
    fn it_other() {  // 这不是测试函数，但运行 cargo test 也会被编译
    
    }
}
```

### 测试私有函数
* Rust 里允许测试私有函数

```rust
// lib.rs

pub fn add_two(a: i32) -> i32 {
    internal_add(a, 2)
}

// private func
fn internal_add(a: i32, b: i32) -> i32 {
    a + b
}


#[cfg(test)]
mod tests {
    // 将外部模块所有内容都导入到这个模块
    use super::*;

    #[test]
    fn it_work_internal() {
        assert_eq!(4, internal_add(2, 2));
    }
}
```

## 集成测试
* 完全位于被测试库的外部，即只能测试库公开的那些 API
* 目的：是为了测试库的多个部分是否能正确的一起工作
* 集成测试的覆盖率是一个重要指标

### tests 目录
* 首先需要创建集成测试目录：`tests` 目录，与 `src` 目录并列
* `cargo` 会自动在 `tests` 目录下找集成测试文件
* `tests` 目录下的每个测试文件都是单独的一个 `crate`
1. `tests` 目录下的代码只会在 `cargo test` 命令时才会编译
2. 将测试库导入，这里是 `use adder;`
* 无需标注 `#[cfg(test)]`，因为 `tests` 目录会被 `cargo` 进行特殊处理

```rust
// 对 lib.rs 进行测试
// lib.rs 在本模块中，本模块名为 adder
use adder;

#[test]
fn it_adds_two() {
    assert_eq!(4, adder::add_two(2));
}
```

### 运行指定的集成测试
* 运行一个特定的集成测试：`cargo test 函数名`
* 运行某个测试文件内的所有测试：`cargo test --test 文件名`

```shell
// integration_tests 是文件名
// 只运行 integration_tests 里的测试
$ cargo test --test integration_tests
```

### 集成测试中的子模块
* `tests` 目录下的每个文件都会被编译成单独的 `crate`
1. 这些文件不共享行为（与 `src` 下的文件规则不同）

* 在 `tests` 目录下创建，`common` 文件夹，在 `common` 文件夹里创建 `mod.rs` 这样 Rust 不会把其当为测试代码，`common` 相当于是一个模块，
  * 可以放一些通用函数在这里
  * 这样 `tests` 目录下的目录不会被编译为单独的 `crate`，只作为模块处理
  
  
![image](/assets/images/rust/test_com.png)


```rust
// tests/integration_test.rs
// 对 lib.rs 进行测试
// lib.rs 在本模块中，本模块名为 adder
use adder;

// 导入 common 模块
mod common;

#[test]
fn it_adds_two() {
    common::setup();
    assert_eq!(4, adder::add_two(2));
}
```

### 针对 Binary Crate 的集成测试
* 如果项目是 Binary Crate，只含有 `src/main.rs` 而没有 `src/lib.rs`
1. 这时不能在 `tests` 目录下创建集成测试
2. 即使有测试也无法把 `main.rs` 函数导入作用域

> 因为只有 `Library Crate` 才能暴露函数给其他 `Crate` 使用，`Binary Crate` 意味着要独立运行
{: .prompt-info }

* 所以一般 `Binary Crate` 会把逻辑放到 `src/lib.rs` 中，在 `src/main.rs` 中只有简单的调用，这样就可以将其变为 `Library Crate`，并通过 `use` 关键字访问里边的逻辑，只要里边逻辑代码没有问题了，那么核心功能代码就没有问题了
