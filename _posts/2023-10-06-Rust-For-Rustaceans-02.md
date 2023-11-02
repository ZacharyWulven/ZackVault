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
* 对象安全：描述一个 `trait` 是否可以安全的包装成为一个 `trait 对象`

### 对象安全的 `trait` 是满足以下条件的 `trait`(RFC 255)
* 所有的 `supertrait`(父级 traiit) 必须是安全的
* `Sized trait` 不能作为 `supertrait`（不能要求 `Self:Sized`）
* 不能有任何关联的常量
* 不能有任何带有泛型的关联类型
* 所有的关联函数必须满足以下条件之一：
  * 1 可以从 `trait 对象`分发的函数（Dispatchable functions）：
    * 要求没有任何类型参数（但生命周期参数是允许的）
    * 并且必须是一个方法，只在接收器类型（接收类型）中使用 `Self`
    * 接收器是以下类型之一：
      * `&Self 即 &self`
      * `&mut Self 即 &mut self`
      * `Box<Self>`
      * `Rc<Self>`
      * `Arc<Self>`
      * `Pin<P>`，其中 `P` 是以上类型之一
    * 没有 `where Self: Sized` 约束（Self 的接收器类型，即 self 暗含了这一点）
  * 2 显示不可分发的函数（non-dispatchable functions）要求：
    * 具有 `where Self: Sized` 约束（Self 的接收器类型，即 self 暗含了这一点）



* 如果这个 `trait` 是对象安全的，那么：
  - 就可以使用 `dyn trait` 将实现该 `trait` 的不同类型视为单一通用类型

* 如果 `trait` `不是`对象安全的，那么：
  - 编译器会禁止使用 `dyn trait`


### 在设计接口时，建议 `trait` 是对象安全的（即使是稍微降低使用的便利程度），因为它提供了新的使用方式和灵活性，

* 看个对象安全的例子：

```rust
/*
    假设我们有一个 Animal 特征，它有两个方法：name 和 speak
    name 方法返回一个 &str，表示动物名称
    speak 方法打印动物发出的声音
    我们可以为 Dog 和 Cat 类型实现这些特征

*/


trait Animal {
    fn name(&self) -> &str;
    fn speak(&self);
}

struct Dog {
    name: String
}

impl Animal for Dog {
    fn name(&self) -> &str {
        &self.name
    }

    fn speak(&self) {
        println!("woof!");
    }
}

struct Cat {
    name: String
}

impl Animal for Cat {
    fn name(&self) -> &str {
        &self.name
    }

    fn speak(&self) {
        println!("Meow!");
    }
}

/*
    这个 Animal 特征是 object-safe 的，因为它没有返回 Self 类型或使用泛型参数
    所以我们可以用它来创建一个 trait 对象

    这样我们就可以用一个统一的类型 Vec<&dyn Animal> 来存储不同类型的动物
    并且通过 trait 对象来调用它们的方法
*/

fn main() {
    let dog = Dog { name: "Fido".to_string() };

    let cat = Dog { name: "Whiskers".to_string() };

    // 

    let animals: Vec<&dyn Animal> = vec![&dog, &cat];

    
    for animal in animals {
        println!("This is {}", animal.name());
        animal.speak();
    }
}
```


* 非对象安全的例子：

```rust
/*
    如果 Animal 有返回 Self 类型，那它就不是对象安全的了
    那这个 Animal trait 就不是对象安全的，因为 clone 方法违反了规则（返回类型不能是 Self）
    现在就不能用它来创建 trait object 了，因为编译器无法知道 Self 具体指代哪个类型
*/
trait Animal {
    fn name(&self) -> &str;
    fn speak(&self);
    fn clone(&self) -> Self; 
}

struct Dog {
    name: String,
}

impl Animal for Dog {
    fn name(&self) -> &str {
        &self.name
    }

    fn speak(&self) {
        println!("wang wang!");
    }

    fn clone(&self) -> Self where Self: Sized, {
        todo!()
    }
}

struct Cat {
    name: String,
}

impl Animal for Cat {
    fn name(&self) -> &str {
        &self.name
    }

    fn speak(&self) {
        println!("miao miao!");
    }

    fn clone(&self) -> Self where Self: Sized, {
        todo!()
    }
}

fn main() {
    let dog = Dog { name: "Fido".to_string() };

    let cat = Dog { name: "Whiskers".to_string() };

    // 下边代码就报错了，因为 Animal 不是对象安全的
    let animals: Vec<&dyn Animal> = vec![&dog, &cat]; // error

    for animal in animals {    // error
        println!("This is {}", animal.name());
    }
}
```

