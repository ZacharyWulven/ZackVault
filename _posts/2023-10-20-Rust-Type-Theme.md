---
layout: post
title: Rust 中级教程-类型主题-01
date: 2023-10-14 16:45:30.000000000 +09:00
categories: [Rust, Rust Type]
tags: [Rust, Rust Type]
---


# 0x01 指针

## 1 什么是指针
* 指针：即是计算机引用无法立即直接访问的数据的一种方式
* 数据在物理内存（RAM）中是分散的存储的，地址空间是检索系统


> 指针就是被编码为内存地址，使用 `usize` 类型的整数表示的。一个地址就好指向地址空间中的某个地方。
{: .prompt-info }


* 地址空间的范围是 `OS` 和 `CPU` 提供的外观界面
  * 程序只知道有序的字节序列，不会考虑系统中实际的 `RAM` 的数量
  
## 2 名词解释
* 内存地址（地址）：即指代内存中单个字节的一个数
  * 内存地址是汇编语言提供的一种抽象
* 指针（有时扩展称为原始指针）：即指向某种类型的一个内存地址
  * 指针是高级语言提供的抽象
* Rust 中的引用，就是指针。如果是动态大小的类型，就是指针和具有额外保证的一个整数
  * 引用是 Rust 提供的一种抽象


## 3 Rust 的引用
* 引用始终引用的是有效的数据
* 引用与 `usize` 的倍数的对齐的，从而不会降低运行的速度
* 引用可以为动态大小的类型提供上述的保障


## 4 Rust 的引用和指针

```rust
static B: [u8; 10] = [99, 97, 114, 114, 121, 116, 111, 119, 101, 108];
static C: [u8; 11] = [116, 104, 97, 110, 107, 115, 102, 105, 115, 104, 0];

fn main() {

    let a = 42;
    let b = &B;
    let c = &C;

    /*
        {:p} 表示打印指针的地址
     */
    println!("a: {}, b: {:p}, c: {:p}", a, b, c); // a: 42, b: 0x10c7ea44c, c: 0x10c7ea456
}
```

![image](/assets/images/rust/type/memory.png)

* 看图可见，a 占 4 个字节，这里内存是对齐的
* `b 和 c` 是指针是用 `usize` 来表示的，在 32 位的 CPU 上引用是 4 字节，在 64 位上引用是 8 字节
* `b 和 c` 是指针会指向另外一个地址，一个连续的内存块


### 4.1 进一步解释想要的效果
* 想要模拟智能指针和原始指针
* 还是刚才的代码，下图展示了内存结构

> 这个例子中指针的宽度是两个字节
{: .prompt-info }

![image](/assets/images/rust/type/memory_01.png)


* `a` 的类型是 `i16`，在内存中就是 `0x31 和 0x30`
* 这个图中 `b 和 c` 在内存上是不一样的，这是因为上边的代码有一些欺骗性，这其实是我们想要达到的效果
* `b` 的类型是智能指针
  * 它包含：长度字段和地址字段
  * 它的长度字段是 `10`，对应 `0x2D 和 0x2c`
  * 它的地址字段：对应 `0x2F 和 0x2E`，地址是 `32`，即 `0x20` 开始
  * 它就是长度为 10 的固定宽度的 `buffer`，包含不带终止符的字节，不以 `0` 结尾
  * 当在指针类型后面使用时，`buffer` 通常称为后备数组
  * `b 和 B` 一起几乎可以创建出 Rust 中的字符串类型，但字符串类型还包含一个`容量字段`
  
  
* `c` 是原始指针，占两个字节，里边直接存的就是地址，这里是十进制的 `16`，即 `0x10` 开始
  * `c` 是以 `0` 结尾的 `buffer`，就是 `C` 语言中的字符串的内部表示，`C` 中字符串就是一个数组
  * 了解如何将这些类型转换为 Rust 类型，对于通过 FFI(外部函数接口) 处理外部代码非常有用
  * `c 和 C` 一起就是 Rust 里的 `CStr` 类型
  

