---
layout: post
title: RustGuide-02-Ownership
date: 2023-03-11 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 所有权（Rust Core）
* 所有权是 Rust 最独特的特性，它让 Rust 无需 GC 就可以保证内存安全
* 所有权是确保 Rust 程序安全的一种机制
    - 安全：即程序中没有未定义的行为
    - 未定义的行为：当执行一段代码时，结果不可预测且未被编程语言指定的情况
* Rust 的一个基础目标：是确保你的程序永远不会有未定义的行为
* Rust 的一个次要目标：是在编译时而不是运行时防止未定义的行为


## 什么是所有权
* 所有程序在运行时都必须管理它们使用计算机内存的方式
1. 有些语言有垃圾回收机制，在程序运行时它们会不断寻找不用的内存
2. 有些语言程序员需要显示的分配和释放内存

* 而 Rust 使用第三种方式
1. 内存是通过所有权系统管理的，其中包含一组编译器在编译时检查的规则
2. 在程序运行时所有权特性不会减慢程序的运行速度

## Stack（栈内存）VS Heap（堆内存）
* 一个值在 stack 上还是 heap 上对语言的行为和你要做某些决定是有很大影响的

## Stack 栈内存
* 按值的接收顺序存储，按相反的顺序将它们移除（后进先出，LIFO）
1. 添加数据叫压入栈
2. 移除数据叫弹出栈

* 所有存储在 stack 上的数据必须拥有已知的固定的大小

## Heap 堆内存
* 编译时大小未知的数据或运行时大小可能发生变化的数据必须存放在 heap 上
* heap 的内存组织性差一些，当你把数据放入 heap 时，你会请求一定数量的空间
* 操作系统在 heap 中找到一块足够大的空间，把它标记为已用，然后返回一个指针，即这个空间的地址，
这个过程就叫在 heap 上进行分配


![image](/assets/images/rust/heap.png)


> `a` 最开始有 Box 的所有权；将 `a` 赋值给 `b` 后，发生了 `move`, `a` 就失效了, `b` 获得了 Box 的所有权
{: .prompt-info }


## 移动堆（heap）数据原则：
* 如果变量 `x` 将堆（heap）数据的所有权移动给另一个变量 `y`，那么在移动后，`x` 就不能再使用了。
* 避免数据移动的一种方法是使用 `.clone()` 方法进行克隆


## Stack VS Heap

* 因为指针是已知的固定的，所以可以把指针放在 stack 上
1. 但如果想要实际数据，就必须使用指针来定位它

* 把数据压到 stack 上要比 heap 上分配速度快的多
1. 因为系统不需要寻找用来存储新数据的空间，这个位置永远在栈顶

* 在 heap 上分配空间需要更多的操作
1. 首先系统需要找到一个足够大的空间来存储数据，然后做好记录方便下次分配

* 访问 heap 的数据要比访问 stack 上的数据慢的多，因为需要指针才能找到 heap 中的数据
1. 属于间接访问，多了指针跳转操作
2. 现代操作系统的处理器，由于缓存的缘故，如果指令在内存中跳转次数越少，则速度越快

* 如果数据存放的距离比较近，那么处理器处理的速度就会快一些（比如 stack）
* 如果数据存放的距离比较远，那么处理器处理的速度就会慢一些（比如 heap）
1. 在 heap 上分配大量空间也是需要时间的

## 函数调用
* 当调用函数时，被传入到函数（也包括指向 heap 的指针）。函数本地变量被压入到 stack，当函数结束这些变量会从 stack 弹出。
* Rust 不允许手动管理内存
* `Stack Frame` 由 Rust 自动管理：当调用一个函数时，Rust 为被调用的函数分配一个 `Stack Frame`。当调用结束时，Rust 释放该 `Stack Frame`。


## Box 的所有者来管理内存释放
* Rust 会自动释放 Box 的堆（heap）内存


> `Box` 内存释放原则（完全正确）：如果一个变量`拥有`一个 `Box`，当 Rust 释放变量的 `frame` 时，Rust 也会释放 `Box` 的堆（heap）内存
{: .prompt-info }

