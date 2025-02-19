---
layout: post
title: RustGuide-09-MiniGrep
date: 2023-03-24 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 12 项目实战

做一个简单版本的 grep 工具，在指定文件中搜索出指定的文字



## 接收命令行参数

```rust
/*
    用于读取命令行参数，由标准库提供
    args 返回一个迭代器，迭代器返回一系列值，调用 collect 可将其转化为一个集合
    std::env::args()
 */
use std::env;

fn main() {

    // env::args() 产生一个迭代器
    // collect 产生一个集合
    /*
        env::args() 无法处理非法的 unicode 字符，
        如果有非法的 unicode 字符，env::args() 就会 panic

        如果想处理非法的 unicode 字符，那么可以用 env::args_os()
        env::args_os() 返回 OsString
     */
    let args: Vec<String> = env::args().collect();
    println!("{:?}", args);

}
```

```
$ cargo run 123 abc

// 输出 ["target/debug/minigrep", "123", "abc"]
// "target/debug/minigrep" 永远为第一个元素，是当前二进制程序
```

> 第二个元素开始才是真正的命令行参数
{: .prompt-info }

## 读取文件

```rust
// 2. 读取文件
let contents = fs::read_to_string(filename).expect("read file failed");
println!("file contents:\n{}", contents);
```

## 重构：改进模块和错误处理

### 二进制程序关注点分离指导性原则
* 3 个原则
1. 将程序拆分为 main.rs 和 lib.rs，将真正的业务逻辑放到 lib.rs 中
2. 当命令行解析逻辑较少时，将它放在 main.rs 里也行
3. 当命令行解析逻辑复杂时，需要将它从 main.rs 提取到 lib.rs

* 经过上述原则后，留在 main.rs 的功能有
1. 使用参数值调用命令行解析逻辑
2. 进行其他配置
3. 调用 lib.rs 中的 run 函数
4. 处理 run 函数可能出现的错误

* 重构模块化、错误处理

```rust
// lib.rs

/*
    用于读取命令行参数，由标准库提供
    args 返回一个迭代器，迭代器返回一系列值，调用 collect 可将其转化为一个集合
    std::env::args()
 */
// use std::env;

// // 导入处理文件相关事务
// use std::fs;
// use std::process;
// use std:: error::Error;

use  std::{ env, fs, process, error::Error };

pub struct Config {
    pub query: String,
    pub filename: String,
}

impl Config {
    // 参数为 vec 的切片
    // 用户使用上的错误，我们可以考虑使用返回 Result<T,E>
    // 而程序上的错误可以考虑使用 panic
    pub fn new(args: &[String]) -> Result<Config, &'static str>  {
        if args.len() < 3 {
            return Err("not enough arguments");
        }
        // 使用 clone() 将 &str 转为 String
        let query = args[1].clone();
        let filename = args[2].clone();
        Ok(Config { query , filename })
    }
}


// Box<dyn Error> 理解为实现了 Error 这个 trait 的类型
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    // 2. 读取文件
    //let contents = fs::read_to_string(config.filename).expect("read file failed");
    
    // ? 不会发生 panic，如果发生错误会把错误 return 给调用者
    let contents = fs::read_to_string(config.filename)?;
    println!("file contents:\n{}", contents);
    // 如果没有问题，我们返回 Ok
    Ok(())
}


// main.rs

/*
    用于读取命令行参数，由标准库提供
    args 返回一个迭代器，迭代器返回一系列值，调用 collect 可将其转化为一个集合
    std::env::args()
 */
// use std::env;

// // 导入处理文件相关事务
// use std::fs;
// use std::process;
// use std:: error::Error;

use  std::{ env, fs, process, error::Error };

use minigrep::Config;

fn main() {

    // 1. 接收命令行参数
    // 产生一个集合
    /*
        env::args() 无法处理非法的 unicode 字符，
        如果有非法的 unicode 字符，env::args() 就会 panic

        如果想处理非法的 unicode 字符，那么可以用 env::args_os()
        env::args_os() 返回 OsString
     */
    let args: Vec<String> = env::args().collect();
    println!("{:?}", args);

    // let query = &args[1];
    // let filename = &args[2];

    /*
        如果 new 返回的是 Ok，unwrap_or_else 会将 Ok 的值取出并返回
        如果 new 返回的是 Err，就会调用一个闭包(匿名函数)
        unwrap 解包，提取值
     */
    let config = Config::new(&args).unwrap_or_else(|err| {
        // |err| 是闭包的参数
        println!("Problem parsing arguments: {}", err);
        /*
            调用 exit，程序会立即终止
            参数 1 即状态码
            可以使用 cargo run 试试
         */
        process::exit(1);
    });

    /*
        因为 run 函数 Ok 返回空，所以没有必要提取值
        这里我们只需要处理错误就行了
     */
    if let Err(e) = minigrep::run(config) {   // 如果 run 返回了 Err，则处理将程序退出
        println!("Application error: {}", e);
        process::exit(1);
    }

}
```


## 使用测试驱动开发方式（TDD）开发库功能

### 测试驱动开发的步骤
1. 编写一个会失败的测试，运行该测试，确保它是按照预期的原因失败
2. 编写或修改刚好足够的代码，让新测试通过
3. 重构刚刚添加或修改的代码，确保测试会始终通过
4. 返回步骤 1，继续