> `0x0` 就是 `NULL byte`，它是程序的死区。如果指针指向此处，然后解引用，程序通常就会崩溃
{: .prompt-info }



### 4.2 一个更逼真的例子
* 使用更复杂的类型展示指针内部的区别：原始指针和智能指针
* 智能指针包括：
  * 长度字段
  * 地址字段
  * 容量字段
  

```rust
use std::mem::size_of;
static B: [u8; 10] = [99, 97, 114, 114, 121, 116, 111, 119, 101, 108];
static C: [u8; 11] = [116, 104, 97, 110, 107, 115, 102, 105, 115, 104, 0];

fn main() {
    let a: usize = 42;
    let b: Box<[u8]> = Box::new(B); // B 里值的所有权就跟着 Box 走了
    let c: &[u8; 11] = &C;

    println!("a (unsigned 整数)：");
    println!("  地址: {:p}", &a);
    println!("  大小: {:?} bytes", size_of::<usize>());
    println!("  值:   {:?}\n", a);

    println!("b (B 装在 Box 里)：");
    println!("  地址: {:p}", &b);
    println!("  大小: {:?} bytes", size_of::<Box<[u8]>>());
    println!("  指向: {:p}\n", b);

    println!("c (C 的引用)：");
    println!("  地址: {:p}", &c);
    println!("  大小: {:?} bytes", size_of::<&[u8; 11]>());
    println!("  指向: {:p}\n", c);


    println!("B (10 bytes 的数组)：");
    println!("  地址: {:p}", &B);
    println!("  大小: {:?} bytes", size_of::<&[u8; 10]>());
    println!("  值: {:?}\n", B);


    println!("C (11 bytes 的数组)：");
    println!("  地址: {:p}", &C);
    println!("  大小: {:?} bytes", size_of::<&[u8; 11]>());
    println!("  值: {:?}\n", C);
}



a (unsigned 整数):
  地址: 0x7ff7b73eeb48
  大小: 8 bytes
  值:   42

b (B 装在 Box 里):
  地址: 0x7ff7b73eeb50
  大小: 16 bytes
  指向: 0x7fcd17f05b20 // 地址在 heap 上。由于 B 是静态变量，所以它在静态内存区域，但是把 B 放到 Box 中后，会在 heap 上生成新的地址，这里这个地址就是指向 heap 上的地址

c (C 的引用):
  地址: 0x7ff7b73eeb70
  大小: 8 bytes
  指向: 0x108b5009e    // 因为 C 就是个原始指针（引用），这里指向的地址就是下边 C 的地址

B (10 bytes 的数组):
  地址: 0x108b50094
  大小: 8 bytes
  值: [99, 97, 114, 114, 121, 116, 111, 119, 101, 108]

C (11 bytes 的数组):
  地址: 0x108b5009e
  大小: 8 bytes
  值: [116, 104, 97, 110, 107, 115, 102, 105, 115, 104, 0]
```


### 4.3 对 B 和 C 中的文本进行解码的例子
* 它创建了一个与前图更加相似的内存地址布局

```rust
// Cow 是一个智能指针，代表 copy on right, 
// 只有需要写入的时候才需要复制,平时读的时候就不需要复制了
use std::borrow::Cow;
// CStr 类似 C 语言的字符串，允许读取以 0 结尾的字符串
use std::ffi::CStr;
// c_char 就是 Rust 中的 i8 的别名
use std::os::raw::c_char;

static B: [u8; 10] = [99, 97, 114, 114, 121, 116, 111, 119, 101, 108];
static C: [u8; 11] = [116, 104, 97, 110, 107, 115, 102, 105, 115, 104, 0];


fn main() {
    let a = 42;
    let b: String;    // 智能指针
    let c: Cow<str>;  // 智能指针

    unsafe {
        // 需要再 unsafe 块中执行
        // *mut u8 一个可变的原始指针
        let b_ptr = &B as *const u8 as *mut u8;  // 原始指针
        b = String::from_raw_parts(b_ptr, 10, 10);

        let c_ptr = &C as *const u8 as *const c_char;
        c = CStr::from_ptr(c_ptr).to_string_lossy();
    }
    println!("a: {} b: {} c: {}", a, b, c);  // a: 42 b: carrytowel c: thanksfish
}
```