* 那么我们还想保留上边 `Animal 中 clone 方法`，并保持对象安全应该怎么做呢 ？

> 解决方案：`给 Self 添加 Sized 约束`，即 `where Self: Sized`，这时 `clone` 这个方法就只能在具体类型上调用，而不是在 `trait 对象（通用类型）`上调用
{: .prompt-info }


```rust

trait Animal {
    fn name(&self) -> &str;
    fn speak(&self);
    fn clone(&self) -> Self where Self: Sized; // 添加 Sized 约束
}

struct Dog {
    name: String,
}

impl Animal for Dog {
    fn name(&self) -> &str {
        &self.name
    }

    fn speak(&self) {
        println!("wang wang!");
    }

    fn clone(&self) -> Self where Self: Sized, {
        todo!()
    }
}


#[derive(Debug)]
struct Cat {
    name: String,
}

impl Animal for Cat {
    fn name(&self) -> &str {
        &self.name
    }

    fn speak(&self) {
        println!("miao miao!");
    }

    fn clone(&self) -> Self where Self: Sized, {
        Cat {
            name: self.name.clone(),
        }
    }
}

fn main() {
    let dog = Dog { name: "Fido".to_string() };

    let cat = Cat { name: "Whiskers".to_string() };

    /*
        这样我们就可以继续用 Animal 来创建 trait object 了
     */
    let animals: Vec<&dyn Animal> = vec![&dog, &cat];

    for animal in animals {
        println!("This is {}", animal.name());
        animal.speak();
        /*
            Note：但是我们不能用 trait object 来调用 clone 方法
            因为只能在具体的类型上调用 clone 方法
         */
        //animal.clone();  // error
    }

    // 因为只能在具体的类型上调用 clone 方法
    let cat2: Cat = cat.clone();
    println!("Cloned cat is {:?}", cat2);
}
```

### 如果 `trait` 必须有泛型方法时：
* 1 那么建议把泛型参数放在 `trait` 上，来看下例子

```rust
use std::collections::HashSet;
use std::hash::Hash;

// 将泛型参数放在 Trait 上
trait Container<T> {
    fn contains(&self, item: &T) -> bool;
}

// 我们可以为不同容器类型实现 Container Trait，每个实现都具有自己特定的元素类型
// 例，可以为 Vec<T> 和 HashSet<T> 实现 Container Trait
impl<T> Container<T> for Vec<T> 
    where T: PartialEq, 
{
        fn contains(&self, item: &T) -> bool {
            self.iter().any(|x| x == item)
        }
}

impl<T> Container<T> for HashSet<T> 
where T: Hash + Eq, 
{   
    fn contains(&self, item: &T) -> bool {
        self.contains(item)
    }
}

fn main() {
    let vec_container: Box<dyn Container<i32>> = Box::new(vec![1, 2, 3]);

    let set_container: Box<dyn Container<i32>> = Box::new(
        vec![4, 5, 6].into_iter().collect::<HashSet<_>>()
    );

    // 调用 contains 方法

    println!("Vec contains 2: {}", vec_container.contains(&2));
    println!("HashSet contains 6: {}", set_container.contains(&6));
}

```

* 2 泛型参数是否可以使用动态分发，来保证 `trait` 的对象安全，`即使用动态分发来代替泛型`，看个例子