![image](/assets/images/rust/box_heap.png)


## 所有权能解决的问题
* 跟踪代码那些部分正在使用 heap 的哪些数据
* 最小化 heap 上的重复数据量
* 清理 heap 上未使用的数据，以避免空间不足
* 一旦懂得了所有权就不需要去想 stack 和 heap 了
* 管理 heap 数据才是所有权存在的原因，这有助于解释它为什么这么做

## 所有权的规则
1. 每个值都有一个变量，这个变量是该值的所有者
2. 每个值同时只能有一个所有者
3. 当所有者超出作用域（scope）时，这个值将被删除

## 变量的作用域（scope）
* scope 就是程序中一个项目的有效范围

```rust
{
// 这行 s 不可用 
let s = hello; // s 开始可以用了
    // 可对 s 进行相关操作
} // s 的作用域结束，s 不再可用
```

### String 类型与作用域
* String 存在 heap 上
* 字符串字面量是不可变的

#### 创建 String 
* 使用 from 从字面量创建
1. :: 表示 from 是 String 类型下的一个函数

```rust
    let mut s = String::from("hello");
    s.push_str(", World!");
    println!("{}", s);
```

* 字符串字面量不能改而 String 可以改是因为它们处理内存方式不同

### 内存与分配
* 字符串字面量在编译时就知道它的内容了，其内容直接被硬编码到最终的可执行文件中
1. 速度快，高效
2. 不可变

* String 类型，为了支持可变性，需要在 heap 上分配内存来保存编译时未知的文本内容
* Rust 中对于某个值，当拥有它的变量走出了作用域，内存会立即交还给系统，即内存立即释放 


> 当变量离开作用域，Rust 会调用 drop 函数
{: .prompt-info }

## 变量与数据交互方式：移动（Move）
* 多个变量可以与同一个数据使用一种独特的方式来交互
* Case 1

```rust
let x = 5;
let y = x;
println!("{} {}", x, y); // 此时没有 Move，x 依然可访问
//整数是固定的已知值简单值，这里两个 5 被压入到 stack 中
```

* Case 2，String 版本

![image](/assets/images/rust/move_str.png)

Rust 没有使用这种方式
![image](/assets/images/rust/move_str1.png)

Rust 使用的方式
![image](/assets/images/rust/move_str2.png)

```rust
let mut s1 = String::from("hello");
let s2 = s1;
// 因为 s1 已经把所有权给了 s2，这里不能在访问 s1
```

### 浅拷贝（shallow copy）和深拷贝（deep copy）

![image](/assets/images/rust/move_str3.png)

> 你会认为复制指针、长度、容量视为浅拷贝，但由于 Rust 让 s1 失效了，所以我们用一个新的术语叫 Move，由于只有 s2 是有效的，所以只有在 s2 离开作用域时才会释放内存，这样就避免了二次释放的问题，Rust 不会自动创建数据的深拷贝，就运行时性能而言，任何自动赋值都是廉价的
{: .prompt-info }


> 可以认为浅拷贝 heap 上的数据会进行 Move
{: .prompt-info }

### 克隆 Clone
* 如果想对 heap 上的 String 数据进行深度拷贝，可以使用 clone 这个方法

```rust
let mut s1 = String::from("hello");
let s2 = s1.clone();
println!("s1 is {}, s2 is {}", s1, s2); // 深拷贝，可以访问 s1
```
![image](/assets/images/rust/clone.png)

### Stack 上的数据：复制

```rust
let x = 5;
let y = x;
println!("{} {}", x, y); // 此时没有 Move，x 依然可访问
```

> 标量类型深拷贝和浅拷贝没有区别，它们不会 Move。所有權的 move 可理解為：淺拷貝+之前的變量失效
{: .prompt-info }


* Copy trait，可以用于像整数这样完全存放在 stack 上面的类型
* 如果一个类型实现了 Copy trait，那么旧的变量在赋值后仍然可用
* 如果一个类型或该类型的一部分实现了 Drop trait，那么 Rust 不允许它再去实现 Copy trait 了


