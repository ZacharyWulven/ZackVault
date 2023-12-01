---
layout: post
title: Rust 中级教程-类型主题-02
date: 2023-10-28 16:45:30.000000000 +09:00
categories: [Rust, Rust Type]
tags: [Rust, Rust Type]
---


# 0x03 所有权（回顾）

## Rust 内存模型的核心思想：所有的值都只有一个所有者
* 即只有一个位置（通常是作用域）来负责释放每个值。通常变量走出其作用域就被释放了
* 这是通过借用检查器来实现的
  * 如果值移动了（比如：赋值新变量、推到 Vec 中、置于 Heap），其所有者也变成新的位置了
* 但有些类型不遵守这个规则
  * 如果值的类型实现了 `Copy trait`，重新赋值时到新内存地址时，发生的就不是移动了，而是复制
  * 大多数的原始类型，例如整数、浮点类型等，都实现了 `Copy trait`
  
## 如何实现 `Copy trait`
* 必须可以按位（bit）来把值进行复制
  * 所以就不包括：
    * 这个值中（的属性）含有 `non-Copy` 类型的类型
    * 拥有这类资源的类型：当类型的值被丢弃时，必须被释放该资源
  
* 为什么呢？
  * 如果 `Box` 是 `Copy` 的，进行赋值：`box1 = box2`，那么它们都认为自己在 heap 上持有那块内存，当它们走出作用域都会去尝试释放那块内存，就会出现二度释放的问题



## 值的删除（丢弃，dropped）
* 当值不再被需要时，其所有者就会将其删除
  * 当值不再存在于作用域内，就会被删除
* 如果值还包含其他的类型，那么类型会递归的将其所包含的值进行删除
  * 例如删除一个复杂类型的变量，可能会导致很多值被删除


> 因为所有权的设计，Rust 不会发生多次删除同一个值的情况
{: .prompt-info }


* 如果变量含义对其他值（比如叫 B）的引用（不拥有该值），当变量被删除时，其他的值（B）不会被删除


## 值删除的顺序
1. 变量（包括函数的参数）是按定义相反的顺序进行删除的
2. 嵌套的值是按照源代码的顺序进行删除的

```rust
fn main() {
    let x1 = 42;
    let y1 = Box::new(88);

    {
        /*
            x1 实现了 Copy，这里只是发生了 Copy，所以下边还可以把 x1 赋值给 x2
            y1 则发生了移动
            
            值删除的顺序
            第一条：y1 先声明，z 后声明，删除的时候先删除 z 后删除 y1
            第二条：先删除 x1 再删除 y1
         */
        let z = (x1, y1);
    }

    // Ok
    let x2 = x1;

    // Error, 因为上边作用域 y1 移动给了 z，所以这里不能再使用 y1，所以报错
    let y2 = y1;
}
```


> 备注：Rust 暂时不允许在单个值内进行自我引用的
{: .prompt-info }



# 0x04 引用及内部可变性（回顾）

## 引用
* 通过引用，Rust 允许将值借用出去，但不放弃所有权
* 引用就是带有附加合约的指针


### 共享引用
* `&T`，就是可以共享的指针
  * 可以同时存在任意数量的引用指向同一个值
  * 每个共享的引用都实现了 `Copy`
  
* 其背后的值是不可变的
  * 编译器允许假定共享引用指向的值，在该引用存活期间是不会改变的
  * 例如：一个共享引用的值在某函数内被多次读取，那编译器就有权让其只读取一次，然后后续重用这个读取的值


### 可变引用
* `&mut T`，可变引用是独占的
  * 编译器假定没有其他线程访问目标值（即如果存在了一个可变引用，就不会出现其他的共享引用或可变引用）

* 例子

```rust
/*
    input 与 output 指向不同的内存，
    因为 output 是可变的，它是独占的
    函数里，修改 output 不会影响 input

    但如果可变引用不是独占的，那么 input 与 output 就可能指向相同的内存，
    这时函数里对 output 的修改可能影响后边的代码逻辑
*/
fn noalias(input: &i32, output: &mut i32) {
    if *input == 1 {
        *output = 2;
    }
    if *input != 1 {
        *output = 3;
    }
}
```

*（重点）可变引用只允许你修改引用所指向的内存地址
  * 看例子
  
```rust
fn main() {
    let x = 42;
    // 1 可以让 y 指向其他的值，但不可以通过 y 来修改 x 的值
    let mut y = &x;   
    // 因为 z 没有 mut 修饰，所以它只能持有对 y 的可变引用，但可以通过 z 来修改 y 的值
    let z = &mut y;

    // *y = 10;  // Error

    let n = 20;
    *z = &n; // 修改 y 的值，y 指向 &n
}
```

