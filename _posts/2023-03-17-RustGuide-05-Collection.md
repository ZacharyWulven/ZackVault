---
layout: post
title: RustGuide-05-Collection
date: 2023-03-17 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 8 常用的集合

## 本章集合不同于数组和元组，它们执行 Heap 上的数据

## Vector
* 写法 `Vec<T>`
* 由标准库提供，可存储多个值，而且只能存储相同类型数据
* 这些值在内存中是连续存放的

### 创建 Vector
* Vec::new()


```rust
    /*
        虽然 Rust 有类型推断
        但 new 返回空数组，无法推断类型，所以这里需要显示声明类型
        Vec 里可以存任意类型
     */ 
    let v: Vec<i32> = Vec::new();
    println!("{:?}", v);

```

### 使用初始值创建 Vector，使用 vec! 宏


```rust
    let v2 = vec![1,2,3];
    println!("{:?}", v2);
```


### 更新 Vector 

```rust
    let mut v3 = Vec::new();
    // 使用 push 添加元素
    v3.push(1); 
```

### 删除 Vector
* 与任何其他 struct 一样，当 Vector 离开作用域后
1. 它就被清理了
2. 它的所有元素也被清理了

### 读取 Vector 里的元素
* 两种方式可以引用 Vector 的值
1. 索引
2. get 方法


```rust
    let v4 = vec![1,2,3,4,5];
    let third = &v4[2];  // 使用索引方式访问，如果索引越界程序会 panic

    println!("The third element is {}", third);

    /*
        使用 get 方法访问
        get 方法参数是 索引，返回 Option<T>，所以可以使用 match
        如果索引越界，则 get 方法返回 None
     */
    match v4.get(100) { 
        Some(third) => println!("The third element is {}", third),
        None => println!("No third element"),
    }
```


> 使用索引访问，如果索引越界程序会 panic。使用 get 方法访问，如果索引越界，则 get 方法返回 None。
{: .prompt-info }



### 所有权和借用规则
* 不能在同一作用域内同时拥有可变和不可变的引用

```rust
    let mut v5 = vec![1, 2, 3, 4, 5];
    let first = &v5[0];  // 这里 first 是不可变的借用, 引用其第一个元素
    
    // 如果这行可以的话，可能造成 v5 容量不够而重新分配内存，导致 first 还指向原来的位置，这样就有问题了
    //v5.push(6);   // 可以变借用, 这里报错, 因为已经借用为 first 不可变的了 
    
    println!("The first element is {}", first);
```


### 遍历 Vector 中的值

```rust
    // 遍历 vector, 可变引用
    let mut v6 = vec![100, 32, 57];
    for i in &mut v6 {
        // *i 为解引用操作
        *i += 50;
    }
    println!("v6 {:?}", v6);   // [150, 82, 107]
    
    
    // 遍历 vector, 不可变引用
    let v7 = vec![1, 2, 3, 4];
    for i in &v7 {
        println!("v7 item is {}", i)
    }
```


### 通过 `iter()` 遍历 Vector 中的值
* `iter()` 返回的即一个指针


![image](/assets/images/rust/vec_iter.png)


### 通过 `iter()` 遍历的所有权情况
* 遍历时会丧失 `v` 的写权限
* 而 `v.push` 需要 `v` 有写权限, 所以下边报错
 

![image](/assets/images/rust/vec_iter2.png)


### 不使用指针，而通过 `Range` 遍历 Vector 中的值


![image](/assets/images/rust/vec_range.png)


### 使用 enum 在 Vector 中存储多个类型的数据
* enum 变体可以附加不同类型的数据


```rust
enum SpreadsheetCell {
    Int(i32),
    Float(f64),
    Text(String)
}

fn example() {

    let v = vec![
        SpreadsheetCell::Int(32),
        SpreadsheetCell::Text(String::from("hello")),
        SpreadsheetCell::Float(3.14),
    ];
    
}
```


### 思考题: 下边代码能否通过编译

```rust
fn main() {
    let mut v = Vec::new();
    let s = String::from("Hello ");
    v.push(s);   // 这里发生了 move
    v[0].push_str("world");
    println!("original: {}", s);   // Error：v.push(s) 时，s 已被移动
    println!("new: {}", v[0]);
}
```