### 哪些类型实现了 Copy trait
* 任何简单的标量的组合类型都可以实现 Copy trait 的
* 任何需要分配内存或某种资源的都不可实现 Copy trait

* 可简单理解为 Copy trait 和 drop trait 互斥
1. 棧類型實現 copy trait
2. 堆類型可實現 drop trait

* 拥有 Copy trait 的类型
1. 所有的整数类型，例如 u32
2. bool
3. char
4. 所有的浮点类型，例如 f64
5. Tuple（元组），如果其所有字段都是 Copy 的，那么这个 Tuple 是拥有 Copy trait 的，如 (i32,i32) 就是，(i32, String) 则不是

## 所有权与函数
* 在语义上，将值传递给函数和把值赋值给变量是类似的

> 将值传递给函数要么发生移动（move），要么发生复制（copy）
{: .prompt-info }

```rust
fn main() {
    let s = String::from("Hello World");

    // 这里 s 已经 move 到函数中了
    take_ownership(s);

    //println!("s is {}", s); // 由于 take_ownership 调用， s 已经 Move 了，所以 s 不再可用

    let x = 5;

    makes_copy(x);

    println!("x is {}", x); // x 依然可用，因为 makes_copy 调用只是把 x 是标量类型，是把副本传给函数
}

fn take_ownership(string: String) {
    println!("string is {}", string); // 传进来的 s 在这里可用，在函数结束 drop
}

fn makes_copy(number: i32) {
    println!("num is {}", number);
}
```

### 返回值与作用域
* 函数在返回值的过程中同样也会发生所有权的转移

```rust
fn main() {
    // 将 str 的 hello 移动给了 s1
    let s1 = gives_ownership();

    let s2 = String::from("hello");

    // 这里其实是 s2 Move 给了 s3
    let s3 = takes_and_gives_back(s2);

    // 作用域结束 s1、s3 drop，s2 因为已经 Move 了，所以不会有什么变化
}

fn gives_ownership() -> String {
    let str = String::from("hello");
    str
}

fn takes_and_gives_back(a_string: String) -> String {
    a_string
}
```
* 一个变量的所有权总是遵循同样的模式
1. 把一个值赋值给其他变量时就会移动
2. 当一个包含 heap 数据的变量离开作用域时，它的值就会被 drop 清理，除非数据的所有权移动到了另一个变量上（即 heap 上的数据会被 drop）

#### 如何让函数使用某个值，而不获得其所有权呢？
* 方式一，可以函数再把变量值 return 出来，但这种方式比较麻烦
* Rust 有个特性叫引用，即 Reference

## 引用和借用
* 以 String 为例，参数类型是 &String 而不是 String
1. & 符号就表示引用，允许你引用某些值而不取得其所有权
2. s 就是 s1 引用，s 就是一个指向 s1 的指针
3. 引用使用 `&`
4. 解引用使用 `*`（以便可以访问数据）


![image](/assets/images/rust/reference.png)


* 把引用作为函数参数这种方式就叫借用

```rust
fn main() {
    let s1 = String::from("hello");
    /*
        此处只是创建一个 s1 的引用传入了参数，并不拥有 s1，并没有转移所有权
        所以当这个引入走出作用域，并不会把 s1 清理掉
     */
    let len = get_length(&s1);
    println!("s1 = {}, len = {} ", s1, len);
}

fn get_length(str: &String) -> usize {
    /*
    当一个函数的参数是引用时，函数作用域结束，不需要考虑所有权，因为引用没有获得数据的所有权
     */

    str.push_str("world"); // 这里报错，不能修改，因为引用的变量是不可变的
    str.len() // 这里因为是引用，没有所有权，所以走出作用域什么不会做
}
```

### 是否可以修改借用的东西呢 ？
* 答案是不行的，因为引用的变量是不可变的
* 那么如何更改呢 ？方式如下：添加 mut 关键字

