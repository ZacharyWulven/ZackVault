---
layout: post
title: Rust 中级教程-接口设计建议-02-Flexible
date: 2023-10-04 16:45:30.000000000 +09:00
categories: [Rust, Rustaceans]
tags: [Rust, Rustaceans]
---


# 3 灵活

* 你写的代码都隐私的包含一种契约，契约（Contract）：即
1. 要求：代码使用的限制
2. 承诺：代码使用的保证

## 3.1 设计接口时的经验法则
* 避免施加不必要的限制，只做能够兑现的承诺
* 增加限制或取消承诺，到后来可能取消某个承诺，导致重大语义版本更改，导致其他代码出问题
* 而设计接口时最初限制放宽些或提供额外的承诺，后续再加，通常是可以向后兼容的


## 3.2 限制（Restrictions）和承诺（Promises）
* Rust 中的限制常见形式
1. `Trait` 约束
2. 参数类型

* Rust 中承诺的常见形式
1.`Trait` 的实现
2. 函数和方法的返回类型


* 例1：`fn frobnicate1(s: String) -> String` 
1. 这个函数的契约是调用者进行内存分配，承诺返回拥有的 String，这样以后无法改为 `无需内侧分配` 的函数，因为其参数和返回值是拥有所有权的
2. 即这个函数如果签名不改，则里边一定涉及内存分配

* 例2：`放宽了契约`，签名为 `fn frobnicate2(s: &str) -> Cow<'_, str>`
1. 这个函数只接收字符串切片，承诺返回字符串的引用或一个拥有的 String
2. 虽然契约放宽了，但比较死板

* 例3：`进一步放宽契约`，`fn frobnicate3(s: impl AsRef<str>) -> impl AsRef<str>`
1. 要求传入能产生字符串引用的类型，承诺返回值可产生字符串引用


```rust
use std::borrow::Cow;

// 契约更宽松
fn frobnicate3<T: AsRef<str>>(s: T) -> T {
    s
}

fn main() {
    let string = String::from("example");
    let borrowed: &str = "hello";
    let cow: Cow<str> = Cow::Borrowed("world");

    let result1: &str = frobnicate3::<&str>(string.as_ref());
    let result2: &str = frobnicate3::<&str>(borrowed);
    let result3: Cow<str> = frobnicate3(cow);

    println!("Result1: {:?}", result1);
    println!("Result2: {:?}", result2);
    println!("Result3: {:?}", result3);
}
```

> 上边三个例子，都传入字符串，返回字符串，但契约不同，没有哪个更好，只是要仔细规划契约，否则改变契约就会引起破坏性的变化
{: .prompt-info }



> 小结：最开始设计接口时候，尽量往宽松取设计
{: .prompt-info }


## 3.3 使用泛型参数（Generic Arguments）来让接口更灵活
* 通过泛型来放宽对函数的要求
* 大多数情况下是值得使用泛型代替具体类型的

```rust
// 有一个函数，它接收 AsRef<str> traiit 的参数
fn print_as_str<T>(s: T) where T: AsRef<str> {
    println!("{}", s.as_ref());
}

/*
    这个函数是泛型的额，它对 T 进行了泛型化
    这意味着它会对你使用它的每一种实现了 AsRef<str> 的类型进行单态化
    例如 如果你用一个 String 和一个 &str 来调用它
    你就会在你的二进制文件中有两份函数的拷贝
*/
fn main() {
    let s = String::from("hello");
    let r = "world";

    print_as_str(s); // 调用 print_as_str::<String>
    print_as_str(r); // 调用 print_as_str::<&str>
}
```

* 上边代码的问题在于：你就会在你的二进制文件中有两份函数的拷贝，怎么解决呢？
1. 可以改为接收 `&dyn AsRef<str>` 

```rust
/*
    解决方法：改为接收 &dyn AsRef<str>
    这个函数就不再是泛型的了，它接受一个 trait 对象
    要求 s 是任何实现了 AsRef<str> 的类型
    这意味着它会在运行时使用动态分发来调用 as_ref 方法，
    并且你只会在你的二进制文件中有一份函数的拷贝
*/
fn print_as_str(s: &dyn AsRef<str>) {
    println!("{}", s.as_ref());
}

fn main() {
    let s = String::from("hello");
    let r = "world";


    print_as_str(&s); // 传递一个类型为 &dyn AsRef<str> 的 trait 对象
    print_as_str(&r); // 传递一个类型为 &dyn AsRef<str> 的 trait 对象
}
```

* 但是不要走极端，即不要把每个参数都变为泛型的或 `trait 对象`

> 经验法则：如果接口的用户合理、频繁的使用`其他类型代替你最初选定的类型`，那么参数定义为`泛型`更合适。
{: .prompt-info }


### Issue
* 通过单态化会为每个使用泛型代码的类型组合生成泛型代码的副本
1. 这样就会有担心，即如果有很多参数变成泛型，最终二进制文件会过大，如何解决？