### 思考题：下边代码能否通过编译

```rust
    let v = vec![String::from("hello ")];
    let mut s = v[0];            // Error: move occurs because value has type `String`, which does not implement the `Copy` trait
    // let mut s = v[0].clone();    // 使用 clone 解决
    s.push_str("world");
    println!("new: {}", s);
```

### 思考题 
* v2 的元素实际是引用类型

```rust
    let mut v = vec![1,2,3];
    let mut v2 = Vec::new();
    for i in &mut v {
        v2.push(i);
    }
    *v2[0] = 5;

    let a = *v2[0];
    let b = v[0];
    println!("a = {}, b = {}", a, b);  // 5, 5
```


## String
* Rust 中的字符串使用 UTF-8 编码

* 字符串就是基于 Byte 的集合，并添加了一些方法，这些方法可以把字节解析为文本

### 什么是字符串
* Rust 核心语言层面只有一个字符串类型，字符串切片 str 或 &str
* 字符串切片：是对存储在其他地方、UTF-8 编码的字符串的引用
* 字符串字面量：存储在二进制文件里，也是字符串切片

### String 类型
* 由标准库定义
* 内容可增长，可拥有，可修改
* 使用 UTF-8 编码

* 通常说的字符串是指 String 和 &str（字符串切片）
* Rust 标准库还包含了很多其他字符串类型，例如 OsString、OsStr、CString、CStr
1. String 结尾的类型：可获得其所有权的
2. Str 结尾的类型：指可借用的
3. 这些类型使用不同的编码来存储文本或者在内存中的展现形式不同

* Library crate（也就是第三方库）针对存储字符串还提供了更多选项

### 创建一个字符串 String
* 很多 `Vec<T>` 的操作都可用于 String
* String::new() 函数创建 String

```rust
    let mut s = String::new();
```

* 使用初始值来创建 String
1. toString() 方法，可用于实现了 Display trait 类型，包括字符串字面量，作用是将其转换为 String 类型
2. String::from("") 函数，创建 String


```rust
    let data = "initial contents"; // &str
    let s1 = data.to_string();  // String
    
    let s2 = String::from("initial contents");

```

### 更新 String
* push_str() 方法：可以把一个字符串切片附加到 String 上

```rust
    let mut foo = String::from("foo");
    let bar = String::from(" bar");
    // push_str 的参数是 &str，所以不会获取其所有权
    foo.push_str(&bar);
    println!("{}, {}", foo, bar); // foo bar,  bar；so 这里还能使用 bar
```

* push() 方法：把单个字符附加到 String
```rust
    let mut lo = String::from("lo");
    lo.push('l');
    println!("{}", lo); // lol
```

* `+` 运算符，连接字符串
1. `+` 运算符实际上是使用了类似这个签名的方法 `fn add(self, s: &str) -> String {...}`，所以获取了 self 也就是 + 前的变量的所有权
2. `+` 拼接后，`+` 前的变量就不可以再使用了
3. 为什么 `+` 后可以是字符串引用，是因为 Rust 解引用强制转换把字符串引用转为了字符串切片

```rust
    // + 前为 String 类型，+ 后需要为字符串切片（字符串引用也可以编译通过，因为强制换行了）
    let mut s11 = String::from("hello");
    let mut s22 = String::from(" world");
    // 获得 s11 的所有权，然后拼上 s22 的内容，然后把值和所有权同时返回赋值给了 s33
    let s33 = s11 + &s22;
    println!("{}", s33);
    println!("{}", s11); // Note： + 拼接后 s11 不可以再使用了
    println!("{}", s22);
```

* 使用 format! 宏，连接多个字符串
1. 与 println!() 类似，但返回字符串


```rust
    let a = String::from("a");
    let b = String::from("b");
    let c = String::from("c");

    // 这样比较麻烦
    // let abc1 = a + "-" + &b + "-" + &c;
    // println!("{}", abc1);

    // format! 不会取得任何参数的所有权
    let abc = format!("{}-{}-{}", a, b, c);
    println!("{}", abc);
```


> format! 不会取得任何参数的所有权，并返回一个 String 类型
{: .prompt-info }


### 对 String 按索引的形式进行访问
* 直接按 index 访问会报错，因为 String 默认没有实现 Index<integer> 这个 trait
* 不支持 index 语法访问

