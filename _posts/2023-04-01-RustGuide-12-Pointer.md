---
layout: post
title: RustGuide-12-Pointer
date: 2023-04-01 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 15 智能指针
* Rust 中最常见的指针就是 `引用`
* 引用
1. 使用 & 符号
2. 会借用它指向的值
3. 没有其余的开销
4. 最常见的指针类型

## 智能指针
* 智能指针是这样的一些数据结构
1. 行为与指针相似
2. 有额外的元数据和功能

* 常见的智能指针：`Box<T>` 和 `Rc<T>` 和 `Ref<T>` 和 `RefMut<T>` 和 `RefCell<T>`


> 区别：引用仅借用数据，而智能指针通常拥有数据
{: .prompt-info }

## 引用计数（reference counting）智能指针类型
* 通过记录所有者的数量，使一份数据被多个所有者同时持有
* 并且在没有任何所有者时自动清理数据
* 像 OC 的 ARC

> 可以简单理解为 OC 的指针
{: .prompt-info }


* 引用和智能指针的区别
1. `引用：只借用数据`
2. `智能指针：很多时候都拥有它所指向的数据`

### 智能指针的例子
* `String` 和 `Vec<T>` 就是智能指针
1. 都拥有一片内存区域，并且允许用户对其操作
2. 还拥有元数据（例如容量等）
3. 还提供额外的功能和保障（String 保障其数据是合法的 UTF-8 编码）

### 智能指针的实现
* 通常使用 `struct` 实现，并且实现了 `Deref` 和 `Drop` 这两个 `trait`
* `Deref trait`：允许智能指针 `struct` 的实例像引用一样使用，这样就能同时用于引用和智能指针
* `Drop trait`：允许你自定义当智能指针实例走出作用域时的代码


### 标准库中常见的智能指针
* `Box<T>` 在 `heap` 内存上分配值
* `Rc<T>` 启用多重所有权的引用计数类型
* `Ref<T>` 和 `RefMut<T>`，通过 `RefCell<T>` 访问的：在运行时而不是编译时强制借用规则的类型
* 内部可变模式（interior mutability pattern）：这个模式不可变类型可暴露出可修改其内部值的 API（像 swift 的 mutating）


## 使用 `Box<T>` 来指向 Heap 上的数据
* `Box<T>` 是一种最简单的智能指针
* 允许你再 `heap` 上存储数据（而不是在 `stack` 上）
* `stack` 上是指向 `heap` 数据的指针
* 没有性能开销，没有其他额外功能

> 因为它实现了 `Deref` 和 `Drop` 这两个 `trait`，所以它是智能指针
{: .prompt-info }


![image](/assets/images/rust/box.png)


### `Box<T>` 常用的场景
1. 在编译时，某类型的大小无法确定，但使用该类型时，上下文却需要知道它确切的大小
2. 或当你有大量数据，想移交所有权，但需要确保在操作时数据不会被复制
3. 在使用某个值时，你只关心它是否实现了特定的 `trait`（trait 类型），而不关心它的具体类型（即面向协议编程时）


### 使用 `Box<T>` 在 heap 上存储数据

```rust
fn main() {
    println!("Hello, world!");
    /*
        5 存储在 heap 上，类似 OC 的 NSNumber 了
        b 走完 main 函数作用域会被释放
        会释放 stack 上的指针和存在 heap 上的数据
     */
    let b = Box::new(5);
    println!("b = {}", b);
}
```

### 使用 `Box<T>` 赋能递归类型

> 在编译时 Rust 需要知道每一个类型所占的空间大小。而递归类型无法再编译时确定大小。但 `Box` 类型的大小确定，所以可以在递归类型中使用 `Box` 解决。
{: .prompt-info }

* 函数式语言中叫 `Cons List`


![image](/assets/images/rust/digui.png)

### 关于 `Cons List`
* `Cons List` 是来自 `Lisp` 语言的一种数据结构
* `Cons List` 里每个成员由两个成员组成，与上图相似
1. 当前项的值
2. 下一个元素
* `Cons List` 里最后一个成员只包含一个 `Nil` 值（一个终止标记），没有下一个元素
* `Cons List` 并不是 Rust 里的常用集合
* 通常情况下， `Vec<T>` 是更好的选择