```rust
use std::fmt::Debug;

/*
    有一个 trait Foo，它有一个泛型方法 bar，接收一个泛型参数 T：
    trait Foo {
        fn bar<T>(&self, x: T);
    }
    这个 Foo 是对象安全的么？答案是：取决于 T 的类型
    1 如果 T是一个具体类型，比如 i32 或 String，那么它就 `不是` object-safe 的，
    因为它需要再运行时知道 T 的具体类型才能调用 bar 方法

    2 如果 T 是一个 trait object，比如 &dyn Debug 或 &dyn Display，
    那么这个 trait 就是 object-safe 的，因为它可以用动态分发的方式来调用 T 的方法，
    所以定义如下：
*/

trait Foo {
    fn bar(&self, x: &dyn Debug);
}

// 定义 A 让它实现 Foo
struct A {
    name: String,
}

impl Foo for A {
    fn bar(&self, x: &dyn Debug) {
        println!("A {} says {:?}", self.name, x);
    }
}

// 定义 B 让它实现 Foo
struct B {
    id: i32,
}

impl Foo for B {
    fn bar(&self, x: &dyn Debug) {
        println!("B {} says {:?}", self.id, x);
    }
}

fn  main() {
    // 这时就可以用 Foo 来创建 trait 对象了

    let a = A { name: "Bob".to_string() };

    let b = B { id: 42 };

    // 创建一个 Vec，它存储了 Foo 的 trait object
    let foos: Vec<&dyn Foo> = vec![&a, &b];

    // 遍历 Vec，并用 trait object 调用 bar 方法
    for foo in foos {
        // & 让 &str => &dyn Debug
        foo.bar(&"Hello"); // "Hello" 实现了 Debug 特征
    }
    
}
```


### 为了实现对象安全，需要做出多大的牺牲呢？
* 首先考虑你的 `trait` 会被如何使用，用户是否想把你的 `trait` 当做 `trait 对象`
  - 如果用户想使用你的 `trait 的多种不同实例`，那么你应该努力实现对象安全



## 3.5 借用 VS 拥有（Borrowed VS Owned）

### 针对 Rust 中几乎所有每个函数、Trait 和类型，都需要决定两件事：
1. 是否应该拥有数据
2. 或者仅持有对数据的引用

### 如果代码需要数据的所有权
1. 那么就必须存储拥有的数据
2. 当你的代码必须拥有数据时，还必须让调用者提供拥有的数据，而不是提供引用或克隆
3. 这样就可以让调用者控制分配，并且可清楚的看到使用相关接口的成本
 
### 如果代码不需要拥有数据
* 应该操作与引用
* But，也有例外：
  - 即像 `i32/bool/f64` 这种小类型
    * 它们直接存储和复制的成本与通过引用存储的成本基本相同
    * 并不是所有 `Copy 类型` 都适用
      - 例如：`[u8; 8192]` 是 `Copy 类型`，但在多个地方存储和复制它会很昂贵
      
      
### 有时无法确定代码是否需要拥有数据，因为它取决于运行时的情况
* 这时就需要 `Cow 类型`
  * 允许在需要时持有引用或拥有这个值
* 如果只有引用的情况下要求生成拥有的值
  * `Cow 类型` 将使用 `ToOwned trait` 在后台创建一个，通常是通过 `Clone` 方式
  

> 通常在返回类型中使用 `Cow` 来表示有时会分配内存的函数，请看下列
{: .prompt-info }


```rust
use std::borrow::Cow;
/*
    有一个函数 process_data，它接收字符串参数
    有时我们需要修改输入的字符串，并拥有对修改后的字符串的所有权
    然而，大多数情况下，我们值对输入字符串进行读取操作，而不需要修改它
*/
fn process_data(data: Cow<str>) {
    if data.contains("invalid") {
        // 如果输入字符串包含 "invalid"，我们需要修改它
        // into_owned 获取其所有权，返回持有的数据
        let owned_data: String = data.into_owned();
        // 进行一些修改操作
        println!("Processed data: {}", owned_data);
    } else {
        // 如果输入字符串不包含 "invalid"，我们只需要读取它
        println!("Data: {}", data);
    }
}

/*
    本例中，我们使用 Cow<str> 类型作为参数类型
    当调用时，我们可以传递一个普通的字符串引用（&str）
    或一个拥有所有权的字符串（String）作为参数
*/

fn main() {
    let input1 = "This is valid data.";
    process_data(Cow::Borrowed(input1));       // 传入引用

    let input2 = "This is invalid data.";
    process_data(Cow::Owned(input2.to_owned())); // 传入持有的数据
}
```