* 那么解决这个问题方案，即使用`动态分发`，以忽略不计的性能成本来缓解这个问题
1. 它是有成本的，但是非常小
2. 对于以引用方式获取的参数（`dyn trait 不是 Sized 的`，需要使用宽指针来使用它们），可以使用动态分发来代替泛型参数，下边看例子

```rust
/*
    静态分发：
    假设我们有一个名为 process 的泛型函数，它接受一个类型参数 T 并对其执行了某些操作
    这种函数就使用静态分发，这意味着在编译时将为每个具体类型 T 生成相应的实现
*/
fn process<T>(value: T) {
    // 处理 value 代码
    println!("处理 T");
}

/*
    动态分发：运行在运行时选择实现
    它们可以通过传递 Trait 对象作为参数，
    使用 dyn 关键字来实现，以下是一个例子
*/
trait Processable {
    fn process(&self);
}

#[derive(Debug)]
struct TypeA;
impl Processable for TypeA {
    fn process(&self) {
        println!("处理 TypeA");
    }
}

struct TypeB;
impl Processable for TypeB {
    fn process(&self) {
        println!("处理 TypeB");
    }
}

/*
    接收的类型为 trait 对象
    要求对象实现了 Processable
    如果调用者想要使用动态分发并在运行时选择实现
    它们可以调用 process_trait_object 函数，并传递 Trait 对象作为参数
    调用者可以根据需求选择要提供的具体实现 (面向协议编程)
*/
fn process_trait_object(value: &dyn Processable) {
    value.process();
}


fn main() {
    let a = TypeA;
    let b = TypeB;

    // 以动态分发 方式调用
    process_trait_object(&a);
    process_trait_object(&b);

    // 以静态分发 方式调用，传入不同类型产生不同的类型的实现
    process(&a);
    process(&b);
    process(&a as &dyn Processable); // 使用泛型时，调用者始终可以通过传递一个 trait 对象来选择动态分发
    process(&b as &dyn Processable);

    println!("TypeA {:?}", a);
}
```

### 动态分发（dynamic dispatch）
* 如果你的代码不会对性能敏感，那就可以使用动态分发

> 在高性能应用中：在频繁调用的热循环中使用动态派发可能会成为一个致命的问题
{: .prompt-info }


* 在撰写本教程时，只有简单的 trait 约束时，才能使用动态分发
1. 例如：`T: AsRef<str>` 或 `impl AsRef<str>`

* 对于更复杂的约束，Rust 无法构造动态分发的虚函数表（vtable）
1. 因此无法使用类似 `&dyn Hash + Eq` 这种组合的约束


> 使用泛型时(静态分发)，调用者始终可以通过传递一个 `trait` 对象来选择动态分发。而反过来不成立：即静态分发可以接受`泛型参数`或 `trait 对象`，但是接受一个 `trait 对象`作为参数时，调用者必须提供其 `trait 对象`，而无法使用静态分发，例如上边 `process(&b as &dyn Processable);`
{: .prompt-info }


### 那么接口如何考虑泛型参数呢？

* 建议：应该从`具体类型`开始编写接口，然后逐渐将它们转为泛型。
1. 这样是可行的，但不一定向下兼容，下面看一个例子

```rust
fn foo(v: &Vec<usize>) {
    // Code...
}

// 改为使用 Trait 限定 AsRef<[usize]> 即 impl AsRef<[usize]>
fn foo2(v: impl AsRef<[usize]>) {
    // Code...

}

fn main() {
    
    let iter = vec![1, 2, 3].into_iter();

    /*
        这时调用 foo 没有问题，因为编译器可以推断出 iter.collect() 应该收集为一个 Vec<usize> 类型，
        因为我们将其传递给了接受 &Vec<usize> 的 foo 函数
     */
    // foo(&iter.collect());

    /*
        然而改为 foo2 后, 编译器只知道 foo2 函数接受一个实现了 AsRef<[usize]> 特质的类型，
        然而有多个类型满足这个条件，例如 Vec<usize> 和 &[usize]
        因此编译器无法确定应该将 iter.collect() 的结果解释为那个具体类型
        这就导致编译器无法推断类型，并且调用者代码将编译失败

        而为了解决这个问题：
        就需要指定一个确定的类型, foo2(&iter.collect::<Vec<usize>>())
        或 
        let list: Vec<usize> = iter.collect();
        foo2(&list)
     */
    // let list: Vec<usize> = iter.collect();
    // foo2(&list);
    foo2(&iter.collect::<Vec<usize>>());

}
```

## 3.4 对象安全（Object Safety）
* 在定义 trait 时，它是否是对象安全的，这也是契约未写明的一部分

### 什么是对象安全？




<!--![image](/assets/images/rust/web_server/teacher_aim.png)-->