### 编译时确定非递归类型存储空间的大小
* 这里例子为枚举，即编译时找最大存储空间的变体为其需要的存储空间大小
* 即取最大的那个成员或变体的存储空间大小


```rust
enum Message {
    Quit,  // 不需要空间
    Move {x: i32, y: i32}, // 需要两个 i32 空间
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

### 使用 Box 来获的确定大小的递归类型
* 因为 `Box<T>` 是一个指针，所以 Rust 总是知道它需要多少空间
1. 因为指针的大小不会基于它指向的数据大小变化而变化

```rust
use crate::List:: {Cons, Nil};
fn main() {
    let list = Cons(1, 
        Box::new(Cons(2,
             Box::new(Cons(3,
                 Box::new(Nil))))));
    println!("{:#?}", list);
}

#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

* `Box<T>` 
1. 只提供了 `间接` 存储和 `heap` 内存分配的功能
2. 没有其他额外功能
3. 没有性能开销
4. 适用与需要 `间接` 存储的场景，例如 `Cons List`
5. 实现了 `Deref trait` 就允许将 `Box` 的值当成`引用`来处理
6. 实现了 `Drop trait` 就是当 `Box` 离开作用域时，它指向的 `heap` 上数据以及 `stack` 上的指针数据都会被清理
7. Rust 会自动释放 Box 的堆（heap）内存


> Box 内存释放原则（几乎正确）：如果一个变量绑定到一个 `Box`, 当 Rust 释放变量的 `frame` 时，Rust 也会释放 `Box` 的堆（heap）内存
{: .prompt-info }


## Deref Trait
* `Deref` 就是解引用的意思
* 实现了 `Deref Trait` 使我们可以自定义解引用运算符 `*` 的行为


> 通过实现 `Deref Trait`，智能指针可以像常规引用一样来处理，`即处理引用的代码可以不加修改的处理这个智能指针`
{: .prompt-info }



### 解引用运算符
* 常规引用也是一种指针

```rust
    let x = 5;
    let y = &x;
    assert_eq!(5, x);
    assert_eq!(5, *y); // * 即解引用，取出 y 的值与 5 进行比较
    assert_eq!(5, y);  // 这样会报错，因为没有实现 integer 与 &integer 的比较
```

### 把 `Box<T>` 当作引用来使用
* `Box<T>` 可以代替上例中的引用

```rust
    let x = 5;
    //let y = &x;
    let y = Box::new(x);  // 即 `Box<T>` 可以当作引用来使用
    assert_eq!(5, x);
    assert_eq!(5, *y); 
```

### 定义自己的智能指针
* `Box<T>` 被定义成拥有一个元素的 `Tuple Struct` 

* 实现 `Deref Trait`
1. 标准库中的 `Deref Trait` 要求我们实现一个 `deref` 方法
2. `deref` 方法会借用 `self`，并返回一个指向内部数据的引用

```rust
use std::ops::Deref;

// 一个 tuple struct，即一个有名称的元组
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    // 定义了 Deref trait 的关联类型
    type Target = T;
    
    // 自定义解引用 * 的行为
    fn deref(&self) -> &T {
        &self.0
    }

}

fn main() {
    let x = 5;
    //let y = &x;
    let y = MyBox::new(x);
    assert_eq!(5, x);
    /*
        如果 MyBox 没实现 Deref trait，这里会报错
        *y 相当于 *(y.deref())，Rust 会隐式展开为 *(y.deref())
     */
    assert_eq!(5, *y);
}
```

### 函数和方法隐式解引用转化`（Deref Coercion）`
* 隐式解引用转化`（Deref Coercion）`是为函数和方法提供的一种便捷特性
* 原理：假设 `T` 实现了 `Deref trait`，那么 `Deref Coercion` 就可以把 `T` 的引用转化为 `T` 经过 `Deref` 操作后生成的引用
* 当把某个类型的引用传递给函数或方法时，但它的类型与定义的参数类型不匹配
1. 这时 `Deref Coercion` 就会自动发生
2. 编译器会对 `Deref` 进行一些列调用，来把它转为所需的参数类型，这个操作在编译时就完成了，没有额外的性能开销