### 有时候，引用生命周期会让接口很复杂，难以使用
* 如果用户使用接口时遇到编译问题，这`表明您可能需要（即使不必要）拥有某些数据的所有权`
  * 这样做的话（拥有某些数据的所有权），建议首先考虑容易克隆或不涉及性能敏感性的数据（先拥有它们的所有权），而不是直接对大块数据的内容进行堆分配
  * 这样做可以避免性能问题并提高接口的可用性

 
## 3.6 可失败和可阻塞的析构函数（Fallible and Blocking Destructors）对接口灵活性的影响

* 析构函数（Destructor）：即在值被销毁时执行特定的清理操作
* 析构函数通常由 `Drop trait` 实现，它定义了一个 `drop` 方法，这个方法中可以进行清理操作
* 析构函数通常是不允许失败的，并且是非阻塞的执行的，但有时：
  * 例如释放资源时，可能需要关闭网络连接或写入日志文件，这些操作都有可能发生错误
  * 在 `drop` 方法中可能需要执行阻塞操作，例如等待一个线程的结束或等待一个异步任务的完成
    * 例如针对 I/O 操作的类型，在 `drop` 时是需要执行清理的
      * 例如：将写入的数据刷新到磁盘、关闭打开的文件、断开网络连接
      * 这些清理操作就应该在类型的 `Drop 实现` 中完成
        * issue：一旦值被丢弃，就无法向用户传递错误信息了，除非通过 `panic`
        * 而异步代码也有类似的 issue：你可能希望在清理过程中完成这些工作，但有其他工作处于 `pending` 状态
          * 针对异步的情况可以尝试启动另外一个执行器，但这会引入其他问题，例如在异步代码中阻塞

* 针对上述问题没有完美的解决方案：`只能是通过 Drop 进行尽力的清理`
  * 如果清理出错了，至少我们尝试了，就忽略错误并继续吧
  * 如果还有可用的执行器，可尝试生成一个 `future` 来做清理，但如果 `future` 永不会运行，我们也尽力了

* 如果用户不想留下这种 `松散的线程`，那么我们可以提供显式的析构函数
  * 手动实现一个
  * 通常是一个方法，它获取 `self` 的所有权并暴露任何错误 `use Result<_, _>` 或异步性 `use async fn`，这些都是与销毁相关的，看个例子
  
  
```rust
use std::os::fd::AsRawFd;

struct File {
    name: String,
    fd: i32,
}

impl File {
    fn open(name: &str) -> Result<File, std::io::Error> {
        // 使用 std::fs::OpenOptions 打开文件，具有读写权限
        let file = std::fs::OpenOptions::new()
            .read(true)
            .write(true)
            .open(name)?;

        // 使用 std::os::unix::io::AsRawFd 获取文件描述符
        let fd = file.as_raw_fd();
        
        // 返回一个 File 实例，包含 name 和 fd 字段
        Ok(File { 
            name: name.to_string(),
             fd, 
        })
    }

    // 一个显示的析构函数，关闭文件并返回错误
    fn close(self) -> Result<(), std::io::Error> {
        // use std::os::unix::io::FromRawFd 将 fd 转回 std::fs::File
        let file: std::fs::File = unsafe {
            std::os::unix::io::FromRawFd::from_raw_fd(
                self.fd
            )
        };
        // use std::fs::File::sync_all 将任何挂起的写入刷新到磁盘
        file.sync_all()?;
        // 使用 std::fs::File::set_len 将文件截断为 0 字节
        file.set_len(0)?;
        // 再次 use std::fs::File::sync_all 刷新截断
        file.sync_all()?;

        // 丢弃 file 实例，它会自动关闭
        drop(file);

        // 返回 Ok(())
        Ok(())
    }
}

fn main() {
    // 创建一个名为 "test.txt" 的文件，包含一些内容
    std::fs::write("test.txt", "Hello, world!").unwrap();

    // 打开文件并获取一个 File 实例
    let file = File::open("test.txt").unwrap();

    println!("File name: {}, fd: {}", file.name, file.fd);

    // 关闭文件并处理任何错误
    // 收到调用析构函数 close
    match file.close() {
        Ok(_) => println!("File closed successfully"),
        Err(e) => println!("Error closing file: {}", e),
    }

    // check 关闭后的文件大小
    let metadata = std::fs::metadata("test.txt").unwrap();
    println!("File size: {} bytes", metadata.len());
}
```