## 5 原始指针（Raw Pointers）
* 原始指针（Raw Pointers）：即没有 Rust 标准保障的内存地址
  * 这些指针本质上是 unsafe 的
* 语法：
  * 不可变的原始指针： `*const T`
  * 可变的原始指针：`*mut T`
  * Note：`*const T`，这三个标记放到一起才表示的是一个原始指针类型
  * 例子：`*const String` 表示指向 `String` 类型的原始指针，它是不可变的
  
* `*const T` 和 `*mut T` 之间的差异很小，可以相互自由转换
* Rust 的引用（`&mut T 和 &T`）最终会编译为原始指针
  * 所以平时无需冒险进入 `unsafe` 块，就可以获得原始指针的性能

* 例子1：把引用转为原始指针

```rust
fn main() {
    let a: i64 = 42;
    // *const i64，即将其引用 &a 转为 *const i64
    let a_ptr = &a as *const i64;

    println!("a: {}, ({:p})", a, a_ptr); // a: 42, (0x7ff7b358c1b0)
}
```

* 解引用（dereference）：通过指针从 `RAM` 内存提取数据的过程就叫做对指针进行解引用（dereferencing a pointer）。
* 例子2：把引用转为原始指针

```rust
fn main() {
    let a: i64 = 42;
    let a_ptr = &a as *const i64;
    
    let a_addr: usize = unsafe {
        // 使用 transmute 转为 usize 形式
        std::mem::transmute(a_ptr)
    };
    println!("a: {} ({:p}...0x{:x})", a, a_ptr, a_addr + 7); // a: 42 (0x7ff7bf91f170...0x7ff7bf91f177)
}
```

### 关于 `Raw Pointer` 的提醒

> 在底层原理，引用`（&T 和 &mut T）`都没实现为原始指针。但是引用还带有额外的保障，应该始终作为首选来使用。访问 `Raw Pointer` 的值总是 `unsafe` 的。
{: .prompt-info }


* `Raw Pointer` 不拥有值的所有权
  * 在访问时编译器也不会检查数据的合法性
* Rust 允许多个 `Raw Pointer` 指向同一个数据
  * 同时 Rust 无法保证共享数据的合法性


### 使用 `Raw Pointer` 的情况
* 不可避免
  * 某些 OS 或第三方库需要使用，例如与 C 交互
* 对某些内容的共享访问是比较重要的，而且运行时的性能要求也比较高

## 6 Rust 指针的生态

### `Raw Pointer` 是 `unsafe` 的
### `Smart Pointer`（智能指针）倾向于是包装原始指针，同时附加了更多的能力，或叫更多的语义
  * 就是说智能指针不仅仅是对内存地址进行解引用

* Rust 智能指针

![image](/assets/images/rust/type/smart_p.png)


![image](/assets/images/rust/type/smart_p2.png)



# 0x02 内存

## 1 值
* 值：即某个类型 + 某个类型值域中的一个元素
  * 例如：true，类型是 bool，值域中就俩 true 或 false,
* 通过值的类型表示，也可以转化为字节序列
  * 例如 6，u8 类型，数学整数；在内存中表示：`0x06`
  * 例如 str "hello world" 字符串域的值，它的表示是 UTF8 编码
* 值是独立于存储它字节的位置的，即值和存储它的位置是没有关系的，独立的两个概念
  