## 拥有值 VS 拥有到值的可变引用
* 区别在于：所有者需要对删除值（丢弃值）进行负责


> Note：如果你移动了可变引用背后的值，则必须在其位置上留下另一个值。如果不这样做，所有者会认为它需要将其删除（丢弃），但其实却没有值可以删除了。
{: .prompt-info }

*（重要）上边 Note 的例子：

```rust
fn replace_with_84(s: &mut Box<i32>) {
    // this is not okay，as *s would be empty:

    /*
        这句不能执行
        如果这句可以那么值就移动了，但调用者依然认为他们拥有其所有权，
        所以他们还是会对这个值进行丢弃操作
     */
    // let was = *s;

    /*
        take 函数将 s 移动，并将 s 类型的默认值放到了原来的位置上了，
        这时调用者就会拥有那个新的默认的那个值，
        然后调用者再清理这个变量就不会有问题了
        
        Note：如果你移动了可变引用背后的值，则必须在其位置上留下另一个值，使用 take 留下默认值
     */
    let was = std::mem::take(s);

    println!("was init is {}", was); // was init is 42
    println!("was init s is {}", s); // was init s is 0

    /*
        was 即其他一个合理的值，
        相当于又把原来 s 的值赋值回去了，又变成 42 了
     */
    *s = was;
    println!("*s = was is {}", s);   // *s = was is 42

    let mut r = Box::new(84);
    std::mem::swap(s, &mut r);
    assert_ne!(*r, 84);
}

fn main() {
    let mut s = Box::new(42);
    replace_with_84(&mut s);
}
```

## 内部可变性
* 有一些类型提供了内部可变性
  * 即可以通过共享引用（不可变引用）修改值
  
* 这些类型通常依赖于额外的机制（例如原子 CPU 指令）或不变量来提供安全的可变性，而不依赖于独占引用的语义

### 内部可变性分为两类
* 1 通过共享引用获得可变引用
  * 代表类型：`Mutex` 和 `RefCell`
    * 它们提供了保障机制：如果对某个值提供了可变引用，则同时只会存在一个可变引用，并且也没有共享引用
    * 这依赖于 `UnsafeCell` 类型，这也是通过共享引用修改值的唯一正确方式
  
* 2 通过共享引用可以替换值
  * 代表类型：`std::sync::atomic` 和 `std::cell::Cell`
    * 它们没有提供可变引用到内部的值
    * 它们提供的是就地操作值的方法
    * 例如：无法直接获得到 `usize 或 i32` 的直接引用，但是可以读取和替换值


## `Cell` 类型
* 它是标准库里的
* 通过不变量的方式实现内部可变性
  * 它无法跨线程共享（所以内部值不会被并发的修改，即使通过共享引用发生修改）
  * 它也不会提供 `Cell` 到内部值的引用（所以可以一直移动它）
* 提供什么方法？：
  * 对值整体替换（就地操作）
  * 返回值的副本（读取）


# 0x05 生命周期（补充）

* 官方教程提到：
  * Rust 里每个引用都有生命周期，它就是`引用保持合法的作用域（scope）`，大多数时候是隐式和推断出来的
* 即对某个变量取得引用时生命周期就开始了，当变量移动或离开作用域时生命周期就结束了
* 其实对于生命周期：对于某个引用来说，它必须保持合法的一个代码区域的名称
  * 生命周期通常与作用域重合，但也不一定

## 借用检查器（Borrow Checker）
* 每当具有某个生命周期 `'a` 的引用被使用，借用检查器都会检查 `'a` 是否还存活
  * 追踪路径直到 `'a` 开始（获得引用）的地方
  * 从这开始，检查沿着路径是否存在冲突，保证引用指向一个可安全访问的值
  * 例子


```rust
use rand::prelude::*;
 
fn main() {
    let mut x = Box::new(42);
    let r = &x;      // (1) 'a，生命周期开始

    /*
        可以看到 'a 生命周期并没有延伸到 if 中，
        所以生命周期不一定与作用域重合
     */
    if random::<f32>() > 0.5 {
        *x = 84;   // (2) 需要 x 的可变引用，检查到这是合法的
        // println!("{}", r); // 如果这行在这就 Error，因为违反了借用规则
    } else {
        println!("{}", r); // (3) 'a，因为编译器指定 if else 只会走一个所以这里是 Ok 的
    }

}
// (4)
```

