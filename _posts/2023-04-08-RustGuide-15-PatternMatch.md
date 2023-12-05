---
layout: post
title: RustGuide-15-PatternMatch
date: 2023-04-08 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 18 模式匹配

## 模式
* 模式是 Rust 中一种特殊语法，用于匹配复杂和简单类型的结构
* 将模式与匹配表达式和其他构造结合使用，就可以更好的控制程序的控制流
* 模式由以下元素（或它们的组合）组成：
1. 字面量
2. 解构的数组、enum、struct、tuple
3. 变量
4. 通配符
5. 占位符

* `想要使用模式需要将其与某个值进行比较`
1. 如果能够模式匹配，就可以在代码中使用这个值的相应部分

## 能够用到模式的地方

### match 它的 Arm
* match 表达式的要求，详尽包含所有可能性
* match 的特殊模式：下划线 `_`
1. 用于匹配所有可能的值
2. 不会绑定到变量
3. 通常用于 match 的最后一个 arm（分支），或用于忽略某些值

### 条件 `if let` 表达式
* 主要作为一种简短的方式来等价的代替只有一个匹配项的 `match`
* `if let` 可选的可以拥有 `else`，包括
1. `else if`
2. `else if let` 
* 但 `if let` 不会检查穷尽性

```rust
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        println!("Using ur favorite color, {}, as the background", color);
    } else if is_tuesday {
        println!("Tuesday is green day!");
    } else if let Ok(age) = age {
        if age > 30 {
            println!("Using purple as the background color");
        } else {
            println!("Using orange as the background color");
        }
    } else {
        println!("Using blue as the background color");
    }
}
```

### while let 条件循环
* 只要这个模式能继续满足匹配的条件，那它允许 while 循环一直运行

```rust
fn test_while_let() {
    let mut stack = Vec::new();

    stack.push(1);
    stack.push(2);
    stack.push(3);

    // 只要 stack.pop() 返回的不是 None，这个循环就一直运行
    while let Some(top) = stack.pop() {
        println!("{}", top);
    }
}
```

### for 循环
* for 循环是 Rust 中最常见的循环
* for 循环中，模式就是紧随 for 关键字后边的值

```rust
fn test_for() {
    let v = vec!['a', 'b', 'c'];
    /*
        (index, value) 就是它的模式
        调用 iter().enumerate() 后每次返回一个元组
     */
    for (index, value) in v.iter().enumerate() {
        println!("{} is at index {}", value, index);
    }
}
```

### let 语句
* let 语句实际也是一个模式，形式是 `let PATTERN = EXPRESSION;`

```rust
fn test_let() {
    let a = 5;
    // 这的模式是 x、y、z 分别对应 1、2、3 的值
    let (x, y, z) = (1, 2, 3);

    // 这样就会报错，因为变量与值的数对不上
    //let (x, y) = (1, 2, 3);
}
```

### 函数参数
* 函数参数也可以是模式

```rust
// 参数 x 其实就是一个模式
fn foo(x: i32) {
    
}

/*
    参数是一个元组
    这里通过模式将元组的值提取给 x、y 变量里
    这样在函数内部就可以使用 x、y 了
*/
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({}, {})", x, y);
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```

## 可辨驳性（可失败性）：模式是否会无法匹配（或叫匹配失败）

### 模式有两种
1. 可辩驳的（可能会失败的）
2. 不可辩驳的（不会失败的）

### 无可辩驳的
* 即能匹配任何可能传递的值的模式，即怎么匹配都能成功
1. 例如 `let x = 5`，能匹配表达式右侧所有可能的值

### 可辩驳的
* 即对于某些可能的值，无法进行匹配的模式
1. 例如 `if let Some(x) = a_value`，如果右边的值是 `None` 就会发生不匹配的情况，


> `函数参数`、`let 语句`、`for 循环` 只接受无可辩驳的模式。`if let` 和 `while let` 接受可辨驳和无可辩驳的模式，但 `if let` 和 `while let` 接受无可辩驳的模式时编译器会发生警告
{: .prompt-info }


```rust
    let a = Some(5);
    
    // 因为 let 是可辩驳的模式，而 Some(x) 是可辩驳的，所以这样会报错，改成 if let 就行了
    // let Some(x) = a;
    
    // 这样就行了
    if let Some(x) = a {
    
    }
    
    
    // 这样总会成功，so 这样没有意义
    if let x = 5 {
        
    }
```