## 2 变量
* 值会存储到一个地方（这个地方可以容纳值）
* 可以在 `Stack、Heap` 或者其他地方
* 最常见的存储值的地方就是变量，它是 `Stack` 上面的一个被命名的值的槽


![image](/assets/images/rust/type/slot.png)


## 3 指针
* 指针也是一个值，值里面存放的是一块内存的地址，指针指向某个地方
* 指针可以被解引用，来访问它指向的内存里存放的值
* 可以把同一个指针存放在不同的变量里，也就是说多个变量可以间接的引用内存中的同一个地方，也就是相同的底层的值

![image](/assets/images/rust/type/pointer.png)

* 看个例子

```rust
fn main() {
    // 一共 4 个值：42、43、&x、&y
    // 一共 4 个变量：x、y、var1、var2
    // var1、var2 是指针类型，也叫引用，它们是两个变量，两个槽
    let x = 42;
    let y = 43;
    let var1 = &x;
    let mut var2 = &x;
    var2 = &y;


    //---------------------------------------------
    // s 就是一个指针，它指向第一个字符的位置
    let s = "Hello World";
}
``` 


## 4 深入讲下变量
* 高级模型：从生命周期、借用等角度来看变量
* 低级模型：从不安全代码、原始指针、内存角度来看变量

### 变量的高级模型：
* 变量就是给值的一个名称
* 把值赋给一个变量的时候，这个值从那时候起就由该变量命名了
  * `let dave = 1234;`
  
 
![image](/assets/images/rust/type/var.png)


* 当变量被访问的时候，可以从变量的上次访问到这次访问画一条线，从而在两次访问之间建立依赖关系
* 如果变量被移动了，就不能从它那画线了
  * 下图因为 `a` 赋值给 `b`，发生了 `move`，所以最后不能打印了，即不能画线了


![image](/assets/images/rust/type/var2.png)

* 在高级模型中，变量只会在它持有合法值的时候才存在。如果变量的值未初始化，或者已经被移动了，那就无法从该变量画线了
  * 下图中的 `c` 因为没有初始化，所以也不能打印，即不能画线

![image](/assets/images/rust/type/var3.png)


* 使用该模型，整个程序会由许多依赖线组成，这些线叫做 `flow`
* 每个 `flow` 都在追踪一个值的特定实例的生命周期
* 当由分支存在时，`flow` 可以分叉或合并，每个分叉都在追踪该值的不同的生命周期
* 在程序中的任何给定点，编译器可以检查所有的 `flow` 是否可以互相兼容、并行存在：
  * 例如：一个值不可能有两个具有可变访问的并行 `flow`。也不能一个 `flow` 借用了一个值，但却没有 `flow` 拥有该值（依据借用规则）
    * 下图 `a` 已经借用为一个不可变引用了，就不能再借用为一个可变引用了
  
![image](/assets/images/rust/type/var4.png)


### 变量的低级模型：
* 变量会给那些可能（不）存储合法值的内存地点进行命名 
* 可以把变量想象为值的槽：当你赋值的时候，槽就装满了，而它里边原来的值（如果有的话）就被丢弃或替换了


> 当访问它时，编译器检查槽是不是空的，如果是空的，就说明变量未初始化，或者它的值被 `move` 了
{: .prompt-info }


* 而指向变量的指针，其实是指向变量的幕后的内存，通过解引用可以获得它的值
* 在本例中，我们忽略了 CPU 的寄存器，并将其视为优化。实际上，如果变量不需要内存地址，编译器可以使用寄存器而不是内存区域来存放该变量


## 5 内存区域
* 有许多内存区域，并不是都在 `DRAM` 上
* 三个比较重要的区域：`stack 内存`、`heap 内存`、`static 内存`

* `stack 内存` 和 `heap 内存`
  * `stack 内存` 比较快、比较整齐
  * `heap 内存` 比较慢、比较混乱