```rust
/*
    返回值是从 contents 里取的，所以它俩要有相同的生命周期
 */
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();

    // lines() 返回一个迭代器
   for line in contents.lines() {
       if line.contains(query) {
            results.push(line);
       }
   }
   results
}
```

* 测试模块代码

```rust
// 添加测试模块
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn case_sensitive() {
        let query = "duct";
        // \ 表示换行输入
        let contents = "\
Rust:
safe, fast, productive.
Duct tape.";
        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }
}

```

## 使用环境变量来区分大小写搜索
```rust
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
```

```shell
// 设置 CASE_INSENSITIVE 不区分大小写
➜  minigrep git:(master) ✗ CASE_INSENSITIVE=1 cargo run to poem.txt
```


## 将错误消息写入标准错误而不是标准输出

### 标准输出 VS 标准错误

#### 标准输出
* 标准输出：stdout，一般信息应该输出到标准输出中
* 使用 `println!` 将信息输出到标准输出中
* 将标准输出重定向到 `output.txt`，而没有处理标准错误，这样错误不会出现在控制台而是出现在 `output.txt`

```shell
$ cargo run > output.txt
```

#### 标准错误
* 标准错误：stderr，一般错误应该输出到标准错误中
* `eprintln!` 将信息输出到标准错误中

```shell
// 再次运行
$ cargo run > output.txt
```

> 将 `println!` 替换为 `eprintln!`。这样错误信息会输出到控制台，而 `output.txt` 里没有错误，
{: .prompt-info }

### 小结

> `println!` 内容输出到指定文件中（这里是 `output.txt`），`eprintln!` 内容输出到控制台中显示。可以使用 `println!` 将结果输出到文件，而 `eprintln!` 将 `Err` 显示在控制台。
{: .prompt-info }



## 项目最终代码

```rust
// lib
/*
    用于读取命令行参数，由标准库提供
    args 返回一个迭代器，迭代器返回一系列值，调用 collect 可将其转化为一个集合
    std::env::args()
 */
// use std::env;

// // 导入处理文件相关事务
// use std::fs;
// use std::process;
// use std:: error::Error;

use  std::{ env, fs, process, error::Error };



pub struct Config {
    pub query: String,
    pub filename: String,
    pub case_insensitive: bool,
}

impl Config {
    // 参数为 vec 的切片
    //pub fn new(args: &[String]) -> Result<Config, &'static str>  {
    // 加 mut 因为，迭代器会修改自身的状态
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


// Box<dyn Error> 理解为实现了 Error 这个 trait 的类型
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    // 2. 读取文件
    //let contents = fs::read_to_string(config.filename).expect("read file failed");
    
    // ? 不会发生 panic，如果发生错误会把错误 return 给调用者
    let contents = fs::read_to_string(config.filename)?;
    let results = if config.case_insensitive {
        search_insensitive(&config.query, &contents)
    } else {
        search(&config.query, &contents)
    };

    for line in results {
        println!("line: {}", line);
    }

     // 如果没有问题，我们返回 Ok
    Ok(())
}

/*
    返回值是从 contents 里取的，所以它俩要有相同的生命周期
 */
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    //let mut results = Vec::new();

    // lines() 返回一个迭代器
//    for line in contents.lines() {
//        if line.contains(query) {
//             results.push(line);
//        }
//    }
//    results

    contents.lines()
            .filter(|line| line.contains(query))
            .collect()
}

pub fn search_insensitive<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();

    let query = query.to_lowercase(); 
    // lines() 返回一个迭代器
   for line in contents.lines() {
       if line.to_lowercase().contains(&query) {
            results.push(line);
       }
   }
   results
}


// 添加测试模块
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn case_sensitive() {
        let query = "duct";
        // \ 表示换行输入
        let contents = "\
Rust:
safe, fast, productive.
Duct tape.";
        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }

    #[test]
    fn case_insensitive() {
        let query = "rUst";
        let contents = "\
Rust:
safe, fast, productive.
Trust me.";
        assert_eq!(vec!["Rust:", "Trust me."], search_insensitive(query, contents));
    }

}


//-------------------------------------------------------------------------------------
// main.rs
/*
    用于读取命令行参数，由标准库提供
    args 返回一个迭代器，迭代器返回一系列值，调用 collect 可将其转化为一个集合
    std::env::args()
 */
// use std::env;


// // 导入处理文件相关事务
// use std::fs;
// use std::process;
// use std:: error::Error;

use  std::{ env, fs, process, error::Error };

use minigrep::Config;

fn main() {

    // 1. 接收命令行参数
    // 产生一个集合
    /*
        env::args() 无法处理非法的 unicode 字符，
        如果有非法的 unicode 字符，env::args() 就会 panic

        如果想处理非法的 unicode 字符，那么可以用 env::args_os()
        env::args_os() 返回 OsString
     */
    //let args: Vec<String> = env::args().collect();
    let args = env::args();

    eprintln!("{:?}", args);

    // let query = &args[1];
    // let filename = &args[2];

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

    /*
        因为 run 函数 Ok 返回空，所以没有必要提取值
        这里我们只需要处理错误就行了
     */
    if let Err(e) = minigrep::run(config) {
        eprintln!("Application error: {}", e);
        process::exit(1);
    }

}

```