> Note：显式的析构函数需要再文档中突出显示  
{: .prompt-info }

### 添加显式的析构函数时也会遇到问题：
* 当类型实现了 `Drop trait`，在析构函数中就无法将该类型的任何字段移出
  * 因为在显式的析构函数调用后，`Drop::drop` 方法仍然会被调用，它接收 `&mut self (到 self 的可变引用)`，它要求 `self` 的所有的部分都没有被 `move`
* `Drop trait` 接收的是 `&mut self`，而不是 `self`，因此 `Drop trait` 无法实现简单的调用显式的析构函数并忽略其结果`（因为 Drop 不拥有 self）`
  * 看下代码例子
  
```rust
use std::os::fd::AsRawFd;

struct File {
    name: String,
    fd: i32,
}

impl File {
    fn open(name: &str) -> Result<File, std::io::Error> {
        // 使用 std::fs::OpenOptions 打开文件，具有读写权限
        let file = std::fs::OpenOptions::new()
            .read(true)
            .write(true)
            .open(name)?;

        // 使用 std::os::unix::io::AsRawFd 获取文件描述符
        let fd = file.as_raw_fd();
        
        // 返回一个 File 实例，包含 name 和 fd 字段
        Ok(File { 
            name: name.to_string(),
             fd, 
        })
    }

    fn close(self) -> Result<(), std::io::Error> {
        // 移出 name 字段并打印
        let name = self.name; // 不能从 `self.name` 中移出值，因为它位于 `&mut` 引用后面
        println!("Closing file {}", name);

        // use std::os::unix::io::FromRawFd 将 fd 转回 std::fs::File
        let file: std::fs::File = unsafe {
            std::os::unix::io::FromRawFd::from_raw_fd(
                self.fd
            )
        };
        // use std::fs::File::sync_all 将任何挂起的写入刷新到磁盘
        file.sync_all()?;
        // 使用 std::fs::File::set_len 将文件截断为 0 字节
        file.set_len(0)?;
        // 再次 use std::fs::File::sync_all 刷新截断
        file.sync_all()?;

        // 丢弃 file 实例，它会自动关闭
        drop(file);

        // 返回 Ok(())
        Ok(())

    }
}

// 为 File 实现 Drop trait，用于在值离开作用域时运行一些代码
impl Drop for File {
    // drop 方法，接受一个可变引用到 self 作为参数
    fn drop(&mut self) {
        /*
            在 drop 中调用 close 方法并忽略它的结果

            这里调用 close 报错，不能从 `*self` 中移出值，因为它位于 `&mut` 引用后面，
            因为这里要获取其所有权
            那这里怎么解决？没有完美的解决方案

         */
        let _ = self.close();        // error：
        // 打印一条消息，表明文件被丢弃了
        println!("Dropping file {}", self.name);
    }
}
```

#### 上边代码在 `drop 方法`中调用 `close` 报错，如何解决呢？
* 答案是：没有完美的解决方案
* 解决方案一：
  * 将顶层类型作为包装了 `Option` 的新类型，`Option` 中持有一个内部类型，之前那些相关字段都放在该类型中
  * 然后定义`两个析构函数`，在这两个析构函数中使用 `Option::take`，当内部类型还没有被取走时，调用内部类型的显示析构函数
  * 而由于内部类型没有实现 `Drop trait`，所以你可以获取所有字段的所有权
  * 缺点：即想在顶层类型上提供所有的方法，都必须包含通过 `Option` 来获取内部类型上的字段的代码，
  * 下边是代码例子
  