## 6 `stack 内存`
* 当有疑问时，首先使用 `stack 内存`
  * 想把数据放在 `stack 内存` 上，编译器必须知道类型的大小

> 换句话说：就是使用了实现 `Sized` 的类型，才能放到 `stack 内存` 上
{: .prompt-info }

* `stack` 是一段内存，程序把它作为一个暂存控件，用于函数调用
* `stack` 是后进先出（LIFO）

### `stack frame`

* 每个函数每次被调用时，在 `stack` 顶部都会分配一个连续的内存块，它叫做 `stack frame`（栈帧）
* 在接近 `stack` 的底部附近是 `main` 函数的 `frame`，随着函数的调用，其余的 `frames` 都推到了 `stack` 上


![image](/assets/images/rust/type/stack.png)


* 函数的 `frame` 包含了函数里所有的变量，以及函数所带的参数。
* 当函数返回的时候，它的 `frame` 就被回收了
  * 构成函数本地变量值的那些字节不会立即被擦除，但访问它们也是不安全的。
  * 因为它们可能被后续的函数调用所重写（如果后续函数调用的 `frame` 与回收的这个有重合的话，这时访问它们是不安全的）
  * 即使没有被重写，它们也可能包含无法合法使用的值。例如函数返回后被移动的值
  


### `stack pointer`
* 随着程序的执行，CPU 里有一个游标会随着更新，它会反映出当前 `stack frame` 的当前地址，这个游标叫 `stack pointer`（stack 指针）
* 随着函数内不断地调用函数，`stack` 就会增长，而 `stack pointer` 的值会减少，当函数返回时 `stack pointer` 的值会增加

![image](/assets/images/rust/type/stack2.png)


### 继续说 `stack frame`

* `stack frame` 也叫 `activation frames` 或 `allocation record`
  * 只有当 `activation frames` 分配在栈内存上时候才叫 `stack frame`
  * 每个 `stack frame` 的大小是不同的，可以参考上图
* 在函数调用期间，`stack frame` 会包含函数的状态，当一个函数在另外一个函数内调用时，原来的函数的值会被及时的冻结
* `stack frame` 为函数参数，以及指向原来调用栈的指针和本地变量（不包含在 `heap` 上分配的数据）提供了空间
* `stack` 的主要任务是为本地变量创造空间，但它为什么快呢？
  * 因为函数的所有变量在内存里都是紧挨着的，所以找起来比较快


* 例子：比如想让一个函数同时能处理 `&str` 和 `String`
  * `&str` 是在 `stack` 上
  * `String` 是在 `heap` 上

```rust

fn main() {
    let pw = "jackdss";
    let is_strong = is_strong(pw);

}

// &str -> Stack; String -> Heap
// 只能接收 String 类型，传入 &str 会报错
// fn is_strong(password: String) -> bool {
//     password.len() > 5
// }


// T 是实现了 AsRef<str>，即将它作为到 AsRef<str> 的引用
// 这个版本可行
// fn is_strong<T: AsRef<str>>(password: T) -> bool {
//     password.as_ref().len() > 5
// }

// T 可以转化为 String 类型
// 这个版本可行
fn is_strong<T: Into<String>>(password: T) -> bool {
    password.into().len() > 5
}
```
 
* `stack frames` 它们最终也会消失这个事实，其实与 Rust 生命周期的概念密切相关的
* 任何在 `stack` 上的 `frame` 里存储的变量，在 `frame` 消失后，它就无法访问了
  * 所以任何到这些变量的引用的生命周期，最多只能与 `frame` 的生命周期一样长