* 生命周期很复杂：也有漏洞，间歇性的失效的例子
  * 有时需要重启生命周期

```rust
fn main() {
    let mut x = Box::new(42);

    let mut z = &x;  // (1) 'a，生命周期开始，这里在获得引用的时候开始
    for i in 0..100 {           
        println!("{}", z);      // (2) 'a
        x = Box::new(1);        // (3)，其实到这第一个生命周期就结束了，因为 x 重新赋值了
        /*
            如果最后这句注释掉，上边就会报错，
            x 一直处于被借用的状态，这时对它赋值就不行了
            所以这个例子生命周期是有间歇性的失效
            然后又有重启的操作，这就是所谓的 `漏洞`
         */
        // z = &x;                 // (4) 'a，即重启了新的生命周期，所以后续循环是 Ok 的
    }
    println!("{}", z);
}
```


* 借用检查器是保守的：
  * 如果不确定某个借用是合法的，借用检查器就会拒绝这个借用
* 借用检查器有时需要帮助来理解某个借用为什么是合法的
  * 这就是 `Unsafe Rust` 存在的部分原因


## 泛型生命周期
* 有时我们需要再自己的类型里存储引用
  * 这些引用都是有生命周期的，以便借用检查器检查其合法性
  * 例如：在该类型方法中返回引用，并且存活时间比 `self` 还要长 
* Rust 允许你基于一个或多个生命周期将类型的定义泛型化

### 两点提醒
1. 如果类型实现了 `Drop`，那么丢弃这个类型时，就会被记作是使用了这个类型所泛型的生命周期或类型
  * 即这个类型的实例要被 `drop` 时，在 `drop` 之前，借用检查器会检查看是否仍然能合法的去使用你类型的泛型生命周期，因为 `drop 方法` 中的代码可能会使用到这些引用
* 如果你的类型没有实现 `Drop`，那么丢弃类型的时候就不会当做使用了生命周期，可以忽略类型里的引用

2. 类型是可以泛型多个生命周期的，但这么做通常会不必要的让类型签名更复杂
  * 只有类型包含多个引用时，你才应该使用多个生命周期参数
  * 而且这个类型方法返回的引用也应该只绑定到其中一个引用的生命周期中
  * 例子：
  
```rust
// 下面代码使用两个生命周期 's, 'p，代码没有问题
use std::path::Iter;

struct StrSplit<'s, 'p> {
    delimiiter: &'p str,   // 分隔符
    document: &'s str,     // 文档
}

// 实现 Iterator trait 
impl<'s, 'p> Iterator for StrSplit<'s, 'p> {
    type Item = &'s str;

    fn next(&mut self) -> Option<Self::Item> {
        todo!()
    }
}

fn str_before(s: &str, c: char) -> Option<&str> {
    StrSplit {
        document: s,
        delimiiter: &c.to_string(),
    }
    .next()
}


// 下面代码改为使用一个生命周期 's 代码有问题
// 因为 str_before 函数内 &c.to_string() 是在函数内创建的所以报错
// 错误为无法引用值到临时的变量，因为一个生命周期，导致它这里的生命周期最短

struct StrSplit<'s> {
    delimiiter: &'s str,   // 分隔符
    document: &'s str,     // 文档
}

// 实现 Iterator trait 
impl<'s> Iterator for StrSplit<'s> {
    type Item = &'s str;

    fn next(&mut self) -> Option<Self::Item> {
        todo!()
    }
}

fn str_before(s: &str, c: char) -> Option<&str> {
    StrSplit {
        document: s,
        delimiiter: &c.to_string(),
    }
    .next()
}
```

## 生命周期的 Variance
* Variance：即哪些类型是其他类型的 `子类`
* 什么时候 `子类` 可以替换 `超类`（vice verse）
* 通常来说：
  * 如果 A 是 B 的子类，那么 A 至少和 B 一样有用
    * Rust 例子：
      * 如果函数接收 `&'a str` 的参数，那么就可以传入 `&'static str` 的参数
      * 因为 `&'static str` 是 `&'a str` 的子类，因为 `'static` 至少跟任何 `'a` 存活的一样长
        * 因为 `'static` 是在程序运行的生命周期一直存活的，例如字符串字面值


### 三种 Variance
* 所有的类型都有 Variance
  * 这个 Variance 就定义了哪些类似的类型可以用在该类型的位置上，或说可以替代该类型