```rust
fn main() {
    // 这里加 mut 变为可变变量
    let mut s1 = String::from("hello");

    let len = get_length(&mut s1);

    println!("s1 = {}, len = {} ", s1, len);
}

fn get_length(str: &mut String) -> usize {

    str.push_str("world"); // 由于是可变引用所以这里，不会报错
    str.len()
}
```

### 引用是没有`所有权`的指针


```rust
fn main() {
    let m1 = String::from("hello");
    let m2 = String::from("world");
    println!("m1:{}", m1);
    println!("m1 的地址:{:p}", &m1);
    greet(&m1, &m2);
    let s = format!("{} {}", m1, m2);
}

fn greet(g1: &String, g2: &String) {
    println!("g1:{}, g2:{}!", g1, g2);
    let address_in_g1 = g1 as *const String;
    println!("g1 value:{g1}");
    println!("g1 存的内容:{:p}", address_in_g1);
    println!("g1的地址:{:p}", &g1);
}

//m1:hello
//m1 的地址:0x16db9aa00
//g1:hello, g2:world!
//g1 value:hello
//g1 存的内容:0x16db9aa00
//g1的地址:0x16db9a780

```


### 可变引用

> 可变引用有个重要限制一：在特定作用域内，对某一块数据，只能有一个可变的引用。
{: .prompt-info }


```rust
    let mut s = String::from("hello");

    let s1 = &mut s;
    let s2 = &mut s; // 这里报错，因为这个作用域 s 可变引用个数不能超过一个

    println!("s1 {}, {}", s1, s2);
```

> Note：这个限制的好处是在编译时就能防止数据竞争。
{: .prompt-info }


#### 以下三种行为会发生数据竞争
1. 两个或多个指针同时访问同一数据
2. 至少有一个指针用于写入数据
3. 没有使用任何机制来同步对数据的访问



* 我们可以通过创建新的作用域，来允许非同时的创建多个可变引用

```rust
    let mut s = String::from("hello");
    {
        let s1 = &mut s;  
    }

    let s2 = &mut s; // s1 在块作用域中，与 s2 不冲突，即它们不是同时的

    println!("s1 {} ", s2);
```


> 可变引用另外限制：不可以同时拥有一个可变引用和不可变引用，因为它们是冲突的。多个不可变引用是可以的
{: .prompt-info }

```rust
    let mut s = String::from("hello");
    let s1 = &s;
    let s2 = &s;
    let s3 = &mut s; // 这里报错，因为 s 已经借用为不可变的引用了

    println!("s1 {}, s2 {}, s3 {}", s1, s2, s3);
```

### 悬空引用（Dangling References）
* 悬空指针（Dangling Pointer）：即一个指针引用了内存的某个地址，而这块内存已经被释放了并分配给了其他数据使用了。
* 在 Rust 里，编译器可以保证引用永远不会变为悬空引用
1. 如果你引用了某些数据，编译器会保证在引用离开作用域之前数据不会离开作用域

```rust
fn main() {
    let s = dangle();
}

// 这个函数会返回一个悬空指针，但 Rust 中编译就会报错
fn dangle() -> &String {
    let mut s = String::from("hello");
    &s
}
```


### 小结：引用的规则
1. 在任何给定的时刻，只能满足下列条件之一
* 一个可变的引用
* 任意数量不可变引用
2. 引用必须一直有效才行



### 引用补充
* 解引用指针以访问数据


![image](/assets/images/rust/jieyinyong.png)


> 上图中的 `r2` 的 `&*` 相当于相互抵消了，最终 `r2` 的类型为 `&i32`，因为 `Box<i32>` 就是一个指针
{: .prompt-info }


* 隐式解引用

![image](/assets/images/rust/jieyinyongauto.png)


### 失效的例子

![image](/assets/images/rust/move.png)

> L2 将 4 push 到 v 中，导致 v 的内存重新分配后，num 指向地址就失效了所以后边再想使用 num 进行打印时就会报错
{: .prompt-info }


### 指针安全原则

* 别名和可变性不可同时存在。别名即：通过不同的变量访问同一数据。

#### 数据不应同时“被别名引用(不可变引用)”和具有可变性