## 7 `heap 内存`
* `heap` 就意味着混乱
* `heap` 是一个内存池，并没有绑定到当前程序的调用栈，而 `stack` 相当于绑定到了当前程序的调用栈
* `heap` 是为在编译时没有已知大小的类型准备的
* 什么叫编译时类型大小不已知呢？
  * 一些类型随着时间会变大或变小，例如 `String、Vec<T>`
  * 另一些类型的大小不会改变，但是无法告诉编译器需要分配多少内存
  * 这些都叫做动态大小的类型，例如 [T]（切片）
  * 另一个例子就是 `trait` 对象，它允许程序员来模拟一些动态语言的特性：通过允许将多个类型放进一个容器来实现的
* `heap` 允许你显式的分配连续的内存块，当这么做时，你会得到一个指针，它指向内存块开始的地方
* `heap` 内存中的值会一直有效，直到你对它显式的释放
* 如果你想让值活的比当前函数 `frame` 生命周期还长，那么使用 `heap` 就很有用了
  * （在栈上实现）如果值是函数的返回值，那么调用函数可以在它的 `stack` 上留一些空间给被调用函数让它把值在返回前写入进去

### `heap` 线程安全的例子
* 如果想把值送到另一个线程，当前线程可能根本无法与那个线程共享 `stack frames` 你就可以把它存放在 `heap` 上
* 因为函数返回时 `heap` 上的分配并不会消失，所以你在一个地方为值分配内存，把指向它的指针传给另一个线程，就可以让那个线程继续安全的操作于这个值
* 换一种说法：即当你分配 `heap` 内存时，结果指针会有一个无约束的生命周期，你的程序让它活多久都行

### `heap` 交互机制
* `heap` 上的变量必须通过指针来访问

```rust
fn main() {
    let a: i32 = 40; // Stack
    let b: Box<i32> = Box::new(30); // Heap

    // Error，因为 b 在 heap 上只能通过指针来访问
    // let result = a + b; 

    // Ok
    let result = a + *b;

    println!("{} + {} = {}", a, b, result); // 40 + 30 = 70

}
```

* Rust 中与 `heap` 交互的首要机制就是 `Box` 类型


> 当 `Box::new(value)` 时，值就会被放到 `heap` 上，返回的 `Box<T>` 就是指向 `heap` 上该值的指针。当 `Box` 被丢弃时，内存就被释放了。
{: .prompt-info }

* 如果忘记释放 `heap` 内存，就会导致内存泄漏
  * 但有时你需要让内测泄漏：
    * 例如有一个只读的配置，整个程序都需要访问它，就可以把它分配在 `heap` 上，通过 `Box::leak` 得到一个 `'static` 引用，从而显式的让其进行泄漏，
    让整个程序在关闭前一直可以访问（有点类似单例）

```rust
use std::mem::drop; // 可手动进行释放

fn main() {
    let a = Box::new(1);
    let b = Box::new(1);
    let c = Box::new(1);

    let result1 = *a + *b + *c;
    
    drop(a);  // 手动释放 a

    let d = Box::new(1);
    let result2 = *b + *c + *d;

    println!("result1={}, result2={}", result1, result2);
}
```

下图为上边代码的内存示意图

 
![image](/assets/images/rust/type/heap_a_b_c.png)


* 上边例子，当 `drop(a)` 后，`a` 在 `stack` 上不可安全的进行访问，其生命周期已经结束


## 8 `Static` 内存
* `Static` 内存实际上是一个统称，它指的是程序编译后的文件中几个密切相关的区域
  * 当程序执行时，这些区域会自动加载到你的程序的内存里
* `Static` 内存里的值在你的程序的整个执行期间会一直存活
* 程序的 `Static` 内存也包含程序的二进制代码（通常映射为只读的）
  * 随着程序的执行，它会在本文段的二进制代码中挨个指令进行遍历，而当函数被调用时就进行跳跃（了解即可）
* `Static` 内存会持有使用 `static` 声明的变量的内存，也包括某些常量`值`，例如字符串


### `'static` 生命周期
* `'static` 是一个特殊的生命周期
  * 它的名字就来自于 `static 内存区`，它将引用标记为只要 `static 内存` 还存在（即程序关闭前），那么怎么引用都合法 