```rust
use std::os::fd::AsRawFd;

// 一个表示文件句柄的类型
struct File {
    inner: Option<InnerFile>
}

// 一个内部类型，持有文件名和文件描述符，即原来的 File
struct InnerFile {
    name: String,
    fd: i32,
}

impl File {
    fn open(name: &str) -> Result<File, std::io::Error> {
        // 使用 std::fs::OpenOptions 打开文件，具有读写权限
        let file = std::fs::OpenOptions::new()
            .read(true)
            .write(true)
            .open(name)?;

        // 使用 std::os::unix::io::AsRawFd 获取文件描述符
        let fd = file.as_raw_fd();
        
        // 返回一个 File 实例，包含 name 和 fd 字段
        Ok(File { 
            inner: Some(InnerFile {
                name: name.to_string(),
                fd,
            })
        })
    }
    // 一个显示的析构函数，关闭文件并返回任何错误
    fn close(mut self) -> Result<(), std::io::Error> {
        // use Option::take 取出 inner 字段的值，并检查是否是 Some(InnerFile)
        if let Some(inner) = self.inner.take() {
        // 移出 name 和 fd 字段并打印
        // 因为 inner 没有实现 drop trait，所以这里可以获取其所有权
        let name = inner.name; 
        let fd = inner.fd;
        println!("Closing file {} with fd {}", name, fd);
        // use std::os::unix::io::FromRawFd 将 fd 转回 std::fs::File
        let file: std::fs::File = unsafe {
            std::os::unix::io::FromRawFd::from_raw_fd(
                fd
            )
        };
        // use std::fs::File::sync_all 将任何挂起的写入刷新到磁盘
        file.sync_all()?;
        // 使用 std::fs::File::set_len 将文件截断为 0 字节
        file.set_len(0)?;
        // 再次 use std::fs::File::sync_all 刷新截断
        file.sync_all()?;

        // 丢弃 file 实例，它会自动关闭
        drop(file);

        // 返回 Ok(())
        Ok(())

        } else {
            // 如果 inner 字段是 None，说明文件已经被关闭或丢弃了，返回一个 error
            Err(std::io::Error::new(
                std::io::ErrorKind::Other,
                "File already closed or dropped",
            ))
        }
    }
}

// 为 File 实现 Drop trait，用于在值离开作用域时运行一些代码
impl Drop for File {
    // drop 方法，接受一个可变引用到 self 作为参数
    fn drop(&mut self) {
        if let Some(inner) = self.inner.take() {
            let name = inner.name;
            let fd = inner.fd;
            println!("Closing file {} with fd {}", name, fd);
            // use std::os::unix::io::FromRawFd 将 fd 转回 std::fs::File
            let file: std::fs::File = unsafe {
                std::os::unix::io::FromRawFd::from_raw_fd(
                    fd
                )
            };
            drop(file);
        } else {
            // 如果 inner 字段是 None，说明文件已经被关闭或丢弃了
        }
    }
}

// 用于测试 File 类型的主函数
fn main() {
    // 创建一个名为 "test.txt" 的文件，包含一些内容
    std::fs::write("test.txt", "Hello, world!").unwrap();

    // 打开文件并获取一个 File 实例
    let file = File::open("test.txt").unwrap();

    println!(
        "File name: {}, fd: {}", 
        file.inner.as_ref().unwrap().name, 
        file.inner.as_ref().unwrap().fd
    );

    // 关闭文件并处理任何错误
    // 收到调用析构函数 close
    match file.close() {
        Ok(_) => println!("File closed successfully"),
        Err(e) => println!("Error closing file: {}", e),
    }

    // check 关闭后的文件大小
    let metadata = std::fs::metadata("test.txt").unwrap();
    println!("File size: {} bytes", metadata.len());
}
```


* 解决方案二：
  * 所有字段都可以 `take`
  * 如果类型具有合理的 `空值`，那么这个方案的效果更好
  * 缺点：如果你必须将每个字段都包装在 `Option` 中，然后对这些字段的每次访问都进行匹配 `unwrap`，那么这样就很繁琐
  * 下面是代码例子
  