```rust
use std::ops::Deref;

fn hello(name: &str) {
    println!("Hello, {}", name);
}

fn main() {
    // 
    let m = MyBox::new(String::from("Rust"));

    /*
        &m 是 &MyBox<String> 类型
        由于 MyBox 实现了 deref trait，
        所以可以自动把 &MyBox 转化为 &String 
        String 也实现了 deref trait 会返回 &str
        最终 &m 的类型与 hello 函数的类型就匹配了
     */
    hello(&m);

    // 如果没有实现 deref trait 如何调用呢
    // 这样不便于阅读
    hello(&(*m)[..]);

    hello("Rust");
}
```


### 解引用`（Deref Coercion）` 与可变性
* `Deref` 即对不可变引用的  `*`（解引用）运算符的重载
* `DerefMut` 即对可变引用的 `*` （解引用）运算符的重载
* 在类型和 `trati` 满足下列三种情况时，Rust 会执行 `deref coercion`
1. `T:Deref<Target=U>`（即 T 现实了 Deref trait，而 Deref trait 方法返回类型是 U），允许 &T 转化为 &U
2. `T:DerefMut<Target=U>`（即 T 现实了 DerefMut trait，而 DerefMut trait 方法返回类型是 U），允许 &mut T 转化为 &mut U
3. `T:Deref<Target=U>`，允许 &mut T 转化为 &U，即可变引用可转为不可变引用，反之不行


## Drop Trait
* 类型实现了 `Drop trait` 之后，可以让我们自定义当它的值将要离开作用域时发生的动作
1. 例如：文件、网络资源释放等
2. 任何类型都可以实现 `Drop Trait`
* `Drop Trait` 只要求你实现 `drop` 方法
1. 方法的参数就是对 `self` 的可变引用
* `Drop Trait` 在与导入模块里`（prelude）`，所以使用时不需要手动进行引入

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    /*
        drop 方法通常用于释放资源
        这里出于演示所以打印了一句话
     */
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}` !", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("my stuff") };
    let c = CustomSmartPointer { data: String::from("other stuff") };
    println!("CustomSmartPointer created.");
}
```

### 使用 `std::mem::drop` 来提前 `drop` 值
* 很难直接禁用自动的 `drop` 功能，也没有必要
1. `Drop Trait` 的目的就是进行自动的释放处理逻辑
* Rust 不允许手动调用 `Drop Trait` 里的 drop 方法
1. 类似 `swift` 的 `deinit` 也是不能手动调用的
* 但可以手动调用标准库的 `std::mem::drop` 函数，来提前 `drop` 值

```rust
    let c = CustomSmartPointer { data: String::from("my stuff") };
    c.drop();   // Error! 不能显式的调用
    
    drop(c);    // 提前把 c drop 掉，drop 函数在 prelude 里 
    let d = CustomSmartPointer { data: String::from("other stuff") };
    println!("CustomSmartPointer created.");
```

> `std::mem::drop` 函数的调用不会干扰 Rust 的自动清理机制，因为它通过接管值的所有权并在调用后销毁它，避免了双重释放问题
{: .prompt-info }

## `Rc<T>` 引用计数智能指针
* 有时一个值会有多个所有者
* 为了支持多重所有权，Rust 提供了 `Rc<T>` 类型
1. reference counting 引用计数
2. 在值的内部维护一个引用次数的计数器，追踪所有到这个值的引用
3. 如果引用计数是 0，说明该值可以被清理掉了，而且不会触发引用失效的问题

### `Rc<T>` 使用场景
* 当你希望 heap 上的数据，分享给程序多个部分使用时，但无法在编译时确定具体哪个部分最后使用，这时就可以使用 `Rc<T>`
* 如果我们能确定那部分程序是最后才释放的，那就让那部分成为其所有者就行了，这样依靠编译时的所有权规则就可以保证程序的安全性
* `Rc<T>` 也实现了 Drop trait

> `Rc<T>` 只能用于单线程的场景，它不在预导入模块中
{: .prompt-info }


#### `Rc<T>` 使用例子

![image](/assets/images/rust/rc_t.png)

