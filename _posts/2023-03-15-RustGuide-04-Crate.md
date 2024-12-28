---
layout: post
title: RustGuide-04-Crate
date: 2023-03-15 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 7 Package（包）、Crate（单元包）、Module（模块）

## Rust 代码组织
* 哪些细节可以暴露，哪些细节是私有的
* 作用域内哪些是有效的等


## 模块系统
* Package（包）：cargo 特性，让你构建、测试、共享 crate
* Crate（单元包）：一个树型模块结构，它可以产生一个 library 或一个可执行文件
* Module（模块）、use：让你控制代码的组织、作用域、私有路径
* Path（路径）：为 struct、function、module 等项命名的方式

## Package 和 Crate

### Crate 类型
1. binary，二进制的，可执行的，需要有 `main 函数(为其入口点)`。一个 `Package` 可包含多个 `binary crate`
2. library 库，`不可执行`，没有 `main 函数`。一个 `Package` 可包含 `0-1 个 library crate`


### Crate Root
* 它是源代码文件
* Rust 编译器会从这里开始，是一个入口文件，可以组成你 Crate 的根 Module
* Crate 上边是 Package

### Package
* 包含一个 `Cargo.toml`， 它描述了如何构建这个 Crate
* 只能包含 0-1 个 library crate
* 可以有任意数量的 binary crate
* 必须至少包含一个 crate （binary 或 library）

### 新建一个 Package
```
$ cargo new package
```

### 新建一个 Lib 的 Package
```
$ cargo new package --lib
```

### Cargo 惯例
* src/main.rs（binary crate，是入口文件）
1. 这个文件是 binary crate 的 crate root。Cargo 默认将这个文件作为这个 crate 的根
2. crate 与 package 的名相同

* src/lib.rs（library crate，是入口文件）
1. package 包含一个 library crate
2. 这个文件是这个 library crate 的 crate root
3. 这个 library crate 的名与 package 的名相同


> Cargo 会把 crate root 交给 Rust 编译器来构建 library 或 binary
{: .prompt-info }

* 一个 Package 可以同时拥有 src/main.rs 和 src/lib.rs
1. 这表明 Package 包含一个 library crate 和一个 binary crate
2. 名称与包名相同

* 一个 Package 可以有多个 binary crate
1. 文件放在 src/bin/ 下
2. 在 src/bin/ 下每个单独的文件都是一个 binary crate


## Crate 的作用
* 可以将相关功能组合到一个作用域内，便于在项目间进行共享
1. 防止冲突

## Module
* 定义 Module 来控制作用域和私有性
* Module 在一个 Crate 里，它能够将代码进行分组
* 增加可读性，易于复用
* 控制 item 的私有性，是 public 或 private

### 创建 Module
* 使用 mod 关键字
* Module 可以嵌套
* Module 可以包含其他项（struct、enum、常量、trait、函数等）的定义

* 嵌套结构展示

```rust
// file: src/lib.rs

mod front_of_house {

    mod hosting { // 这是一个子 module
        fn add_to_waitlist() {}
        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}
        fn serve_order() {}
        fn take_payment() {}
    }

}
```


![image](/assets/images/rust/crate.png)

> `src/main.rs（这是一个 binary crate）` 和 `src/lib.rs（这是一个 library crate）` 叫做 crate roots，这俩文件都形成了 `crate 它位于模块树的根部`
{: .prompt-info }


### 自定义 Module

#### 现在考虑我们要创建一个 models 模块，models 模块下有 enums 子模块,  应该如何创建？

* rust 会从三个地方找这个 modules 模块
 1. inline 模式：在声明模块的地方（当前文件）找是否有大括号
 2. 声明到 models.rs 里
 3. 找 models/mod.rs 文件


![image](/assets/images/rust/crate_mod.png)

* 结构如上图：
  1. 创建 models.rs 文件和 models 文件夹
  2. 创建 models/enums.rs 文件


* 添加第三方库依赖: `serde`


```
$ cargo add serde
```

#### csv demo