```rust
use std::os::fd::AsRawFd;

struct File {
    name: Option<String>,
    fd: Option<i32>,
}

impl File {
    fn open(name: &str) -> Result<File, std::io::Error> {
        // 使用 std::fs::OpenOptions 打开文件，具有读写权限
        let file = std::fs::OpenOptions::new()
            .read(true)
            .write(true)
            .open(name)?;

        // 使用 std::os::unix::io::AsRawFd 获取文件描述符
        let fd = file.as_raw_fd();
        
        // 返回一个 File 实例，包含 name 和 fd 字段
        Ok(File { 
            name: Some(name.to_string()),
            fd: Some(fd),
        })
    }

    // 一个显示的析构函数，关闭文件并返回任何错误
    fn close(mut self) -> Result<(), std::io::Error> {
        // use std::mem::take 取出 name 字段的值，并检查是否是 Some(name)
        if let Some(name) = std::mem::take(&mut self.name) {
        // use std::mem::take 取出 fd 字段的值，并检查是否是 Some(fd)
            if let Some(fd) = std::mem::take(&mut self.fd) {

                println!("Closing file {} with fd {}", name, fd);
                // use std::os::unix::io::FromRawFd 将 fd 转回 std::fs::File
                let file: std::fs::File = unsafe {
                    std::os::unix::io::FromRawFd::from_raw_fd(
                        fd
                    )
                };
                // use std::fs::File::sync_all 将任何挂起的写入刷新到磁盘
                file.sync_all()?;
                // 使用 std::fs::File::set_len 将文件截断为 0 字节
                file.set_len(0)?;
                // 再次 use std::fs::File::sync_all 刷新截断
                file.sync_all()?;

                // 丢弃 file 实例，它会自动关闭
                drop(file);

                // 返回 Ok(())
                Ok(())
            } else {
                // 如果 inner 字段是 None，说明文件已经被关闭或丢弃了，返回一个 error
                Err(std::io::Error::new(
                    std::io::ErrorKind::Other,
                    "File already closed or dropped",
                ))
            }
        } else {
            // 如果 inner 字段是 None，说明文件已经被关闭或丢弃了，返回一个 error
            Err(std::io::Error::new(
                std::io::ErrorKind::Other,
                "File already closed or dropped",
            ))
        }
    }
}

// 为 File 实现 Drop trait，用于在值离开作用域时运行一些代码
impl Drop for File {
    // drop 方法，接受一个可变引用到 self 作为参数
    fn drop(&mut self) {
        if let Some(name) = std::mem::take(&mut self.name) {
            if let Some(fd) = std::mem::take(&mut self.fd) {
                // 打印
                println!("Closing file {} with fd {}", name, fd);
                // use std::os::unix::io::FromRawFd 将 fd 转回 std::fs::File
                let file: std::fs::File = unsafe {
                    std::os::unix::io::FromRawFd::from_raw_fd(
                        fd
                    )
                };
                drop(file);
            }
        } else {
            // 如果 fd 字段是 None，说明文件已经被关闭或丢弃了
        }
    }
}

// 用于测试 File 类型的主函数
fn main() {
    // 创建一个名为 "test.txt" 的文件，包含一些内容
    std::fs::write("test.txt", "Hello, world!").unwrap();

    // 打开文件并获取一个 File 实例
    let file = File::open("test.txt").unwrap();

    println!(
        "File name: {}, fd: {}", 
        file.name.as_ref().unwrap(), 
        file.fd.as_ref().unwrap()
    );

    // 关闭文件并处理任何错误
    // 收到调用析构函数 close
    match file.close() {
        Ok(_) => println!("File closed successfully"),
        Err(e) => println!("Error closing file: {}", e),
    }

    // check 关闭后的文件大小
    let metadata = std::fs::metadata("test.txt").unwrap();
    println!("File size: {} bytes", metadata.len());
}
```


* 解决方案三：
  * 即将数据持有在 `ManuallyDrop` 类型中，它会解引用内部类型，不必再使用 `unwrap`
  * 而在 `drop` 中销毁时，可以用 `ManuallyDrop::take` 函数来获取所有权
  * 缺点：`ManuallyDrop::take` 函数是 `unsafe` 的
  * 下边看例子