* `static` 变量的内存在程序开始运行时就分配了，到 `static` 内存中变量的引用，按定义来说就是 `'static` 的，因为在程序关闭前它不会被释放
* 反过来说就不行，是可以有 `'static` 的引用不指向 `static` 内存的
* 但是 `static` 名称仍然适用：
  * 一旦你创建一个 `'static` 生命周期的引用，就程序的其余部分而言，它所指向的内容都可能在 `static` 内存中，因为程序想要使用它多久就可以使用多久

* 你可能更多会遇到 `'static` 生命周期，而不是 `static` 内存
* `'static` 经常出现在类型参数的 `trait bounds` 上
  * 例如：`T: 'static` 表示类型 `T` 可以存活我们想要的任何时长（直到程序关闭），`同时这也要求 T 是拥有所有权和自给自足的`
    * 即 `T` 要么它不借用其他（非 `static`） 的值
    * 或即 `T` 要么借用的值都是 `static` 的
  * 这样这个 `T` 将一直保留到程序结束
  
### `const` VS `static`
* `const` 关键字会把紧随它的东西声明为常量，例如 `const X: i32 = 123;`
* 常量可以在编译时完全计算出来，在编译期间，任何引用常量的代码会被替换为常量的计算结果值
* 所以常量没有内存或关联其他存储的意思（因为它不是任何一个地方）
* 你可以把常量理解为某个特殊值的方便的名称


## 9 动态内存分配
* 任何时刻，运行中的程序都需要一定数量的内存
* 当程序需要更多内存时，就需要从 OS 请求，这就叫动态内存分配（dynamic allocation）


### 动态内存分配步骤

![image](/assets/images/rust/type/alloc.png)


### 为什么 `Stack` 和 `Heap` 存在性能差异?
* `Stack` 和 `Heap` 只是概念而已，内存在物理上并不存在这两个区域
* 访问 `Stack` 是比较快的，原因如下：
  * 函数的本地变量（都分配在 `Stack` 上）在 RAM 上都彼此相邻，连续布局
  * 连续布局对缓冲比较友好

* 访问 `Heap` 是比较慢的，原因如下：
  * 分配在 `Heap` 上的变量不太可能彼此相邻
  * 访问 `Heap` 上的数据还涉及对指针进行解引用（页表查找和去访问主存）
  
  
### `Stack` VS `Heap`


![image](/assets/images/rust/type/s_h.png)


> `Stack` 上的数据结构在生命周期内大小不能变，`Heap` 上的数据结构更灵活，因为指针可以改变
{: .prompt-info }


## 10 虚拟内存
* 即程序的内存视图，程序可以访问的所有数据都是由操作系统在其他地址空间中提供的
* 直觉上，程序的内存就是一系列的字节，从开始位置 `0` 到结束位置 `n`
  * 例如：程序汇报使用了 100 KB 的 RAM（内存），那么 n 就应该是在 100000 左右


```rust
fn main() {

    let mut n_nonzero = 0;

    /*
        扫描运行内存，从 0 开始扫描
        当 i == 0 时，就是 null 指针，是非法的
    */
    for i in 0..10000 {
        let ptr = i as *const u8;
        let byte_at_addr = unsafe { *ptr };

        if byte_at_addr != 0 {
            n_nonzero += 1;
        }
    }

    println!("内存中的非 0 字节：{}", n_nonzero); // Error: 68355 segmentation fault  cargo run

}
```

> `segmentation fault` 指当 CPU 或 OS 检测到程序试图请求非法（无权访问）内存地址时，所产生的错误
{: .prompt-info }


* 看一下各个变量的内存地址分布