* 三种 Variance
  1. covariant（协变）：某类型只能用 `子类型` 来替代
    * 例如：`&'static T` 可替代 `&'a T`（`&'a T` 也是 `T` 的一个协变）
  2. invariant（不变）：必须提供指定的类型
    * 例如：`&mut T`，对于 `T` 来说就是 invariant
  3. contravariant（逆变）：函数对参数的要求越低，参数可发挥的作用越大
    * 例子：函数对其参数类型的逆变
    
```rust
// let x1: &'static str;       作用更大, 活的更长
// let x2: &'a      str;       作用小, 活的短

// fn take_func1(&'static str) 对参数要求比较严格，作用更小
// fn take_func2(&'s str)      对参数要求比较宽松，作用更大
```

### 多生命周期与 Variance

```rust
/*
    s 有两个生命周期
    'a 是可变的
    'b 不可变
*/
struct MutStr<'a, 'b> {
    s: &'a mut &'b str,
}

fn main() {
    let mut r: &str = "hello";     // &'statiic str => &'a str
    /*
        MutStr { s: &mut r } 是创建实例
        .s 可以修改值，所以它是 'a mut 
        &mut r 传入后生命周期就结束了，下边才可以 println

        如果只有一个生命周期，就报错了，
        只有一个生命周期 到 &mut r 其实是缩短了，因为 &mut 是赋值时候发生的
        也就相当于对 r 缩短了，但是 &'statiic str 可替换为 &'a str
        但是 &mut 是精确类型，所以这里缩短借用就会失败
        所以使用一个生命周期就会 Error
     */
    *MutStr { s: &mut r }.s = "world";
    println!("{}", r);   // 是 'b 的生命周期
}
```


# 0x06 内存中的类型
* 每个 Rust 值都有类型
  * 而一个基本职责：就是告诉你如何解释内存中的 bits（比特位）
  * 例如：`0b10111101` 这串 bit 本身不代表什么
    * 按 u8 类型解释：就是数字 189
    * 按 i8 类型解释：就是数字 -67
    
* 当自定义类型时：编译器决定该类型的各部分在内存表示中的位置


## 对齐（Alignment）
* 对齐（Alignment）：它决定了类型的字节可以被存在哪
* 实际上，计算机硬件对给定的类型可以存放的位置是有约束的
  * 例如：指针指向字节（bytes）而不是位（bits）
    * 如果将某个类型 `T` 的值放在计算机内存中索引为 4 的 bit 上，那就无法引用它的地址了
    * 因为指针必须对齐到字节（`即 8bit`），你只能创建一个指针指向 byte0 或 byte1
* 所有的值（无论什么类型），都必须开始于 `byte` 的边界
  * 必须至少是字节对齐的（byte-aligned）
  * 或者说存放的地址必须是 `8bit` 的倍数

  
### 更严格的对齐规则
* 一些类型的对齐规则比字节对齐还严格
  * 在 CPU 和内存系统里，内存经常按大于单个 byte 的块进行访问
    * 例如：在 64 位 CPU 上，大部分的值是按 `8bytes` 的块进行访问的，每个操作都开始于 `8bytes 对齐` 的地址上（这也叫 CPU 的字长，word size）
    * CPU 有办法处理更小值的读写，以及跨越块边界的值
    
* 我们应该尽可能保证硬件可以操作于它的 `原生（native）` 对齐
  * 例如：想读取的 `i64` 的值开始于 `8bytes` 块的中间，这时就需要读取两次，说明目标值分布在两个块中，第一次读取前一半，第二次读取后一半，这样是低效的


### 没对齐的操作
* 对数据进行没对齐的操作叫做：`misaligned access`
  * 会导致性能低下和并发问题
* 很多 CPU 操作多要求或强烈建议：它们的参数是自然对齐的（naturally aligned）
* 自然对齐的值：它们的对齐是匹配它们值的大小的
  * 例如：要加载 8 bytes，那么提供的地址就需要 8 bytes 对齐

### 编译器会尽可能利用对齐
* 基于类型包含的内容，编译器通过计算，为类型分配一个对齐，或分配一个对齐方案
* 对于内置的值，通常就是对齐到他们的大小
  * u8 按 1byte 对齐
  * u16 按 2bytes 对齐
  * u32 按 4bytes 对齐
  * u64 按 8bytes 对齐
* 复杂类型（包含其他类型的类型），则通常被赋予所含类型的最大对齐：
  * 例如：某个类型含有 `u8、u16、u32`，那么该类型就应该是 `4 bytes` 对齐，即按 `u32` 对齐，因为它最大


## 类型的布局（Layout）
* 类型的布局（Layout）：编译器如何决定类型在内存中的表示
* Rust 编译器对于类型如何布局，并没有给出多少保证
* Rust 提供了 `repr` 属性（attribute）：
  * 可以添加到你类型的定义上，来请求特定的内存表示
  
