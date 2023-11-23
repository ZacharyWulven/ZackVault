---
layout: post
title: Rust 中级教程-类型主体
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



    // s 就是一个指针，它指向第一个字符的位置
    let s = "Hello World";
}
```