> match 表达式的除了最后一个分支其他都应该是可辩驳的（可失败的），`最后一个分支必须是不可辩驳的，因为它需要匹配所有剩余的情况`
{: .prompt-info }


## 模式匹配的语法

### 匹配字面值
* 模式可以直接匹配字面值

```rust
fn main() {

    let x = 1;

    match x {
        1 => println!("one"),
        2 => println!("two"),
        3 => println!("three"),
        _ => println!("anything"),
    }

}
```

### 匹配命名变量
* 命名的变量是可匹配任何值的无可辩驳模式

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(y) => println!("Matched, y = {:?}", y),  // 5
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {:?}", x, y);
}
```

### 多重模式
* 在 match 表达式中，使用 `|` 语法（就是或的意思），可以匹配多种模式

```rust
fn main() {
    let x = 1;

    match x {
        1 | 2 => println!("one or two"),  // 匹配多种模式，这里匹配 1 或 2
        //2 => println!("two"),
        3 => println!("three"),
        _ => println!("anything"),
    }
}
```

### 使用 `..=` 来匹配某个范围的值

```rust
fn main() {
    let x = 5;
    match x {
        // 当 x >= 1 && x <=5 时走这个分支
        1..=5 => println!("one through five"),
        _ => println!("something else"),
    }

    let x = 'c';
    match x {
        // 当 x 是 a~j 时候就走这个分支
        'a'..='j' => println!("early ASCII letter"),
        'k'..='z' => println!("late ASCII letter"),
        _ => println!("something else"),
    }
}
```

### 解构以分解值
* 我们可以使用模式来解构 struct、enum、tuple、从而引用这些类型的值的不同部分

```rust
fn main() {
    let p = Point { x: 0, y: 7 };
    
    // 这里使用模式对 p 进行结构
    let Point { x: a, y: b} = p;
    assert_eq!(0, a);
    assert_eq!(7, b);
    println!("test point finished 1");

    // 简写版本
    // 如果上边的 a 名称改为 x，b 名称改为 y，这样就可以简写，类似初始化时候同名的 Field 简写
    let Point { x, y } = p;
    assert_eq!(0, x);
    assert_eq!(7, y);
    println!("test point finished 2");


    match p {
        // 匹配 x 是任何值，y 必须是 0
        Point { x, y: 0 } => println!("On the x axis at {}", x),
        Point { x: 0, y } => println!("On the y axis at {}", x),
        Point { x, y } => println!("On neither axis: ({}, {})", x, y),
    }

}
```

### 解构 enum

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32},     // 结构体的枚举变体
    Write(String),              // 元组的枚举变体，有一个元素
    ChangeColor(i32, i32, i32), // 元组的枚举变体，有三个元素
}

fn main() {
    let msg = Message::ChangeColor(0, 160, 255);

    match msg {
        Message::Quit => {
            println!("The Quit variant has no data to destructure.")
        }
        Message::Move { x, y } => {
            println!("Move in the x direction {} and in the y direction {}", x, y)
        }
        Message::Write(text) => println!("Text message: {}", text),
        Message::ChangeColor(r, g, b) => {
            println!("Change the color to red {}, green {}, and blue {}", r, g, b)
        }
    }
}
```

### 解构嵌套的 struct 和 enum

```rust

enum Color {
    RGB(i32, i32, i32),
    HSV(i32, i32, i32),
}

enum Message {
    Quit,
    Move { x: i32, y: i32},     // 结构体的枚举变体
    Write(String),              // 元组的枚举变体，有一个元素
    ChangeColor(Color)
}

fn main() {

    let msg = Message::ChangeColor(Color::HSV(0, 160, 255));

    match msg {
        // 一层一层的写就行
        Message::ChangeColor(Color::RGB(r, g, b)) => {
            println!("Change the color to red {}, green {}, and blue {}", r, g, b)
        }
        Message::ChangeColor(Color::HSV(h, s, v)) => {
            println!("Change the color to hue {}, saturation {}, and value {}", h, s, v)
        }
        _ => (),
    }
}
```

### 解构 struct 和 tuple

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    /*
        这里外层是一个元组，元组的第一个元素还是元组，第二个元素是结构体
     */
    let ((feet, inches), Point { x, y }) = ((3, 10), Point { x: 3, y: -10 });
}
```

### 在模式中忽略值
#### 有几种方式可以在模式中忽略整个值或部分值
* 使用 `_` 就是忽略整个值

```rust
// 忽略 第一个值
fn foo(_: i32, y: i32) {
    println!("this code only uses the y parameter: {}", y);
}