### repr(C)
* 最常见的就是 repr(C)，布局方式与 C、C++ 编译器对同类型的布局兼容
 * 对于使用 FFI 与其它语言交互的 Rust 代码很有用
 * 使用 FFI 与其它语言交互时，Rust 会生成一个匹配其他语言编译器期望的布局
* 因为 C 的布局是可预测、不易改变的
  * 在 `unsafe` 上下文中是非常有用的
    * 例如：使用指向该类型的原始指针时候
    * 或者在两个具有相同字段的类型间进行转换时

### repr(transparent)
* repr(transparent)：仅能用于只含有单个字段的类型，它保证了外层的布局与内层类型一样
  * 这与 `newtype 模式` 结合是很好用的
    * 例如：你想操作于 `structA` 和 `struct NewA(A)` 的内存表示，使用了 repr(transparent) 后它们的内存表示就如同是一样的，否则，编译器无法保证相同


> `newtype 模式`：即在 `tuple struct` 里创建一个新的类型，只有一个字段，针对某类型的薄的封装
{: .prompt-info }

 
![image](/assets/images/rust/type/alignment.png)

* 上图是按 `8bytes` 对齐的，因为 `u64` 需要的空间是最大的
* 不足部分需要填充，填充部分是被代码忽略的，只是为了内存对齐
 

### repr(Rust)
* 上边讲的 C 的表示是有限制的：即需要将所有字段按照原 `struct` 定义的顺序进行放置
* repr(Rust) 去掉了上边说的限制同时还去掉了几个较小的限制：
  * 对恰好具有相同字段的类型的确定性字段排序
    * 如果两个不同类型，它们的字段相同，字段类型也相同，定义顺序也一样
      * 这时在 Rust 中，如果使用 repr(Rust)，就不能保证这两种类型的布局是一样的
* repr(Rust) 允许对字段重新进行排序：
  * 可以按大小递减的顺序进行排列（对于 Foo 例子来说就不需要填充了）
* 所以对布局的保证少了，编译器有余地进行重新安排，产生高效代码

* 下图展示使用 repr(Rust)，编译器对字段排序后，会先将大的放前边，如果可见这个例子正好 16 bytes，所以就不需要填充了


![image](/assets/images/rust/type/alignment_1.png)



### 无填充的布局
* 可以告诉编译器字段之间无需任何填充：
  * 这样就必须承担不对齐访问的性能损失
  * 使用场景举例：
    1. 内存有限时，类型实例较多时
    2. 通过低带宽网络连接发送内存表示时


> 如何启用该功能？即在你的类型上添加 `#[repr(packed)]` 注解即可。注意，这可能导致代码运行速度变慢。极端情况下，如果 CPU 仅支持对齐操作，可导致程序崩溃
{: .prompt-info }


 
## 给特定字段或类型更大的对齐

* 可以使用 `#[repr(aliign(n))]` 注解：
  * 例如：保证在内存中连续（相邻）存储（就像数组）的不同值最终位于 CPU 上不同的缓存行上，就可以避免伪共享（false sharing）
    * 缓存是由缓存行组成的，缓存都是以缓存行作为一个单位来处理的，缓存行（cache line）是可以映射到缓存中的最小数据部分
    * 伪共享（false sharing）问题：即两个不同的 CPU 访问共享同一个缓存行的不同变量时，就发生了伪共享（false sharing）。理论上它们可以并行操作，但最终它们都争相更新缓存中的同一条目
    它可导致并发类程序中巨大的性能降级
  
## 复杂类型的内存表示
* 元组（Tuple）：与 struct 很像，即其 struct 字段类型与元组元素类型按顺序是相同
* 数组：它是所包含类型的连续序列，元素间并没有填充
* Union：对于每个变体来说，其布局的选择是独立的，`对齐就是所有变体里最大的那个`
* 枚举：与 Union 一样，不同点是：额外有一个隐藏的共享字段，用于存储枚举变体的鉴别符
  * 代码用鉴别符的值来判定给定值所含的到底是哪个变体
  * 鉴别符的大小取决于变体的数量


## 动态大小的类型和宽指针
* Rust 中大多数类型自动实现了 `Sized`：
  * 实现了 `Sized` 后，它们的大小在编译时就已知了
* 但两个常见的类型除外：`Trait 对象` 和 `切片(slice)`
  * 例如：`dyn Iterator、[u8]（切片）`等，它们称为动态大小的类型（DST：dynamically sized types），它们的大小在运行时才能确定