* 1 `Box`（有`所有权`的指针）：不能别名。将一个 `Box` 变量赋值给另一个变量就会 `移动所有权`。也就说不能让多个 `Box` 同时拥有一块数据
    
```rust
fn main() {

    let x = Box::new(5);
    let y = x;
    println!("{}", x); // 已经 move，不能让多个 `Box` 同时拥有一块数据
    println!("{}", y);

    let r1 = &y;  // 多个引用可以同时指向同一个 Box
    let r2 = &y;
    println!("r1: {r1}, r2: {r2}");
}
```

* 2 引用（无所有权的指针）：旨在临时创建别名


##### Rust 通过借用检查器确保引用的安全性

* 变量对其数据有三种权限：这些权限在运行时并不存在，仅在编译器内部存在
1. 读（R）：数据可以被复制到另一个位置
2. 写（W）：数据可以被修改
3. 拥有（O）：数据可以被移动或释放

* 默认情况下，变量对其数据具有读/拥有权限（RO）
* 如果一个变量被标注为 `let mut`，那么它还具有写权限（W）


> 注意 注意 注意：引用可以临时移除这些权限
{: .prompt-info }


![image](/assets/images/rust/ro.png)


```rust
fn main() {
    let x = 0;
    let mut x_ref = &x;
    let y = 1;

    *x_ref += 1; // 编译报错: 因为其借用的是不可变的
    x_ref = &y;  // 编译通过: 因为 x_ref 是可变的可以指向其他 &i32 地址
}
```

### 权限
* 权限是定义在位置上的（不仅仅是变量）
* 位置：即任何可以放在赋值语句左侧的东西。
    - 例如，变量 `a`
    - 位置解引用，例如 `*a`
    - 位置的数组访问，例如 `a[0]`
    - 位置的字段访问，例如 `a.0`（针对元组）或 `a.field`（针对结构体）


* 为什么当“位置”变成不使用的时候，它们会失去权限？
    - 有些权限是互斥的


![image](/assets/images/rust/place.png)


> 上图：当 `num` 还在使用时， `num` 不能被释放或修改否则就会有问题。
{: .prompt-info }


```rust
fn main() {
    // compile error sample
    let mut v: Vec<i32> = vec![1, 2, 3];
    let num: &i32 = &v[2];
    v.push(4);         // error: 因为下边还在使用 num
    println!("num is {:?}", *num);
}
```

### 可变引用提供对数据`唯一的`并且`非拥有的`访问
* 不可变引用（共享引用）：只读的
* 可变引用（独占引用）：在不移动数据的情况下，临时提供可变访问。`独占引用`就是上边说的`唯一的`


![image](/assets/images/rust/ro1.png)

* `num` 没有写权限，就是说它永远只能指向 `v[2]`
* 当 打印完 `num` 后其权限就回收了，然后 `v` 获得了 `RWO` 权限


### 可以变引用可以临时降级为只读引用


![image](/assets/images/rust/down.png)

* `*num` 就是 `v[2]`，而 `&*num` 就相当于引用的是 `&v[2]`，`num2` 是不可变的引用
* 上图第三行代码走完时，`*num` 就被降级为只读引用了


```rust
fn main() {
    let mut v: Vec<i32> = vec![1, 2, 3];
    let num: &mut i32 = &mut v[2];
    let num2 = &*num;
    println!("*num is {}, *num2 is {}", *num, *num2);
}
```

### 权限在引用生命周期结束时被返回

![image](/assets/images/rust/life.png)


![image](/assets/images/rust/if_else.png)


### 数据必须在其所有的引用存在的期间存活


### 流动权限 F（flow）：函数里的表达式，在表达式使用输入引用或返回输出引用时都需要`流动权限 F`

![image](/assets/images/rust/flow.png)

* 上图函数中，第一行为`表达式使用输入引用`,第二行为`表达式返回输出引用`,它们需要流动权限
* `F` 权限在函数体内不会发生变化


> 如果一个引用被允许在特定的表达式中使用（即流动），那么它就具有 F 权限
{: .prompt-info }