* 下边代码为实现上图结构

```rust
enum List {
    //Cons(i32, Box<List>),
    Cons(i32, Rc<List>),
    Nil,
}
use crate::List:: { Cons, Nil };
use std::rc::Rc;

fn main() {
    // let a = Cons(5, 
    //     Box::new(Cons(10, Box::new(Nil))));
    // 这里获得了 a 的所有权
    // let b = Cons(3, Box::new(a));
    // 这里报错 因为 a 发生了 move，如果修改？把 Box 改成 Rc<T> 类型
    // let c = Cons(4, Box::new(a)); 


    let a = Rc::new(Cons(5, 
        Rc::new(Cons(10, Rc::new(Nil)))));  // a 的引用计数 1
    println!("Count after creat a = {}", Rc::strong_count(&a));

    // 这里不会获得 a 的所有权，而是 a 的引用计数 +1
    let b = Cons(3, Rc::clone(&a)); // a 的引用计数 2
    println!("Count after creat b = {}", Rc::strong_count(&a));

    {
        let c = Cons(4, Rc::clone(&a)); // a 的引用计数 3
        println!("Count after creat c = {}", Rc::strong_count(&a));
        /*
            Rc::clone 不会深 copy，执行速度快
            而 a.clone() 会进行深 copy，需要花费大量时间来完成数据的 copy
            
            c 走出作用域，会自动 drop，a 的引用计数减 1 
        */
    }
    println!("Count after c goes out of scope = {}", Rc::strong_count(&a));

}
```


* 克隆函数 `Rc::clone(&a)` 会增加引用计数
* `Rc::strong_count(&a)` 函数会获得强引用的计数
* `Rc::weak_count(&a)` 函数会获得弱引用的计数

### `Rc::clone()` VS `clone()` 方法
* `Rc::clone` 增加引用计数，不会执行数据的深 `copy`，执行速度快
* 而类型的 `clone()` 很多都会进行深 `copy`，需要花费大量时间来完成数据的 `copy`


> `Rc<T>` 实际上是通过`不可变引用`，使你可以在程序不同部分之间共享只读数据。如果 `Rc<T>` 允许你持有可变引用的话就会违反借用规则，即多个指向同一区域的可变引用会导致数据的竞争和数据的不一致。
{: .prompt-info }


> 小结：`Rc<T> 引用是不可变引用`
{: .prompt-info }


## RefCell 和内部可变性

### 内部可变性（interior mutability）
* 内部可变性是 Rust 的设计模式之一
* 它允许你在只持有不可变引用的前提下对数据进行修改
1. 通常而言，类似的规则会被借用规则禁止
2. 但为了改变数据，内部可变性模式对数据结构中使用了 unsafe 代码来绕过 Rust 正常的可变性和借用规则


### `RefCell<T>`
* 与 `Rc<T>` 不同，`RefCell<T>` 类型代表了其持有数据的唯一所有权


* 回忆下借用规则
1. 在任何给定时间里，要么只能拥有一个可变引用，要么只能拥有任意数量的不可变引用
2. 引用总是有效的

### `RefCell<T>` VS `Box<T>`
* `Box<T>`
1. `编译阶段`就强制代码遵守这个借用规则
2. 没有满足则出现错误


* `RefCell<T>`
1. 只会在`运行时`检查借用规则
2. 没有满足就会触发 panic

### 借用规则在不同阶段进行检查的比较
* 编译阶段
1. 尽早暴露问题
2. 没有任何运行时的开销
3. 对大多数场景是最佳的选择
4. Rust 的默认行为

* 运行时
1. 问题暴露延后，甚至到生成环境
2. 因借用计数产生些许性能损失
3. 实现某些特定的内存安全场景（比如不可变环境中修改自身数据）

> 与 `Rc<T>` 相似，`RefCell<T>` 只能用于单线程的场景
{: .prompt-info }


### 选择 `Box<T>`、`Rc<T>`、`RefCell<T>` 的依据
* 同一个数据的所有者
1. `Box<T>`：一个
2. `Rc<T>`：多个
3. `RefCell<T>`：一个