* 问题是：通常编译器需要知道类型的大小以便来产生合理的代码：
  * 例如：为 tuple 分配多大空间，或者访问第四个元素时需要多少偏移量，这些信息在编译时都需要知道，如果类型不是 `Sized`，这些信息就无法获得

### 所以编译器是需要 `Sized` 的类型的
* 几乎所有地方，编译器都需要 `Sized` 类型：
  * 包括 `struct 字段`、函数参数、返回值、变量、数组类型都必须是 `Sized` 
  * 自定义的 `Type Bound` 自动包含 `T:Sized`，除非你写明 `T:?Sized`（可能不是`Sized`）
  
* 但当函数需要接收 DST（例如`Trait 对象` 和 `切片(slice)`等）作为参数时怎么办呢？
  * 可以使用宽指针（wide pointer 或叫 fat pointer）

### 宽指针（wide pointer 或叫 fat pointer）
* 通过将`非 Sized` 类型放在宽指针后边，就可以弥补 `Sized` 和非 `Sized` 类型间的差距
* 什么是宽指针：
  * 首先它是一个普通指针
  * 附加了一个 `字大小（word-sized）` 的字段
    * 它可以提供给编译器所需要的关于指针的额外信息：生产使用该指针的合理代码

* 当引用 DST 时，编译器自动为你组建一个宽指针。例如：
  * 切片 slice，它的附加信息就是切片的长度

* 宽指针本身是实现了 `Sized`：它的大小是 `usize` 的两倍
  * `usize` 的大小就是所在目标平台上一个字（word）的大小
  * 其中一个 `usize` 用于持有指针
  * 另外一个 `usize` 用于持有附加信息，以便可以`完善`该类型


> `Box 和 Arc` 都是支持存储宽指针的，所以它们都是支持 `T:?Sized`
{: .prompt-info }


# 0x07 Trait Bounds 的编译与分配

## 静态分派（static dispatch）

### 编译 `泛型代码` 或`调用 dyn Trait 上的方法`时会发生什么呢？
* 当编写关于泛型 `T` 的类型或函数时：
  * 编译器会针对每个 `T` 的类型，都将类型或函数复制一份
  * 例如当你构建 `Vec<i32>` 或 `HashMap<String, bool>` 的时候：
    * 编译器会复制它的泛型类型以及所有的实现块
      * 例如：`Vec<i32>`，就是对 Vec 做一个完整的复制，所有遇到的 `T` 都换成 `i32`
    * 就是把每个实例的泛型参数都使用具体类型进行替换


> 编译器其实并不会做完整的复制粘贴，它只复制你用到的代码
{: .prompt-info }


```rust
impl String {
    pub fn contains(&self, p: impl Pattern) -> bool {
        p.is_contained_in(self)
    }
}
```

* 针对不同的 `Pattern` 类型，该方法都会复制一遍，Why？
  * 因为我们需要知道 `is_contained_in` 方法的地址，以便进行调用。CPU 需要知道在哪跳转和继续执行
  * 对于任何给定的 `Pattern`，编译器知道那个地址是 `Pattern` 类型实现 `Trait` 方法的地址
  * Note：这里不存在一个可给任意类型用的通用地址


* 也就是说需要为每个类型复制一个（方法体），每份都有自己的地址，用来跳转。这就是所谓的静态分派（static dispatch）：
  * 因为对于方法的任何给定副本，我们`分派到`的地址都是静态的、已知的

* 静态（static）：就是指编译时已知的事务。


## 单态化（monomorphizatiion）
* 单态化：即从一个泛型类型到多个非泛型类型的过程
* Rust 的 Trait 就有单态化的特点
* 当编译器开始优化代码时，就好像根本没有泛型
  * 每个实例都是单独优化的，具有了所有的已知类型
  * 所以上边代码例子中 `is_contained_in` 方法调用的执行效率就如同 Trait 不存在一样
  * 编译器对涉及的类型完全掌握，甚至可以将它进行 `inline` 实现


### 单态化的代价
* 所有的实例都需要单独编译，编译时间增加（如果不能优化编译）
* 每个单态化的函数都会有自己的一段机器码，让程序更大
* 而且指令在泛型方法的不同实例间是无法共享的，CPU 的指令缓存效率降低，因为它需要持有相同指令的多个不同副本


## 动态分派（dynamic dispatch）

* 动态分派：使代码可以调用泛型类型上的 `trait 方法`，而无需知道具体的类型


```rust
impl String {
    pub fn contains(&self, p: &dyn Pattern) -> bool {
        p.is_contained_in(&*self)
    }
}
```