![image](/assets/images/rust/flow1.png)

* 上图 `main` 中可能 s 引用了 `default` 下边将 `default` 进行 `drop` 有可能有问题


> Rust 不管函数如何实现，Rust 只看函数签名。上图会报错，需要引用标识生命周期 
{: .prompt-info }


#### 另一个例子

![image](/assets/images/rust/flow1.png)


* 最后返回值不具有 `F 权限`，因为这个函数走完，`S` 就被释放了


## 修复所有权常见的错误

### 1 Fixing an Unsafe Program: Returning a Reference to the Stack

```rust
fn return_a_string() -> &String {
    let s = String::from("hello");
    &s
}

// 方案一：
fn return_a_string() -> String {
    let s = String::from("hello");
    s
}

// 方案二：&'static 会随程序运行一直存在，活的最长，除非程序嘎了
fn return_a_string() -> &'static str {
    "Hello, world!"
}

// 方案三：可对引用计数的指针
use std::rc::Rc;
fn return_a_string() -> Rc<String> {
    let s = Rc::new(String::from("hello"));
    Rc::clone(&s)
}

// 方案四：不推荐
fn return_a_string(output: &mut String) {
    // 将整个字符串替换为 "Hello World"
    output.replace_range(.., "Hello World");
}
```


### 2 Fixing an Unsafe Program: Not Enough Permissions (权限不足)

```rust
// Question
fn stringify_name_with_title(name: &Vec<String>) -> String {
    name.push(String::from("Esq."));  // 没有写的权限，所以报错
    let full = name.join(" ");
    full
}



// 方案一：使用 clone() 解决, 此方案弊端是内存有些浪费
fn main() {
    let name = vec![String::from("Ferris")];
    let first = &name[0];
    let full = stringify_name_with_title(&name);
    println!("{}", first);
    println!("{}", full);
    println!("{:?}", name);

}
fn stringify_name_with_title(name: &Vec<String>) -> String {
    let mut name_clone = name.clone();   
    name_clone.push(String::from("Esq."));  
    let full = name_clone.join(" ");
    full
}


// 方案二：使用 join() 解决
fn stringify_name_with_title(name: &Vec<String>) -> String {
    let mut full = name.join(" "); // join 返回一个新 String
    full.push_str("Esq.");
    full
}
```

### 3 Fixing an Unsafe Program: Aliasing and Mutating a Data Structure (同时启用别名和可变性)

```rust
// Question
fn add_big_strings(dst: &mut Vec<String>, src: &[String]) {
    // 本来 dst 是可变引用，由于下边行走完就有了一个引用指向 dst 里的元素，导致 dst 失去了写的权限
    // ，失去了可变性
    let largest = dst.iter().max_by_key(|s| s.len()).unwrap();

    for s in src {
        if s.len() > largest.len() {
            dst.push(s.clone());
        }
    }
}

// 方案一: 使用 clone() 解决，弊端是 clone 涉及浪费内存
fn add_big_strings(dst: &mut Vec<String>, src: &[String]) {
    // 本来 dst 是可变引用，由于下边行走完就有了一个引用指向 dst 里的元素，导致 dst 失去了写的权限
    // ，失去了可变性
    let largest = dst.iter().max_by_key(|s| s.len()).unwrap().clone();

    for s in src {
        if s.len() > largest.len() {
            dst.push(s.clone());
        }
    }
}


// 方案二: 创建新的 Vec，相较于方案一更浪费内存 
fn add_big_strings(dst: &mut Vec<String>, src: &[String]) {
    // 本来 dst 是可变引用，由于下边行走完就有了一个引用指向 dst 里的元素，导致 dst 失去了写的权限
    // ，失去了可变性
    let largest = dst.iter().max_by_key(|s| s.len()).unwrap();

    let to_add = src.iter()
        .filter(|s| s.len() > largest.len())
        .cloned().collect();

    dst.push(to_add);


}

// 方案三: 使用 len()
fn add_big_strings(dst: &mut Vec<String>, src: &[String]) {
    // 本来 dst 是可变引用，由于下边行走完就有了一个引用指向 dst 里的元素，导致 dst 失去了写的权限
    // ，失去了可变性
    let largest_len = dst.iter().max_by_key(|s| s.len()).unwrap().len();

    for s in src {
        if s.len() > largest_len {
            dst.push(s.clone());
        }
    }

}
```

