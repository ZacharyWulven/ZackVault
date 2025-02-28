---
layout: post
title: RustGuide-11-Publish
date: 2023-03-28 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 14 cargo 和 crates.io

## 通过发布配置（release profile）来自定义构建
### 发布配置（release profile）
* 是预定义的
* 可以自定义：可使用不同配置，对代码编译拥有更多的控制
* 每个 `profile` 的配置都是独立于其他 `profile` 的

### cargo 主要有两种常见的 profile
* dev profile：适用于开发，执行 `cargo build` 命令时，执行的就是 `dev profile`
* release profile：适用于发布，执行 `cargo build --release`，执行的就是 `release profile`

### 自定义 profile
* 针对每个 `profile，cargo` 都提供了默认的配置
* 如果想自定义某个 `profile` 的配置
1. 可以在 `Cargo.toml` 里添加 `[profile.xxx]` 区域，在里边覆盖默认配置的子集
2. 不覆盖就使用默认的配置


```
// cargo.toml
// opt-level 用于 Rust 编译时对代码执行那种程度的优化，值范围是 0~3
// 数值越高优化程度越高，优化程度越高，代码编译的时间越长

// 覆盖 dev profile 的 opt-level 值
[profile.dev] 
opt-level = 1 // 0 为 dev profile 模式下的默认值


// 覆盖 release profile 的 opt-level 值
[profile.release]
opt-level = 3 // 3 为 release profile 模式下的默认值
```