* 此时需要调用者提供两个信息：
  * `Pattern` 地址
  * `is_contained_in` 地址
  
  
### vtable
* 实际上，调用者会提供指向一块内存的指针，它叫做虚方法表（virtual method table）或叫 `vtable`
  * `vtable` 持有上例该类型所有的 trait 方法实现的地址
    * 其中一个就是 `is_contained_in` 的地址
* 当代码想调用提供类型的一个 `trait` 方法时，就会从 `vtable` 查询 `is_contained_in` 方法的实现地址，并调用它
  * 这样就允许我们使用相同的函数体，而不关心调用者想要使用的类型

* 每个 `vtable` 还包含具体类型的布局和对齐信息（总是需要这些的）


### 对象安全（Object-Safe）
* 类型实现了一个 `trait` 和它的 `vtable` 的组合就形成了一个 `trait object`（trait 对象）
* 大部分 `trait` 可以转为 `trait object`，但不是所有的都可以：
  * 例如：`Clone trait` 就不行（`因为它的 clone 方法返回 Self`），`Extend trait` 也不行
    * 这些例子就不是对象安全的

* 对象安全的要求：
  * `trait` 所有的方法都不能是泛型的，也不可以使用 `Self`
  * `trait` 不可以拥有静态方法（因为无法知道在哪个实例上调用的方法）


### Self:Sized
* `Self:Sized` 意味着 `Self` 无法用于 `trait object`（因为它是 !Sized(非 Sized)）
* 而将 `Self:Sized` 用于某个 `trait`，就相当于要求它永远不使用动态分派
* 也可以将 `Self:Sized` 用在特定方法上，但这时当 `trait` 通过 `trait object` 访问的时候，该方法就不可用了
* 当检查 `trait` 是否是对象安全的时候，使用了 `where Self:Sized` 的方法就会被免除


### 动态分派优点
* 编译时间减少
* 提升 CPU 指令缓存效率


### 动态分派缺点
* 编译器无法对特定类型优化
  * 只能通过 `vtable` 调用函数
  
* 直接调用方法的开销增加
  * `trait object` 上的每次方法调用都需要查 `vtable`
  

## 静态分派、动态分派如何选择（一般而言的建议）

### 静态分派
* 在 `library` 中使用静态分派
  * 因为无法知道用户的需求
  * 如果使用动态分派，用户也只能如此了
  * 如果使用静态分派，用户可自行选择
  
### 动态分派
* 在 `binary` 中使用动态分派
  * `binary` 是最终代码，而不是库
  * 动态分派使代码更整洁了（省去了泛型参数）
  * 以边际性能为代价，代价比较小
  
  

# 0x08 泛型 `Trait`

## `Trait` 的泛型方式
* 两种：
1. 泛型类型参数：`trait Foo<T>`
2. 关联类型：`trait Foo { type Bar; }`

### 上边两种的区别
* 使用关联类型：对于指定类型的 `trait` 只有一种实现
* 而使用泛型类型参数：会有多个实现


> 建议：可以的话尽量使用 `关联类型`
{: .prompt-info }

## 泛型类型参数 `Trait`
* 这种形式必须指定所有的泛型类型参数，并重复写这些参数的 `Bound`
  * 这样维护较难
    * 如果添加泛型类型参数到某个 `trait`，该 `trait` 的所有用户必须都进行更新代码
* 针对给定的类型，一个 `trait` 可以存在多重实现
  * 缺点：对于你想要用的是 `trait` 的哪个实例，编译器决定起来更困难了
    * 导致有时不得不调用类似这样的函数，这样调用 `FromIterator::<u32>::from_iter` 才可消除因歧义引起的编译错误（这称为消除歧义的函数）
  * 这也是优点：
    * `impl PartialEq<BookFormat> for Book`：可以为某个类型实现多个类型，`BookFormat` 就可以是不同的类型
    * 也可同时实现 `FromIterator<T>` 和 `FromIterator<&T> where T:Clone` 
    

## 关联类型 `Trait`

```rust
trait Contains {
    type A;
    type B;

    fn contains(&self, _: &Self::A, _: &Self::B) -> bool;
}
```


* 使用关联类型
  * 编译器只需要知道实现 `Trait` 的类型
  * `Bound` 可完全位于 `Trait` 本身，不必重复使用
  * 未来再添加关联类型也不影响用户的使用
  * 具体的类型会决定 `Trait` 内关联类型的类型，无需使用消除消除歧义的函数