foo(3, 4);
```

* 使用 `_` 配合其他模式可以忽略部分值

```rust
fn main() {
    let mut setting_value = Some(5);
    let new_setting_value = Some(10);

    match (setting_value, new_setting_value) {
        // 匹配只要它俩都是 Some() 就可以，值不关心
        (Some(_), Some(_)) => {
            println!("Can not over write an existing customized value");
        }
        _ => {
            setting_value = new_setting_value;
        }
    }

    println!("setting is {:?}", setting_value);



    let numbers = (2, 4, 8, 16, 32);
    match numbers {
        // 匹配只需要知道第 1、3、5 个数，忽略其他数
        (first, _, third, _, fifth) => {
            println!("Some number: {}, {}, {}", first, third, fifth);
        }
    }

}
```

* 使用以 `_` 开头的命名来忽略未使用的变量

```rust
fn main() {
    // 未使用的变量以 _ 开头可以消除编译器的警告
    let _x = 5;
    let y = 10; 

    let s = Some(String::from("Hello"));

    // 写法 1
    // 这里 s 就 move 到 _s 了，使用 _s 会发生 bind 操作
    // if let Some(_s) = s {
    //     println!("found a string");
    // }
    //println!("{:?}", s); // 这里报错，因为 s 已经 move

    // 写法 2
    if let Some(_) = s {
        println!("found a string");
    }
    println!("{:?}", s); // 这里不会报错，因为 Some(_) 不会进行 bind 操作，不 bind 就不会移动所有权
}
```

* 使用 `..` 忽略值的剩余部分

```rust
struct Position {
    x: i32,
    y: i32,
    z: i32,
}

fn test_dot() {
    let origin: Position = Position { x: 0, y: 0, z: 0 };
    match origin {
        // 匹配只需要 x 字段
        Position { x, .. } => println!("s is {}", x),
    }

    let numbers = (2, 4, 8, 16, 32);
    match numbers {
        // 匹配只需要第一个和最后一个数
        (first, .., last) => {
            println!("Some numbers: {}, {}", first, last);
        }
    }

    /*
    match numbers {
        // 这样会报错，因为不知道你要中间那个元素，会发生歧义
        // 使用 .. 时不能发生歧义
        (.., second, ..) => {
            println!("Some number: {}", second);
        }
    }
    */
}
```

### 使用 match 守卫来提供额外的条件
* `match` 守卫就是 `match arm` 模式后额外的 `if` 条件，想要匹配该条件也必须满足这个 `if`
* `match` 守卫用于比单独模式更复杂的场景

```rust
fn main() {
    // case 1
    let num = Some(4);
    match num {
        /*
            if x < 5 就是 match 的守卫，需要 x 小于 5
            
         */
        Some(x) if x < 5 => println!("less than five: {}", x),
        Some(x) => println!("x >= 5, is {}", x),
        None => (),
    }

    // case 2
    let x = Some(5);
    let y = 10;
    match x {
        Some(50) => println!("Got 50"),
        /*
            Note 守卫不是模式，所以守卫不会引入新的变量
            而 n 这个变量是在 Some(n) 模式匹配时引入的
         */
        Some(n) if n == y => println!("Matched, n = {:?}", n),
        _ => println!("Default case, x = {:?}", x),
    }
    println!("at the end: x = {:?}, y = {:?}", x, y);


    // case 3，多重模式使用 match 守卫
    let m = 4;
    let n = false;

    match m {
        // 如果 m 是 4 或 5 或 6 并且 n 是 true 的话就打印 yes
        4 | 5 | 6 if n => println!("yes"),
        _ => println!("no"),
    }
}
```

### `@`绑定
* `@` 符号让我们可以创建一个变量，该变量可以在测试某个值是否与模式匹配的同时保存该值

```rust
enum Msg {
    Hello { id: i32},
}

fn test_at() {
    let msg = Msg::Hello { id: 5 };

    match msg {
        /*
            当 id 的值在 3~7 时，使用 @ 将值存到 id_var 变量中
         */
        Msg::Hello { id: id_var @ 3..= 7, } => {
            println!("Found an id in range: {}", id_var)
        }
        Msg::Hello { id: 10..=12 } => {
            println!("Found an id in another range")
        }
        Msg::Hello { id } => {
            println!("Found some other id: {}", id)
        }
    }
}
```