```rust
// file: src/main.rs
use crate::models::structs::HousePrice;
use crate::models::enums::YesNo;

fn main() {
    println!("Hello, world!");
    let y = crate::m1::m2::method_1();

    let house_price = HousePrice {
        price: 1000,
        area: String::from("Center"),
        bed_rooms: 6,
        main_road: YesNo::No
    };

    // read_csv("ss")
}

mod models;

mod m1 {
    pub mod m2 {
         pub fn method_1() {
             println!("Method 1");
         }
    }
}

mod x1 {
    mod x2 {
        use crate::x1::x2;

        fn method_2() {
            super::super::m1::m2::method_1();
            println!("call m1::m2::Method 1");
        }

        fn method_3() {
            // x2::method_2();
            self::x2::method_2();
        }
    }
}

// ------ file: src/lib.rs
pub mod models;

pub mod functions;


// ------ file: src/models.rs
pub mod enums;

pub mod structs;


// ------ file: src/functions.rs
use crate::models::structs::HousePrice;
use csv::{Writer, ReaderBuilder, Reader};

pub fn read_csv(path: String) -> Vec<HousePrice> {
    let mut rdr = Reader::from_path(path).unwrap();
    vec![]
}

// ------ file: src/models/enums.rs
pub enum YesNo {
    Yes,
    No
}


// ------ file: src/models/structs.rs

use crate::models::enums::YesNo;

pub struct HousePrice {
    pub price: u32,
    pub area: String,
    pub bed_rooms: u32,
    pub main_road: YesNo,
}
```

* csv demo 目录结构

![image](/assets/images/rust/cvs.png)



## Path（路径）
* 为了在 Rust 中找到某个条目，需要使用 Path，类似命名空间
* Path 的两种形式
1. 绝对路径：从 crate root 开始，使用 crate 名或字面值 crate
2. 相对路径：从当前模块开始，`使用 self（本身），super（上一级） 或当前模块的标识符`
* Path 至少有一个标识符组成，如果有多个标识符使用两个冒号分隔，::

### 私有边界（private boundary）
* Module 不仅可以组织代码，还可以定义私有边界
* Rust 中所有条目（函数、struct、enum、方法、模块、常量）`默认都是私有的`
* 使用 pub 关键字声明公开，即 public

> 父级模块无法访问子模块中的私有条目，子模块中可以使用所有祖先模块中的条目。
{: .prompt-info }


```rust
mod front_of_house {

    pub mod hosting { // 这是一个子 module
        pub fn add_to_waitlist() {}
    }

}

pub fn eat_at_restaurant() {

    // 使用绝对路径调用
    // lib.rs 隐式组成了名为 crate 的模块
    // 因为 eat_at_restaurant 与 front_of_house 都在 crate 根级，所以不需要声明 pub 就可相互调用
    crate::front_of_house::hosting::add_to_waitlist();

    // 使用相对路径调用
    front_of_house::hosting::add_to_waitlist();

}
```

### super 关键字
* 用来访问父级模块路径中的内容，类似文件系统中的 `..`

```rust
fn serve_common_order() {}

mod front_of_house {

    fn serve_order() {
        // 使用相对路径 super 寻找调用函数
        // super 找到 front_of_house 的上级模块，调用 serve_common_order
        super::serve_common_order(); 
        
        
        // 使用绝对路径进行调用
        crate::serve_common_order();

    }

}

```

### pub struct
* 将结构体设置为 public
1. 但这样 struct 的字段默认是 private 的
2. struct 里的字段需要单独设置 pub 来变成公有


```rust
mod back_of_house {

    // 一个 public 的 struct
    pub struct Breakfast {
        pub toast: String,      // public
        seasonal_fruit: String, // private
    }

    impl Breakfast {
        // 关联函数
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast { 
                toast: String::from(toast), 
                seasonal_fruit: String::from("peaches"),
             }
        }
    }
}

pub fn eat_up() {
    let mut meal = back_of_house::Breakfast::summer("Rye");
    meal.toast = String::from("Wheat");
    println!("I'd like {} toast please", meal.toast);
    //meal.seasonal_fruit = String::from("bluecerries"); // 这里报错，因为 seasonal_fruit 是私有的
}
```

### pub enum
* 将枚举变为 public
* 枚举的值也都是 public，不需要再声明了，这里和默认私有规则不同

```rust
mod back_of_house {
    pub enum Appetizer {
        Soup,   // 默认就是 public
        Salad,  // 默认就是 public
    }
}
```

## use 关键字
* 可以使用 use 关键字将路径导入到作用域内
* 引入的东西仍然遵守私有性的规则，即只有 public 的引入后才能使用

```rust
mod front_of_house {
    pub mod hosting { // 这是一个子 module
        pub fn add_to_waitlist() {}
        fn seat_at_table() {}
    }
}


// 使用绝对路径引入模块 hosting，类似符合链接
use crate::front_of_house::hosting;

// 使用相对路径引入 hosting，因为 front_of_house 在同一级
// use front_of_house::hosting;

pub fn eat_use() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    // hosting::seat_at_table(); 报错 因为这个方法是私有的
}
```


### use 的习惯用法 
* 函数：将函数的父级模块引入作用域，这样可以知道是哪个模块的函数（指定到父级）
* struct、enum、其他：指定完整的路径（指定到本身）


```rust
// 引入时直接指定到 HashMap 本身
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);    
}
```