* 可变性、借用检查
1. `Box<T>`：可变、不可变借用（编译时检查）
2. `Rc<T>`：不可变借用（编译时检查）
3. `RefCell<T>`：可变、不可变借用（运行时检查）

> 即使 `RefCell<T>` 本身不可变，但仍然能修改其中存储的值
{: .prompt-info }


### 内部可变性：可变的借用一个不可变的值
* 其实本来借用规则导致你无法可变的借用一个不可变的值


```rust
fn main() {
    let x = 42;
    let y = &mut x; // Error! 你无法可变的借用一个不可变的值
}
```

* 在某些特定情况下某个值对外部是不可变的，但可以在函数内部可以修改其值，除本身这个函数其余函数都不能修改其值，`RefCell<T>` 就是获得内部可变性的方法
* 但 `RefCell<T>` 并没有完全绕开借用规则，只是借用检查延后到运行时而已
* 如果违反了借用规则会 panic


### 使用 `RefCell<T>` 在运行时记录借用信息
* 两个方法（安全接口）
1. `borrow` 方法，返回智能指针 `Ref<T>`，它实现了 `Deref`
2. `borrow_mut` 方法，返回内部值的可变引用，返回 `RefMut<T>`，它实现了 `Deref`
* `RefCell<T>` 会记录当前存在多少个活跃的 `Ref<T>` 和 `RefMut<T>` 智能指针
1. 每次调用 `borrow` 方法，不可变借用计数加 1。任何一个 `Ref<T>` 的值离开作用域被释放时，不可变借用计数减 1
2. 每次调用 `borrow_mut` 方法，可变借用计数加 1。任何一个 `RefMut<T>` 的值离开作用域被释放时，可变借用计数减 1

> Rust 就是通过这个技术来维护借用检查规则的。在任何一个给定时间里，只允许拥有多个不可变借用或一个可变借用，如果违反了，则 `RefCell<T>` 在运行时就会 `panic`
{: .prompt-info }

```rust
pub trait Messenger {
    fn send(&self, msg: &str);
}

// 一个泛型类型的 struct
pub struct LimitTracker<'a, T: 'a + Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T> where T: Messenger {
    pub fn new(messenger: &T, max: usize) -> LimitTracker<T> {
        LimitTracker { messenger, value: 0, max, }
    }

    // 这其实是测试函数
    pub fn set_value(&mut self, value: usize) {
        self.value = value;

        let percentage_of_max = self.value as f64 / self.max as f64;
        if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        } else if percentage_of_max >= 0.9 {
            self.messenger.send("Urgent warning: You've used up over 90% of your quota!");

        } else if percentage_of_max >= 0.75 {
            self.messenger.send("Warning: You've used up over 75% of your quota!");
        }
    }

}
 
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;
    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger { sent_messages: RefCell::new(vec![]), }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, msg: &str) {
            /*
                这里使用 borrow_mut 获得内部值的可变引用，进行数据添加
             */
            self.sent_messages.borrow_mut().push(String::from(msg));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);
        limit_tracker.set_value(80);
        // borrow() 获取内部值的不可变引用
        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }

}
```



### `Rc<T>` 与 `RefCell<T>` 结合使用来实现一个拥有多重所有权的可变数据

* 通过在 `Rc<T>` 中存储 `RefCell<T>` 达到修改 `Rc<T>` 中的值的作用

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use crate::List:: { Cons, Nil };

fn main() {
    let value = Rc::new(RefCell::new(5));
    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));
    let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));

    // *value is Rc<RefCell<i32>>
    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("a after = {:?}", b);
    println!("a after = {:?}", c);
}


// 运行结果
//a after = Cons(RefCell { value: 15 }, Nil)
//a after = Cons(RefCell { value: 6 }, Cons(RefCell { value: 15 }, Nil))
//a after = Cons(RefCell { value: 10 }, Cons(RefCell { value: 15 }, Nil))
```

> `将 Rc<T> 与 RefCell<T> 结合使用是常见的方法`
{: .prompt-info }



### 其他课实现内部可变性的类型
* `Cell<T>` 通过复制数据来访问数据
* 而 `RefCell<T>` 是通过借用来访问数据
* `Mutex<T>` 用于实现跨线程情形下的内部可变性模式


## 循环引用导致内存泄露
* 例如使用 `Rc<T>` 和 `RefCell<T>` 就可能制造出循环引用，从而发生内存泄露
1. 即每个项的引用计数不会变成 0，值也不会被处理掉

```rust
use std::{cell::RefCell, rc::Rc};

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