### 4 Fixing an Unsafe Program: Copying vs. Moving Out of a Collection（从一个集合 Copy 一个元素，或叫移除一个元素的所有权）


```rust
// Question
fn main() {
    // Copy Trait
    let v: Vec<i32> = vec![0, 1, 2];
    let n_ref: &i32= &v[0];
    let n: i32 = *n_ref; // 由于是 i32 类型，这里会发生 Copy

    // Move
    let v: Vec<String> = vec![String::from("Hello world")];
    let n_ref: &String= &v[0];
    /*
        下边 error: 无法移动, 引用是没有所有权的, 想通过解引用获得所有权是不行的
        下边发生的实际是想要获得所有权，而引用是没有所有权的
     */
    let n: String = *n_ref;
}
```


* 如果一个值不拥有堆数据，那么它可以在不移动的情况下被复制
    - 一个 `i32` 不拥有堆数据，因此可以在不移动的情况下被复制
    - 一个 `String` 拥有堆数据，因此`不能`在不移动的情况下被复制
    - 一个 `&String` 不拥有堆数据，因此可以在不移动的情况下被复制

* 修复方案: 使用 `clone()` 解决

```rust
fn main() {
    // Copy Trait
    let v: Vec<i32> = vec![0, 1, 2];
    let n_ref: &i32= &v[0];
    let n: i32 = *n_ref; // 由于是 i32 类型，这里会发生 Copy

    // Move
    let v: Vec<String> = vec![String::from("Hello world")];
    let n_ref: String= v[0].clone(); // 使用 clone() 解决

}
```

* 5 Fixing an Unsafe Program: Mutating Different Tuple Fields（修改 Tuple 不同字段）

```rust
// 没有问题的 case

fn main() {
    let mut name = (String::from("Ferris"), String::from("Rustacean"));
    // 下边这句走完 name 对第一个元素就不具备写权限了，同时 name 失去了对整个 tuple 的写权限
    let first = &name.0;
    name.1.push_str(", Esq."); // 没有问题，但 name.1 依然有写权限
    println!("{first}, {}", name.1);
}
```


```rust
// 有问题的 case

fn main() {
    let mut name = (String::from("Ferris"), String::from("Rustacean"));
    let first = get_first(&name);
    // 报错：rust 只看方法签名，get_first 函数返回 &String，所以 rust 认为 name.0 和 name.1 都被不可变借用
    name.1.push_str(", Esq.");
    println!("{first}, {}", name.1);
}

fn get_first(name: &(String, String)) -> &String {
    &name.0
}
```

* 6 Fixing an Unsafe Program: Mutating Different Array Elements


```rust
// 报错 case
fn main() {
    let mut a = [0, 1, 2, 3];
    let x = &mut a[1]; // 可变借用，这行走完，a 失去所有权限，只有 x 有权限

    *x += 1;

    let y = &a[2];     // 不可变借用，由于 a 没有任何权限了，所以这里也不可能给予读权限，所以报错
    *x += *y;          // 可变借用持续到这行结束，所以在此之前不能有不可变借用

    println!("{a:?}");
}
```


```rust
// 修复方案
fn main() {
    let mut a = [0, 1, 2, 3];

    let y = a[2];

    let x = &mut a[1];  // 可变借用，这行走完，a 失去所有权限，只有 x 有权限

    *x += y;             // 可变借用持续到这行结束，所以在此之前不能有不可变借用

    println!("{a:?}");
}
```



## 切片
Rust 提供了一种不持有所有权的数据类型：切片（slice）

编写一道题
* 它接收字符串作为参数
* 返回它在这个字符串里找到的第一个单词
* 如果函数没找到任何空格，那么整个字符串就被返回了