```rust
struct Container(i32, i32);

trait Contains {
    type A;
    type B;

    fn contains(&self, _: &Self::A, _: &Self::B) -> bool;
    fn first(&self) -> i32;
    fn last(&self) -> i32;

}

impl Contains for Container {
    type A = i32;
    type B = i32;

    fn contains(&self, num_1: &i32, num_2: &i32) -> bool {
        (&self.0 == num_1) && (&self.1 == num_2)
    }

    fn first(&self) -> i32 { self.0 }

    fn last(&self) -> i32 { self.1 }
}
```

* 但是：
  * 不可以对多个 `Target` 类型来实现 `Deref`
  * 不可以使用多个 `Item` 来实现 `Iterator`
  

# 0x09 孤儿规则与连贯性/一致性

## 连贯性/一致性属性
* 定义：对于给定的类型和方法，只会有一个正确的选择，用于该方法对该类型的实现
* 孤儿原则（orphan rule）：
  * 只要 `Trait` 或者类型在你本地的 `crate`，那么就可以为该类型实现该 `Trait`
    * 例如：可以为你的类型实现 `Debug`，可以为 `bool` 实现 `MyTrait`
    * 但是不能为 `bool` 实现 `Debug`，因为这俩类型都不在你本地 `crate` 中
* Note：也有其他的注意事项、例外


## Blanket Implementation
* `impl<T> MyTrait for T where T:`
  * 例如：`impl<T:Display> ToString for T {}`，要求 `T` 实现了 `Display`，然后为 `T` 实现 `ToString trait`
* 优点是不局限于一个特定的类型，而是应用于更广泛的类型


> 只有定义 `trait` 的 `crate` 允许使用 `Blanket Implementation`。而添加 `Blanket Implementation` 到现有 `trait` 属于破坏性变化。
{: .prompt-info }



## 基础类型
* 有些类型太基础了，需要允许任何人在它们上实现 `trait`（即使违法孤儿原则）
* 这些类型被标记了 `#[fundamental]`，目前包括 `&`、`&mut`、`Box`
  * 出于孤儿规则的目的，在孤儿规则检查前，它们是会被抹除掉的
* 而对于基础类型使用 `Blanket Implementation` 也被认为是破坏性变化


## Covered Implementation
* 有时候需要为外部类型实现外部 `trait`
  * 例如：`impl From<MyType> for Vec<i32>` 为 Vec<i32> 实现 From trait，（为外来类型实现外来 `trait`）
* 孤儿规则制定了一个狭窄的豁免：
  * 允许在非常特定的情况下为外来类型实现外来 `trait`

* 形式 `impl<Pi..=Pn> ForeignTrait<Ti..=Tn> for T0` 只在以下条件被允许：
  * 至少有一个 `Ti` 是本地的类型
  * 并且没有 `T` 在第一个这样的 `Ti` 之前（T 是指泛型类型 Pi..=Pn 中的一个）
  * 而泛型类型参数 `Ps` 允许出现在 `T0..Ti` 中，只要它们被某种中间(intermediate)类型所 `cover`
* 如果 `T` 作为其他类型（例 `Vec<T>`）的类型参数出现，那就说 `T` 被 `cover` 了
* 而 `T` 只作为本身，或者位于基础类型后（`&`、`&mut`、`Box`），就不是 `cover` 的 


### Ok 的例子
* `impl<T> From<T> for MyType`
* `impl<T> From<T> for MyType<T>`
* `impl<T> From<MyType> for Vec<T>`
* `impl<T> ForeignTrait<MyType,T> for Vec<T>`


### `Not Ok` 的例子
* `impl<T> ForeignTrait for T`
* `impl<T> From<T> for T`
* `impl<T> From<Vec<T>> for T`
* `impl<T> From<MyType<T>> for T`
* `impl<T> From<T> for Vec<T>`
* `impl<T> ForeignTrait<T, MyType> for Vec<T>`


### Covered Implementation 是否有破坏性变化？答案是：看情况
* 如果为现有 `trait` 添加新的实现，并且至少包含一个新的本地类型，该本地类型满足豁免条件，这时就是`非破坏性的变化`
* 如果为现有 `trait` 添加的实现不满足上述要求，就是破坏性变化

* Note：
  * `impl<T> ForeignTrait<LocalType, T> for ForeignType`，是合法的
  * `impl<T> ForeignTrait<T, LocalType> for ForeignType`，是非法的


# 关于类型其他知识
* `Trait Bound` 的各种高级写法
* `Marker Trait`（标记 trait）, `Marker Type`（标记类型）
* `Existential Type`（存在类型）