```rust
// 静态全局变量
static GLOBAL: i32 = 1000;

fn noop() -> *const i32 {
    let noop_local = 123456;
    &noop_local as *const i32
}

fn main() {
    let local_str = "a";
    let local_int = 123;
    let boxed_str = Box::new('b');
    let boxed_int = Box::new(789);
    let fn_int = noop();

    println!("GLOBAL:        {:p}", &GLOBAL as *const i32);    // GLOBAL:        0x10df4b190
    println!("local_str:     {:p}", local_str as *const str);  // local_str:     0x10df4b194
    println!("local_int:     {:p}", &local_int as *const i32); // local_int:     0x7ff7b1ff3f8c
    println!("boxed_int:     {:p}", Box::into_raw(boxed_int)); // boxed_int:     0x7fefeef05b50
    println!("boxed_str:     {:p}", Box::into_raw(boxed_str)); // boxed_str:     0x7fefeef05b40
    println!("fn_int:        {:p}", fn_int);                   // fn_int:        0x7ff7b1ff3ebc
}
```

* `segment`：虚拟内存中的块。虚拟内存被划分为很多块，目的是可以以最小化虚拟和物理地址之间转换所需要的空间。
* 通过上边例子，可以发现某些内存地址是非法的：访问越界的内存，OS 就会关掉你的程序。
* 另外内存地址并不是随机的：看起来在地址空间内分布的比较广，值相当于是分成了几堆儿。


### 翻译虚拟地址到物理地址
* 程序里访问数据都是需要虚拟地址的（程序只能访问虚拟地址）
* 虚拟地址会被翻译成物理地址
  * 这里涉及程序、OS、CPU、RAM 硬件，有时涉及硬盘或其他设备
  * CPU 复杂执行翻译，OS 负责存储指令
  * CPU 包含一个内存管理单元（MMU）负责这项工作
  * 这些指令也存在内存中一个预定义的地址中
* 最坏情况下，每次访问内存都会发生两次内存查找
  * 一个是内存的查找，另一个是指令查找
* CPU 会维护一个最近转换地址的缓存
  * 它有自己的快速内存来加速内存的访问
  * 历史原因，该内存称为转换后备缓冲区（Translation Lookaside Buffer，TLB）
  
* 为了提高性能，程序员需要保持数据结构的精简，避免深度嵌套
  * 尤其是达到 TLB 的容量后（对于 x86 处理器，通常约为 100 页）可能成本更高

* 页（Page）：实际内存中固定大小的字块，64 为系统通常是 4K 一页
* 字（Word）：指针大小的任何类型，对应 CPU 寄存器的宽度
  * `usize` 和 `isize` 就是字长类型（word-length type）

* 虚拟地址被分为很多块，叫做页，通常 4K 大小
  * 这样好处是：避免了需要为每个变量都存储转换映射
  * 页是统一大小的，这有助于避免内存碎片（碎片即：可用 RAM 中出现空的，但不可用的空间）


> Note：上面这些只是通用性指导，例如像微控制器情况就不同了。
{: .prompt-info }


### 数据在 RAM 中展示的指导建议
* 将程序热工作的部分保持在 4KB 以内，从而保持快速查找
* 如果 4KB 不合理，那么下一个目标就是 `4KB * 100`
  * 这意味着 CPU 可维护其转换缓存（TLB）来支持你的程序
* 另外一点就是避免深度嵌套的数据结构
  * 如果指针指向另一个页，那么性能就会受到影响
* 测试嵌套循环的顺序：
  * CPU 会从 RAM 读取小块字节（cache line、缓存行）。在处理数组时，可以通过判断是按列操作还是按行操作来利用这一点


### 注意
* 虚拟化会让情况更糟糕，如果在虚拟机中运行程序，Hypervisor 还必须为其客户 OS 转换地址。
  * 这就是为什么许多 CPU 附带虚拟化支持，这可以减少额外开销
* 如果在虚拟机中运行容器又添加了一层间接，也增加了延迟
* 如果想获得裸机的性能，就必须在裸机上运行程序

* 系统调用（system call）：OS 提供了接口可让程序发出请求