* 同名条目：指定到父级


```rust
use std::fmt;
use std::io;

fn f1() -> fmt::Result {

}

fn f2() -> io::Result<i32> {

}
```


## as 关键字
* 可以为引入的路径指定本地别名


```rust
use std::fmt::Result;
use std::io::Result as IOResult;  // 设置本地别名为 IOResult

fn f1() -> Result {

}

fn f2() -> IOResult<i32> {

}
```

## pub use 重新导入名称
* 使用 use 将路径（名称）导入到作用域内，该名称在此作用域内是私有的
* pub use
1. 将条目导入作用域
2. 该条目可以被外部代码引入到它们的作用域内
3. 因为模块内部结构可与外部使用者看到的结构不相同


```rust
mod front_of_house {

    pub mod hosting { // 这是一个子 module
        pub fn add_to_waitlist() {}
        fn seat_at_table() {}
    }

}


/*
使用 use 将路径（名称）导入到作用域内，该名称在此作用域内是私有的，只有本模块的代码可以使用 hosting
*/
//use crate::front_of_house::hosting;


/*
    可以使用 pub use 将其声明为 public，这样外部就可以调用了
*/
pub use crate::front_of_house::hosting;

pub fn eat_use() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    // hosting::seat_at_table(); 报错 因为这个方法是私有的
}

```


## Package 使用
* Cargo.toml 添加依赖包及其版本
1. Cargo 就会从 https://crates.io/ 下载依赖的包和其依赖项目，下载到本地
2. 然后使用 use 将特定条目引入作用域


* 引入 rand 


```
// 修改 Cargo.toml 
[dependencies]
rand = "0.5.5"
```


```
// 查看 cargo 位置
$  where cargo  

$ cd /Users/xxx/.cargo
$ ls -al // 应该能看到 .package-cache 文件
$ rm .package-cache

// 再回到工程目录
$ cargo build  // 这时应该可以下载完并编译通过了

// 更换代码源
// 如果下载太慢 可以进入 /Users/xxx/.cargo，$touch config，编辑 config 修改源代码库的源

```


```rust

use rand::Rng;

```


* 标准库（std）也被当做外部包
1. 由于 std 已经内置在语言中了，所以不需要修改 Cargo.toml 来包含 std
2. 使用 std 时，只需要使用 use 将需要的条目引入即可使用


* 使用嵌套路径清理大量的 use 语句
1. 如果使用同一个包或模块下多个条目
2. 可以使用嵌套路径在同一行内将上述条目进行引入
3. 格式：路径相同部分::{路径差异部分}


```rust
use std:: { collections::HashMap, fmt::Result, io::Result as IOResult};

// -------------------------------------------

// 如果两个路径之一是子路径使用 self
use std::io:: {self, Write}; 
// 等价于下边
use std::io;
use std::io::Write
```

* 通配符 `*`
1. 通常不这么引入，谨慎使用
2. 使用场景是测试时，将所有测试代码引入 test 模块，
- 有时被用于预导入（prelude）模块

```rust
use std::collections::*;
```

## 将模块拆分为不同文件
* 模块定义时，如果模块名后是 `;`，而不是代码块
1. 那么 Rust 就会从与模块同名的文件中加载内容
2. 模块树的结构不会发生变化




在 big_house.rs 中定义
```rust
pub mod host;
```

/big_house/host.rs

```rust
pub mod host {

    pub fn check_host() {}

}
```


lib.rs 文件中 定义 mod big_house;

```rust
// Rust 会找与 big_house 同名的文件即找 big_house.rs
mod big_house;


// 引入模块 host 
// 如果想把 host 也以 ; 形式定义为模块，需要创建 big_house 文件夹，定义 host.rs
// 第一个 host 表示 host.rs
// 第二个 host 表示 host.rs 中的 host 模块
use big_house::host::host;


// 调用 host 模块的 check_host 方法
pub fn big_house() {
    host::check_host();
}
```

目录结构

```
src-lib.rs
   -big_house/
    |--host.rs
   -big_house.rs
   -main.rs
```

* 随着模块逐渐变大，该技术让你可以把模块的内容移动到其他文件中

> 一个目录或一个文件就是一级，目录的层级结构要跟模块的层级结构相匹配。
{: .prompt-info }


# 更换 crates 源
* 如果网络不好可以换成国内源操作如下

* Step1：进入目录，创建 `config` 文件

```shell
$ cd /Users/xxxx/.cargo
$ touch config
```

* Step2：编辑 `config` 文件

```
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"

replace-with = 'tuna'
[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"


[net]
git-fetch-with-cli = true
```

* Step3：然后再重新 `cargo build` 即可