> 对于每个配置的默认值和完整选项 [参考链接](https://doc.rust-lang.org/cargo)
{: .prompt-info }

## 在 `https://crates.io` 上发布库


### crate 的注册表在 `https://crates.io` 上
* 它会分发已注册的包的源代码
* 主要托管开源的代码


### 文档注释
* 用于生成文档
* 生成 HTML 文档
* 显示公共 API，教你如何使用 API
* 使用 `///` 编写
* 支持 Markdown
* 被放置在说明条目之前


```rust
/// Adds one to the number given.
/// 
/// # Examples
/// 
/// ```
/// let arg = 5;
/// let answer = profile::add_one(arg);
/// 
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```


```shell
// 生成文档
$ cargo doc
```

### 生产 HTML 文档的命令
* cargo doc
1. 它会运行 `rustdoc` 这个工具（Rust 安装时自带）
2. 将生成的 HTML 文档放在 `target/doc` 目录下


```shell
// 生成文档并自动打开文档
// 构建当前 crate 的文档（也包含 crate 依赖项的文档）
$ cargo doc --open
```


### 文档常用的章节
* `# Examples`
* `# Panics` 描述函数可能发生 `panic` 的场景，开发人员需要注意
* `# Errors` 描述如果函数返回 `Result`，可能的错误种类，以及可导致错误的条件
* `# Safety` 如果函数处于 `unsafe` 调用，就应该解释函数 `unsafe` 的原因，以及调用者确保的使用前提

### 文档注释作为测试
* 运行 cargo test，将会把文档注释中的示例代码作为测试来运行

### 为包含注释的项添加文档注释
* 为外层条目添加文档注释
* 符号用 `//!` 
* 这类注释通常用于描述 `crate` 和模块
1. 例如 `crate root`（按惯例就是 src/lib.rs）
2. 或一个模块内，将 `crate` 或模块作为一个整体进行记录

```rust
//lib.rs

//! # Profile  Crate
//! 
//! `Profile` is a collection of utilities
//! calculations more convenient.


/// Adds one to the number given.
/// 
/// # Examples
/// 
/// 这部分运行 cargo test 时会被执行
/// ```
/// let arg = 5;
/// let answer = profile::add_one(arg);
/// 
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

## 使用 `pub use` 重新导出
* 问题: crate 的程序结构在开发时对于开发者很合理，但对于它的使用者不够方便
1. 开发者会把程序结构分为很多层，使用者想找到这种深层结构中的某个类型很费劲
2. 麻烦：`my_crate::some_module::another_module::UsefulType`
3. 方便：`my_crate::UsefulType`

* 解决方案: 使用 `pub use` 重新导出
1. 不需要重新组织内部代码结构
2. 在有很多嵌套情况下，在顶层重新导出可以显著改善使用改 `crate` 的开发者的体验
3. 将所依赖的 `crate` 的定义在当前的 `crate` 中重新导出，使这些依赖的定义成为你的 `crate` 公共 API 的一部分


> 使用 `pub use` 就可以重新导出，创建一个与内部私有结构不同的对外公共结构
{: .prompt-info }


```rust
// lib.rs
// 使用 pub use 对 PrimaryColor 重新导出
pub use self::kinds::PrimaryColor;
pub use self::utils::mix;


pub mod kinds {

    #[derive(Debug)]
    pub enum PrimaryColor {
        Red,
        Blue,
    }

    pub enum SecondaryColor {
        Orange,
        Green,
        Purple,
    }

}

pub mod utils {
    use crate::kinds::*;

    pub fn mix(c: PrimaryColor) -> SecondaryColor {
        SecondaryColor::Orange
    }

}

// main.rs

// pub use 前
// use publishCrate_14::kinds::PrimaryColor;

// pub use 后, 这样导入就可以了！
use publishCrate_14::PrimaryColor;
use publishCrate_14::mix;

fn main() {

    let color = PrimaryColor::Blue;
    println!("color = {:?}", color);
}
```


## 发布 crate

### 创建并设置 `Crates.io` 账号
* 在发布 `crate` 之前，需要在 `crates.io` 创建账号并获得 `API Token`
* `https://crates.io/settings/tokens` 创建 `token`
* 运行命令：`cargo login [你的 token]`
1. 会通知 `cargo`，把你的 `API token` 存储在本地 `~/.cargo/credentials`，这个路径的 token 不要泄露给别人，
如果不小心泄露了，应该及时将泄露的 `token` 从 `https://crates.io` 上撤销

* 发布 `crate` 要确保 `cargo.toml` 里的 `name` 在 `https://crates.io` 上没有

### 为新的 `crate` 添加元数据
* 在发布 `crate` 之前，需要在 `Cargo.toml` 的 `[package]` 区域为 `crate` 添加一些元数据
1. `name`：`crate` 唯一名称
2. `description`：一两句话即可，会出现在 `crate` 搜索的结果里
3. `license`：许可证的标识值（可到 `http://spdx.org/licenses` 查找），可指定多个 `license`，使用 `OR` 分隔
4. `version`：版本
5. `author`：作者


> 更多指令查询 `https://doc.rust-lang.org/cargo/reference/manifest.html`
{: .prompt-info }


* 发布命令

```shell
$ cargo publish
```

### 发布 crate 流程
* 准备工作：在 `https://crates.io` 上验证邮箱


```shell
// 1
$ cargo login 你的 token

// 2
$ cargo publish
// 如果让提交一些文件可以
$ cargo publish --allow-dirty
```

* 发布成功后，在 `https://crates.io` 上查询包名

### 发布到 `Crates.io`
* `crate` 一旦发布，就是永久性的，该版本无法覆盖，代码无法删除
* 目的是依赖于 `crates.io` 该版本的项目可继续正常工作

### 发布已存在 `crate` 的新版本
* 修改 `crate` 后，需要先修改 `Cargo.toml` 里面的 `version` 值，再进行重新发布
* 参照 `http://semver.org` 来使用你的语义版本
* 然后再执行 `cargo publish` 进行发布

### 从 `Crates.io` 撤回版本
* 使用 `cargo yank` 命令进行撤回版本
* 不可以删除 `crate` 之前的版本
* 但可以防止其他项目把它作为新的依赖：`yank`（撤回）一个 `crate` 版本
1. 防止新项目依赖该版本
2. 已经存在的项目可继续将其作为依赖，并且可以下载
* `yank` 意味着：
1. 所有已经产生 `cargo.lock` 的项目都不会中断
2. 任何将来生产 `cargo.lock` 的文件都不会使用被 `yank` 的版本

* 命令（`yank` 一个版本，不会删除任何代码）

```shell
// 撤回
$ cargo yank --vers 1.0.1

// 取消撤回
$ cargo yank --vers 1.0.1 --undo
```

## 通过 Cargo 工作空间（Cargo workspaces） 组织大工程
* `cargo workspaces`：帮助管理多个相互关联且需要协同开发的 `crate`
* `cargo workspaces` 就是一套共享同一个 `cargo.lock` 和输出文件夹的包

### 创建工作空间
* 有多种方式来组建工作空间
* 例子由 1 个 `binary crate` 和 2 个 `lib crate`
1. `binary crate`：有 `main` 函数，依赖于其他 2 个 `lib crate`
2. 其中一个 `lib crate `提供 `add_one` 函数
3. 另一个 `lib crate` 提供 `add_two` 函数

```
$ mkdir workspace
$ cd workspace
$ touch Cargo.toml

// Cargo.toml 内容，用于配置整个 workspace
[workspace]

members = [
    "adder",
    "add-one",
]

$ cargo new adder

$ cargo build
// 产生 Cargo.lock 和 target 目录（所有成员的编译产出物）
// adder 没有自己的 target 目录，都会输出在 workspace 目录下的 target 中
// 所有成员共享 target，避免了依赖导致的重复编译

// 创建 lib crate
$ cargo new add-one --lib
```


> 在 `workspace` 中 `cargo build`，产生 `Cargo.lock` 和 `target` 目录（`所有成员的编译产出物`）。这里 `adder` 没有自己的 `target` 目录，都会输出在 `workspace` 目录下的 `target` 中，所有成员共享 `target`，避免了依赖导致的重复编译
{: .prompt-info }


* 让 `adder` 依赖 `add-one`，编辑 `adder` 里的 Cargo.toml

```
[package]
name = "adder"
version = "0.1.0"
edition = "2021"


[dependencies]
add-one = { path = "../add-one"}
```

* 编辑 adder 的 main.rs
 
```rust
use add_one;

fn main() {
    println!("Hello, world!");
    let n = 10;
    println!("Hello {} plus one is {}!", n, add_one::add_one(n));
}
```

* 运行 adder crate
```shell
// 通过 -p 指定 crate，相当于运行了 adder crate 的 main.rs
$ cargo run -p adder
```

### 在 `workspace` 中依赖外部 `crate`
* `workspace` 只有一个 `Cargo.lock` 文件，它在 `workspace` 顶层目录
* 如果 `workspace` 内的 `crate` 指定了同一依赖的不兼容版本，`Cargo` 会分别解析它们，但尽量使用尽可能少的版本
* `Cargo` 只在保证语义版本规则范围内的兼容性
    1. 假设一个 `workspace` 内有依赖 `rand 0.8.0`，另一个 `crate` 依赖 `rand 0.8.1` 根据语义版本规则，`0.8.1 与 0.8.0` 是兼容的，所以这个俩 `crate` 都会使用 `0.8.1`（或者更高的兼容版本，比如 `0.8.2`）
    2. 如果一个 `crate` 依赖 `rand 0.7.0`，另一个依赖 `rand 0.8.0`，这些版本在语义上是不兼容的。因此 `crate` 会为每个 `crate` 使用不同版本的 `rand`



```
[package]
// add-one Cargo.toml
name = "add-one"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = "0.3.14"

// adder Cargo.toml
[package]
name = "adder"
version = "0.1.0"
edition = "2021"

[dependencies]
add-one = { path = "../add-one"}
rand = "0.3.15"
```

* 执行测试

```shell
// 会执行 workspace 中的所有 test
$ cargo test

// 只执行 add-one 这个 crate 里的 test
$ cargo test -p add-one
```


> Note：`cargo publish` 没有提供类似 `all` 或 `-p` 的参数，所以对于 `workspace` 中的 `crate`，必须到各自的 `crate` 目录下执行 `cargo publish` 进行单独发布
{: .prompt-info }

[Demo](https://github.com/ZacharyWulven/Rust_Getting_Start/tree/master/workspace)


## 安装二进制 `crate`
* 命令 `cargo install`
* 来源：`https://crates.io`

> 只能安装具有二进制目标（binary target）的 `crate`
{: .prompt-info }

* 二进制目标就是一个可执行的程序
1. 由拥有 `src/main.rs` 或其他被指定为二进制文件的 `crate` 生成

* `lib crate` 无法单独执行
* 通常，README 里会有对 `crate` 的描述
1. 指明其是否拥有 `lib target`
2. 指明其是否拥有 `binary target`
3. 或两者都有

### `cargo install`
* 会把安装的二进制存放在根目录下的 `bin` 目录下
* 如果你使用 `rustup` 安装的 Rust，并且没有任何自定义配置，那么存放路径就是 `$HOME/.cargo/bin`
1. 为了确保二进制程序可以运行，需要确保该目录在环境变量 `$PATH` 中

* 安装二进制 crate

```shell
// 我们安装之前上传的 publishCrate_14
$ cargo install publishCrate_14

// 下边是安装输出的安装路径，xxx 是你的 User 名字
//   Installing /Users/xxx/.cargo/bin/publishCrate_14

// 可以执行运行 publishCrate_14
$ publishCrate_14

// 查看环境变量
$ echo $PATH
```

## 使用自定义命令扩展 cargo
* cargo 被设计成可以使用子命令来扩展
* 例如：在 `$PATH` 中的某个二进制是`cargo-something`，你可以像子命令一样运行 `cargo something`
1. 类似这样的自定义命令可以使用命令 `cargo --list` 列出
2. 这样的好处是我们可以使用 `cargo install` 来安装扩展，像内置工具一样来运行