### String 的内部表示
* String 其实是 `Vec<u8>` 的包装，就是对字节的包装
* 一个 Unicode 标量值对应两个字节

```rust
    let str = String::from("Hello");
    // 一个字母（Unicode 标量值），有的语言一个字母占两个字节
```

* 以 `Здравствуйте` 为例其一个字母占两个字节，如果 String 可以按索引访问就会取出其一半的 Unicode 标量值，这样是不合法的，所以 Rust 在编译阶段就避免这么做

```rust
    let hello = "Здравствуйте";
    let s = &hello[1];          // 报错,字符串不能通过 index 访问
```


### 字节（Bytes）、标量值（Scalar Values）、字形簇（Grapheme Clusters）
* 针对 UTF-8 编码的字符串可以有三种看待它们的方式
1. 字节
2. 标量值
3. 字形簇（最接近所谓字母）


```rust
    let str = String::from("Hello");
    // 一个字母（Unicode 标量值）占

    println!("Bytes");
    for byte in str.bytes() {  // 以字节形式
        println!("{}", byte);  // 208 ...
    }

    for char in str.chars() {  // 以 Unicode 标量值形式
        println!("{}", char);  // 3 ...
    }
```

### Rust 不允许对 String 进行索引的最后一个原因
* 索引操作的时间复杂度应该是 O(1)，但是 String 无法保证，需要遍历所有内容，来确定有多少个合法的字符

### 切割 String
* 可使用 `[]` 和一个范围来创建字符串切片
* 必须严谨使用
* 如果切割时跨越了字符边界，程序就会 panic


```rust
    let hello = "Здравствуйте";
    let sStr = &hello[0..3];  // 这里程序会 panic，因为这个语言一个字母占两个字节，切片时需要根据 char 边界进行切割
    let sStr = &hello[0..4];  // Зд
    println!("{}", sStr);
```

### 遍历 String
* 如果想获取标量值，使用 chars() 方法
* 如果想获取字节，使用 bytes() 方法
* 如果想获取字形库，这个比较复杂，标准库没有提供


```rust
    let hello = "Здравствуйте";

    // 一个字母（Unicode 标量值）占

    println!("Bytes");
    for byte in hello.bytes() { // 以字节形式
        println!("byte {}", byte);  // 208 ...
    }

    for char in hello.chars() { // 以 Unicode 标量值形式
        println!("char {}", char);  // З ...
    }
```



### String 不简单
* Rust 选择将正确处理 String 数据作为所有 Rust 程序的默认行为
1. 即程序员需要在处理 UTF-8 前投入更多经历
* 可防止在开发后期处理涉及非 ASCII 字符的错误


## HashMap

### `HashMap<K,V>`
* 键值对形式存储数据，一个 Key 对应一个 Value
* 使用 Hash 函数，以确定如何存放 K 和 V

### 创建 HashMap
* 使用 HashMap::new() 函数
* 使用 insert() 方法添加数据


```rust
use std::collections::HashMap; // HashMap 用的较少，不在 Prelude 中 使用时需要手动导入

//let mut scores: HashMap<&str, i32> = HashMap::new();
let mut scores = HashMap::new(); // 这里可以不显示声明 K V 的类型，通过 insert 系统自动推断
scores.insert("tom", 40);
println!("{:#?}", scores);    
```

### 补充
* HashMap 用的较少，不在 Prelude 中
* 标准库对 HashMap 支持的少，也没有宏可以快速创建 HashMap
* HashMap 的数据存在 Heap 上
* HashMap 是同构的，即同一 HashMap 中
1. 所有的 Key 必须是同一类型
2. 所有的 Value 必须是同一类型


### 使用 collect 方法创建 HashMap
* 在元素类型为 Tuple 的 Vector 上使用 collect 方法，可以组建一个 HashMap
1. 要求是 Tuple 有两个值，第一个值为 Key，第二个值为 Value
* collect 可以把数据整合为很多种数据类型，包括 HashMap
1. so 返回值需要显式声明类型