```rust
fn main() {

    let mut s = String::from("Hello World");
    let wordIdx = first_word(&s);

    // s.clear(); 如果这里将字符串清空，wordIdx 就没有用了
    println!("space index is {}", wordIdx);
}

fn first_word(s: &String) -> usize {
    // 得到一个字节数组, 类型是 &[u8]
    let bytes = s.as_bytes();

    /*
        iter() 为其创建一个迭代器, 这个方法依次返回集合中的每个元素
        enumerate() 会把 iter() 返回的结果进行包装，并把每个结果作为元组的一部分进行返回
        i 就是索引
        &item 就是元素，这里就是字符串里的字节，注意 这里它是一个引用
     */
    for (i, &item) in bytes.iter().enumerate() {
        // b' ' 就是空格字节
        if item == b' ' {
            return i;
        }
    }
    s.len()
}
```

如果 s.clear() 将字符串清空，wordIdx 就没有用了，为此引入了切片

### 字符串切片
* 字符串切片是指向字符串中一部分内容的引用
* 格式：&变量名[startIndex，endIndex]
* 范围是 [startIndex, endIndex)
* startIndex 就是切片起始位置的索引
* endIndex 是切片终止位置的下一个索引
* world 就是一个切片，只有一个指针（ptr）和一个长度（len） 

> 下图的错误，s 的 len 应该是 11
{: .prompt-info }

![image](/assets/images/rust/slice.png)

* 切片的语法糖
1. startIndex 如果是 0 可以不写，即 [..5] 等于 [0..5]
2. endIndex 如果是字符串的 length 可以不写，即 [4..] 等于 [4..str.len()]
3. 整个字符串的切片为 &s[..]，即 startIndex，endIndex 都可以不写

> 字符串切片的范围索引必须发生在有效的 UTF-8 字符边界内，如果尝试从一个多字节的字符中创建字符串切片，程序会报错并推出
{: .prompt-info }


> 字符串切片的类型是 &str，&str 是不可变引用，所以字符串字面量也是不可变的
{: .prompt-info }


* 使用字符串切片重解写上边的问题
```rust
// &str 表示字符串切片类型
fn first_word2(s: &String) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[..i]
        }
    }
    &s[..]
}
```

* 字符串字面量就是是字符串切片
1. 字符串字面量被直接存储在二进制程序中
2. let s1 = "hello"; s2 的类型是 &str，它是一个指向二进制程序特定位置的切片，是不可变的

* 将字符串切片作为参数传递
1. 有经验的 Rust 开发者会采用 &str 作为参数类型，因为这样就可以同时接收 String 和 &str 类型的参数了

```rust
fn first_word1(s: &String) -> &str {

}

// 转为
fn first_word2(s: &str) -> &str {

}
// 如果传入就是字符串切片，直接调用该函数
// 如果传入的是 String 类型，可以创建一个完整的 String 切片来调用该函数
```

> 定义函数时使用字符串切片来代替字符串引用会使我们的 API 更加通用，并且不会损失任何功能
{: .prompt-info }


```rust
fn main() {
    let w = first_word(&s[..]);
    
    let s1 = "hello";
    let w1 = first_word(s1);
    println!("w is {}, w1 is {}", w, w1);
}

fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[..i]
        }
    }
    &s[..]
}
```

* Slice 是特殊的引用类型，因为他们是 `fat` 指针，带有元数据

![image](/assets/images/rust/slice_str.png)


![image](/assets/images/rust/slice_o.png)



* 思考题：下边代码能否通过编译?

```rust
fn main() {
    let mut s = String::from("hello");
    for &item in s.as_bytes().iter() { // 这里借用为不可变引用
        if item == b'l' {
            s.push_str(" world");  // 报错：这里又借用为可变引用
        }
    }
    println!("{}", s);
}
```


### 其他类型的切片
* 数组切片
```rust
let a = [1, 2, 3, 4, 5];
let sliceA = &a[1..3];  // sliceA 的类型是 &[i32]，存储了一个指针和一个长度
```