```rust
use std::{os::fd::AsRawFd, mem::ManuallyDrop};

struct File {
    // 包装在 ManuallyDrop 中
    name: ManuallyDrop<String>,
    fd: ManuallyDrop<i32>,
}

impl File {
    fn open(name: &str) -> Result<File, std::io::Error> {
        // 使用 std::fs::OpenOptions 打开文件，具有读写权限
        let file = std::fs::OpenOptions::new()
            .read(true)
            .write(true)
            .open(name)?;

        // 使用 std::os::unix::io::AsRawFd 获取文件描述符
        let fd = file.as_raw_fd();
        
        // 返回一个 File 实例，包含 name 和 fd 字段
        Ok(File { 
            name: ManuallyDrop::new(name.to_string()),
            fd: ManuallyDrop::new(fd),
        })
    }

    // 一个显示的析构函数，关闭文件并返回任何错误
    fn close(mut self) -> Result<(), std::io::Error> {
        // use std::mem::replace 将 name 字段替换为空字符串，并获取原来的值 
        // 或用  String::new() 代替 "".to_string()
        let name = std::mem::replace(&mut self.name, ManuallyDrop::new("".to_string()));

        // use std::mem::replace 将 fd 字段替换为无需的值，并获取原来的值
        let fd = std::mem::replace(&mut self.fd, ManuallyDrop::new(-1));

        println!("close Closing file {:?} with fd {:?}", name, fd);

        // use std::os::unix::io::FromRawFd 将 fd 转回 std::fs::File
        let file: std::fs::File = unsafe {
            std::os::unix::io::FromRawFd::from_raw_fd(
                *fd
            )
        };
        // use std::fs::File::sync_all 将任何挂起的写入刷新到磁盘
        file.sync_all()?;
        // 使用 std::fs::File::set_len 将文件截断为 0 字节
        file.set_len(0)?;
        // 再次 use std::fs::File::sync_all 刷新截断
        file.sync_all()?;

        // 丢弃 file 实例，它会自动关闭
        drop(file);

        // 返回 Ok(())
        Ok(())


    }
}

// 为 File 实现 Drop trait，用于在值离开作用域时运行一些代码
impl Drop for File {
    // drop 方法，接受一个可变引用到 self 作为参数
    fn drop(&mut self) {
        // 使用 ManuallyDrop::take 取出 name 字段的值，并检查是否是空字符串
        let name = unsafe { ManuallyDrop::take(&mut self.name) };

        // 使用 ManuallyDrop::take 取出 fd 字段的值，并检查是否是无效的值
        let fd = unsafe { ManuallyDrop::take(&mut self.fd) };

        println!("drop Closing file {} with fd {}", name, fd);

        // 如果 fd 不是无效的值，说明文件还没有被关闭或丢弃，需要执行一些操作
        if fd != -1 {
            let file: std::fs::File = unsafe {
                std::os::unix::io::FromRawFd::from_raw_fd(
                    fd
                )
            };
            // 丢弃 file 实例，它会自动关闭
            drop(file);

        }
    }
}

// 用于测试 File 类型的主函数
fn main() {
    // 创建一个名为 "test.txt" 的文件，包含一些内容
    std::fs::write("test.txt", "Hello, world!").unwrap();

    // 打开文件并获取一个 File 实例
    let file = File::open("test.txt").unwrap();

    println!(
        "File name: {}, fd: {}", 
        *file.name, 
        *file.fd
    );

    // 关闭文件并处理任何错误
    // 收到调用析构函数 close
    match file.close() {
        Ok(_) => println!("File closed successfully"),
        Err(e) => println!("Error closing file: {}", e),
    }

    // check 关闭后的文件大小
    let metadata = std::fs::metadata("test.txt").unwrap();
    println!("File size: {} bytes", metadata.len());
}
```


#### 小结
* 如果选择上述三种方案
  * 根据实现情况选择方案
  * 如果代码足够简单，可轻松检查代码安全性，那么 `ManuallyDrop` 方案也挺好