```rust
    let team = vec![String::from("Blue"), String::from("Yellow")];
    let init_score = vec![10, 50];
    // zip 可以创建一个元组的数组
    let scores2: HashMap<_, _> = team.iter().zip(init_score.iter()).collect(); 
    println!("{:#?}", scores2);  // // {"Blue": 10, "Yellow": 50}
    
    
    // or
    let vec = vec![("key1", "value1"), ("key2", "value2")];
    let map = vec.into_iter().collect::<HashMap<_, _>>();
```

### HashMap 的所有权
* 对于实现了 Copy trait 的类型（例如 i32），值会被复制到 HashMap 中
* 对于拥有所有权的值（例如 String），值会被 move，所有权会转移给 HashMap
* 如果将值的引用插入到 HashMap，值本身不会移动
1. 在 HashMap 有效期间内，被引用的值必须保持有效才行

```rust
    let mut map = HashMap::new();
    
    let field_key = String::from("Favorite color");
    let field_value = String::from("Blue");

    map.insert(field_key, field_value);

    // 这里会报错，因为 field_key，field_value 已经 Move 发生了所有权改变
    println!("{}, {}", field_key, field_value);
    
    // 如果传引用进去，下边可以打印不报错，因为没有发生 Move
    // 由于这里是引用，所以需要 hashmap 有效期间，引用的值必须保证有效才行
    map.insert(&field_key, &field_value);
    println!("{}, {}", field_key, field_value);
```


### 访问 HashMap 中的值
* 使用 get 方法访问
1. 参数 K
2. 返回 `Option<&V>`

```rust
    let mut score_map = HashMap::new();
    score_map.insert(String::from("Blue"), 10);
    score_map.insert(String::from("Yellow"), 50);

    let team_name = String::from("Blue");
    let score = score_map.get(&team_name);

    match score {
        Some(s) => println!("team score is {}", s),
        None => println!("team is not exist"),
    }
    
    // 补充：如果没有值，默认为 0
    let score = score_map.get(&team_name).copied().unwrap_or(0);
```

### 遍历 HashMap
* for 循环


```rust
    for (k, v) in &score_map {
        println!("key={}, value={}", k, v);
    }
```

### 更新 HashMap
* HashMap 大小可以变
* 每个 Key 只能对应一个 Value

#### 更新 HashMap 中的数据时的 case
case1：Key 已经存在，对应一个 Value
* 替换现有的 Value：如果向 HashMap 中插入一对 KV，再插入同样的 K，但不同的 V，那么原来的 V 会被替换掉

```rust
    let mut score_map = HashMap::new();
    score_map.insert(String::from("Blue"), 10);
    score_map.insert(String::from("Blue"), 50);
    println!("{:#?}", score_map); // 50
```

case2：Key 不存在 
* 只在 K 不对应任何值的情况下，才插入 V
* 使用 entry 方法，此方法会检查指定 K 是否对应一个 V。参数为 K，返回 enum Entry：代表值是否存在
* entry 的 or_insert() 方法，返回
1. 如果 K 存在，返回到对应的 V 的一个可变引用
2. 如果 K 不存在，将方法参数作为 K 的新值插入 HashMap，返回到这个值的可变引用

```rust
    /*
        entry 方法判断 Yellow 这个 key 是否存在于 score_map
        如果不存在我们就插入 50，如果存在就返回存在的值
     */
    let e = score_map.entry(String::from("Yellow"));
    // e is  VacantEntry("Yellow"), VacantEntry 就是空, 即 Yellow 这个 key 不存在
    println!("e {:#?}", e);
    e.or_insert(50);
    score_map.entry(String::from("Blue")).or_insert(50);
```

* 基于已有 V 更新 V

```rust
    let text = "hello world wonderful world";
    let mut map = HashMap::new();
    for word in text.split_whitespace() {
        // 第一次 word 因为 key 不存在，会插入值为 0
        // 非第一次 word key 已经存在，entry 直接返回 V 也就是 hashMap 里的值
        let count = map.entry(word).or_insert(0);
        *count += 1;
    }
    println!("{:#?}", map);
```


### Hashing 函数
* 默认情况下，HashMap 使用加密功能强大的 SipHash 函数，可以抵抗拒绝服务（DoS）攻击，但不是最快的
1. 不是可用的最快的 Hash 算法
2. 但具有更好的安全性

* 可以指定不同的 hasher（实现 BuildHasher Trait）来切换到另一个 Hashing 函数
1. hasher 就是指实现了 BuildHasher trait 的类型