use crate::List:: { Cons, Nil };

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match self {
            Cons(_, item) => Some(item),
            Nil => None,
        }
    }
}


fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));

    println!("a initial rc count = {}", Rc::strong_count(&a));
    println!("a next item = {:?}", a.tail());

    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));

    println!("a rc count after b create = {}", Rc::strong_count(&a));
    println!("b initial rc count = {}", Rc::strong_count(&b));
    println!("b next item = {:?}", b.tail());

    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    println!("a rc count after changing a = {}", Rc::strong_count(&a));
    println!("b rc count after changing a = {}", Rc::strong_count(&b));

    // 如果 println! 则死循环，stack overflow
    //println!("a next item = {:?}", a.tail());

}
```

![image](/assets/images/rust/retain_cycle.png)


### 防止内存泄露的解放方法
* 依靠开发者来保证，不能依靠 Rust
* 重新组织数据结构：把引用拆为持有所有权，和不持有所有权两种情况
1. 循环引用中的一部分具有所有权关系，另一部分不涉及所有权关系
2. 而只有持有所有权关系才会影响某个值释放被释放清理

* 防止循环引用的解决方案：可以把 `Rc<T>` 换成 `Weak<T>`
* `Rc::clone` 为 `Rc<T>` 实例的 `strong_count` 加 1，`Rc<T>` 的实例只有在 `strong_count 为 0` 的时候才会被清理
* `Rc<T>` 实例可以通过调用 `Rc::downgrade` 方法来创建值的 `Weak Reference`（弱引用），返回类型是 `Weak<T>`（也是智能指针）
* 每次调用 `Rc::downgrade` 会为 weak_count 加 1
* `Rc<T>` 使用 `weak_count` 来追踪存在多少 `Weak<T>`


> `weak_count 不为 0` 并不影响 `Rc<T>` 实例的清理
{: .prompt-info }


### Strong Vs Weak
* `Strong Reference`（强引用）是关于如何分享 `Rc<T>` 实例的所有权
* `Weak Reference`（弱引用）并不表达上述意思
* 使用 `Weak Reference` 并不会创建循环引用
1. 当 `Strong Reference` 的数量为 0 时，`Weak Reference` 会自动断开
* 在使用 `Weak Reference` 之前，需要保证它指向的值仍然存在
1. 可在 `Weak<T>` 实例上调用 `upgraade` 方法，返回 `Option<Rc<T>>`，进行验证

```rust
use std::{cell::{RefCell, Ref}, rc::{Rc, Weak}};

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });
    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
    println!("leaf strong = {}, weak = {}", Rc::strong_count(&leaf), Rc::weak_count(&leaf));

    {
        let branch = Rc::new(Node {
            value: 5,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![Rc::clone(&leaf)]),
        });
        *leaf.parent.borrow_mut() = Rc::downgrade(&branch);
    
        println!("2 leaf parent = {:?}", leaf.parent.borrow().upgrade());
        println!("2 leaf strong = {}, weak = {}", Rc::strong_count(&leaf), Rc::weak_count(&leaf));
        println!("2 branch strong = {}, weak = {}", Rc::strong_count(&branch), Rc::weak_count(&branch));
    }

    println!("3 leaf parent = {:#?}", leaf.parent.borrow().upgrade());
    println!("3 leaf strong = {}, weak = {}", Rc::strong_count(&leaf), Rc::weak_count(&leaf));
}


// 运行结果
// leaf parent = None
// leaf strong = 1, weak = 0
// 2 leaf parent = Some(Node { value: 5, parent: RefCell { value: (Weak) }, children: RefCell { value: [Node { value: 3, parent: RefCell { value: (Weak) }, children: RefCell { value: [] } }] } })
// 2 leaf strong = 2, weak = 0
// 2 branch strong = 1, weak = 1
// 3 leaf parent = None
// 3 leaf strong = 1, weak = 0
```
