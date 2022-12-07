---
layout: post
title: Go 语言快速入门-01-基础语法
date: 2022-08-15 16:45:30.000000000 +09:00
categories: [Golang, 入门]
tags: [Golang]
---

## 0x01 常量

{% highlight ruby %}
// 全局常量
const PI float64 = 3.14

// 定义枚举
// 所谓枚举也是就是固定的整数值
const (
APPLE1 = 0
BANANA1 = 1
PEAR1 = 2
MANGO1 = 3
)

const (
APPLE2 = iota // iota 是从 0 开始的自增序列
BANANA2 // BANANA=1
PEAR2
MANGO2
)

/\*
iota 影响
1 每次"等号"才加一次
2 继承自上边的公式

_/
const (
APPLE, APPLES = iota + 1, iota + 2 // iota 是从 0 开始的自增序列
BANANA, BANANAS // BANANA= iota
PEAR, PEARS = iota _ 2, iota\*2 + 1
MANGO, MANGOS
)

// APPLE is 1 APPLES is 2 BANANA is 2 BANANAS is 3 PEAR is 4 PEARS is 5 MANGO is 6 MANGOS is 7

{% endhighlight %}

## 0x02 数组

- 数组元素个数是固定的
- 切片是可以动态扩展的

{% highlight ruby %}

func main() {
var a1 [5]int = [5]int{1, 2, 3, 4, 5}

fmt.Println(a1)
fmt.Println("a1[1] is", a1[1])

for i := 0; i < 5; i++ {
fmt.Println(a1[i])
}

for i, v := range a1 {
fmt.Println("i is", i, "v is", v)
}

// 定义二维数组
a34 := [3][4]int{ {1, 2, 3, 4}, {5, 6, 7, 8}, {9, 10, 11, 12} }

for i, v := range a34 {
fmt.Println("a34 i is", i, "v is", v)
}
}
{% endhighlight %}

## 0x03 for 循环

- go 语言中循环只有 for，但可以应用其他语言 for while 各种玩法
- 格式：for init; cond; post
  {% highlight ruby %}
  func main() {
  sum := 0
  for i := 1; i <= 100; i++ {
  sum += i
  }
  fmt.Println("sum is", sum)

  // while(cond)
  i := 0
  sum1 := 0
  for i <= 100 {
  sum1 += i
  i++
  }
  fmt.Println("sum1 is", sum1)

  // while(true)
  sum2 := 0
  i = 1
  for {
  sum2 += i
  // 可以使用 break 打断
  if i == 100 {
  break
  }
  i++
  }
  fmt.Println("sum2 is", sum2)
  }
  {% endhighlight %}

## 0x04 函数

{% highlight ruby %}
语法格式： func [函数名](函数参数) 函数返回值
{% endhighlight %}

1. 函数参数可以 0 个或多个，类型后置；参数名称 1 参数类型，参数名称 2 参数类型
2. 返回值也可以 0 个或多个；
3. 一个返回值，直接写返回类型
4. 多个返回值，用小括号（类似元组）

{% highlight ruby %}
func add(a int, b int) int {
return a + b
}

func sub(a, b int) int { // 相邻参数，类型如果相同可以省去前边那个参数的类型定义
return a - b
}

func addOrSub(a int, b int, f func(x, y int) int) int {
return f(a, b)
}

f := add // add 可以理解为函数指针，此时 f 就是 add， f 其实是一个变量，它的类型是函数类型
ret = f(x, y)
fmt.Println(ret)
f = sub
ret = f(x, y)
fmt.Println(ret)

fmt.Println(addOrSub(30, 20, sub))

// 定义一个匿名函数
val := func(a, b int) int {
return a \* b
}(10, 20)

// 调用匿名函数
fmt.Println("val is", val)

{% endhighlight %}

### 函数闭包：

1. 需要至少两层函数，或者父子函数
2. 子函数访问父函数的变量

{% highlight ruby %}

// 制作一个自增的序列表
func getNextNumber() func() int {
num := 0
return func() int {
num++ // 访问父函数的变量，会导致 num 没有被释放
return num
}
}

// 函数闭包
fmt.Println("函数闭包")
f1 := getNextNumber()
fmt.Println(f1())
fmt.Println(f1())
fmt.Println(f1())
fmt.Println(f1())

f2 := getNextNumber()
fmt.Println(f2())
fmt.Println(f2())
// f1 与 f2 自增是各自独立的
{% endhighlight %}

## 0x05 条件判断

- 条件表达式的结果必须是 布尔值, 跟 swift 一样

{% highlight ruby %}
func main() {
// 条件表达式的结果必须是 布尔值, 跟 swift 一样
a := 10

if a > 0 {
fmt.Println("a is", a)
} else {
fmt.Println("a <= 0")
}

// switch-case 可以处理字符串
// 接收输入，获取水果的名字
/_
switch-case 可以处理字符串
fallthrough 可以穿透，不管下一个 case 是否匹配都会触发下一个 case 的代码
_/
fmt.Println("你喜欢什么水果")
fruit := ""
fmt.Scanf("%s", &fruit)
switch fruit {
case "apple":
fmt.Println("You like", fruit)
fallthrough
case "banana":
fmt.Println("fallthrough")
fmt.Println("You like banana")
case "pear":
fmt.Println("You like", fruit)
case "mango":
fmt.Println("You like", fruit)

default:
fmt.Println("R U kidding me?")
}

}
{% endhighlight %}

## 0x06 map

- map 就是 key:value 形式的数据类型
- map 在使用前必须 make, make 不只针对 map 可以做很多事
- key 类型，value 类型
  {% highlight ruby %}
  var scores map[string]uint = make(map[string]uint)

  // 向 map 添加元素
  // Note map 为 nil 时 不能添加元素

  scores["tom"] = 100
  scores["jack"] = 100
  fmt.Println(scores)
  fmt.Println("tom score is", scores["tom"])

  // mike 这个人不存在与 map 中，所以分数是 0
  fmt.Println("tom score is", scores["mike"])

  // 指示器使用： 在获取 map 数据时，可以使用指示器
  // val 是分数值，ok 是指示器，表示 key 的 value 是否存在在 map 中
  val, ok := scores["mike"]
  fmt.Println("val is", val, "ok is", ok)

  // map 的遍历
  // map 的 cap 不能获取，map 的 len 可以获取
  fmt.Println(len(scores))

  // 使用 range 遍历所有容器，比如 map、数组、slice
  for key, val := range scores {
  fmt.Printf("scores[%s] = %d\n", key, val)
  }

{% endhighlight %}

## 0x07 slice

{% highlight ruby %}
var a1 [5]int = [5]int{1, 2, 3, 4, 5}
// 定义切片，从 a1 中切取部分内容
// 切取内容：a1[start:end] ====> [start, end) 半开区间, start 省略表示从头开始，end 省略表示到结尾
// a1[:] 表示完全赋值过来，好处是切片可以扩容
s1 := a1[2:4]
fmt.Println("s1 is", s1)

//s1 的追加对 a1 产生了影响, 数组和切片共用同一块地址， 只需要插入一个元素 a1 能用的空间只剩最后一个位置
// s1 = append(s1, 100)

// s1 的追加对 a1 没有产生影响，插入两个元素，a1 没有足够的空间了，所以这时会新分配空间给 slice
s1 = append(s1, 100, 1000)

fmt.Println(a1)

fmt.Println(s1)

    /*
    切片里有 2 个值：
    1 cap 表示容量, 表示能容纳的上限
    2 len 元素个数，表示当前存储个数
    cap >= len

    slice 的扩容：
    当 slice 的元素个数的数量级较小（比如几千），扩容是成倍的扩展
    数量级比较大时就不会成倍扩展了，为了较少消耗

\*/

var s1 []int
fmt.Println("len=", len(s1), "cap=", cap(s1))

for i := 0; i < 20; i++ {
s1 = append(s1, i)
fmt.Println("len=", len(s1), "cap=", cap(s1))
}
{% endhighlight %}

## 0x08 结构体

- go 没有 class 通过自定义结构支持 class；go 里有 class 和 struct
- 格式： type [name] struct 结构定义新类型
- go 的结构体成员都是 public 的
  {% highlight ruby %}
  // 自定义人类结构
  type Person struct {
  name string
  age uint
  sex string
  fight uint // 战斗力
  }

func main() {

p1 := Person{
"tome",
30,
"man",
10,
}
fmt.Println("p1 is", p1)
fmt.Printf("p1 is %+v\n", p1)

p2 := Person{
name: "Maria",
sex: "female",
// 其他值用默认初始化值
}
fmt.Println("p2 is", p2)
fmt.Printf("p2 is %+v\n", p2)

// 通过匿名结构定义
p3 := struct {
name string
age uint
sex string
fight uint
}{"mike", 30, "man", 5}
fmt.Println("p3 is", p3)

// 当匿名定义结构和非匿名结构拥有的成员相同时，其对应的变量是可以相互赋值的
p2 = p3
fmt.Printf("p2=p3 is %+v\n", p2)

}

{% endhighlight %}

## 0x09 封装

- 类里的函数叫方法，方法是为对象服务的

{% highlight ruby %}
type Person struct {
name string
age uint
sex string
fight uint // 战斗力
}

// 封装（方法）
// 为 Person 提供方法
// 告诉编译器 getName 只能是 Person 对象调用
// (p Person) 叫函数接收器
func (p Person) getName() string {
return p.name
}

// 函数调用 参数都是值传递，会新 new 一个 Person 对象，所以 (this Person) 改不了 age 的值，
// Note 若要修改结构内容 需要使用指针 即 (this *Person) ，是引用传递
func (this *Person) setAge(age uint) {
this.age = age
}

func main() {

fmt.Println("encapsulate")

p1 := Person{
"tom",
30,
"man",
10,
}
fmt.Println("p1 's name is", p1.getName())

p1.setAge(100)
fmt.Println("p1 's age is", p1.age)

}

{% endhighlight %}

## 0x0A 内嵌

- 类似继承，但其实不是，详情见 Go 语言设计那本书
  {% highlight ruby %}
  // 自定义人类结构
  type Person struct {
  name string
  age uint
  sex string
  fight uint // 战斗力
  }

func (this \*Person) setAge(age uint) {
this.age = age
}

func (p Person) getName() string {
return p.name
}

type SuperMan struct {
Person // 内嵌，继承 Person 的全部属性

chaonengli string
name string
}

type SuperMan2 struct {
p Person // 非内嵌，仅仅是属性定义

chaonengli string
name string
}

func main() {
s1 := SuperMan{
Person{
"clark",
30,
"man",
100000,
},
"big li",
"tom",
}

fmt.Println("s1 is", s1)
s1.setAge(20)
fmt.Println("s1 is", s1)

// 继承时候如果取的成员变量自己有就取自己的，自己没有就去父类里找
fmt.Println("name is", s1.name, "name", s1.getName())

s2 := SuperMan2{
Person{
"clark2",
40,
"man",
100000,
},
"big li",
"mike",
}
fmt.Println("name is", s2.name)

}
{% endhighlight %}

## 0x0B 多态

- 多态：同一个调用，不同的状态，使用同一个对象调用同一个方法，呈现不同的状态，实际是调用了不同对象的同名方法
- 前提：（C++，通过虚函数的方式，系统内部定义虚表 即地址空间入口）

1. 基类对象可以指向派生类对象
2. 基类对象在执行时，可以找到派生类的方法所在的地址空间

- 接口：

1. 定义方法，不实现
2. 结构体实现接口定义的方法，当一个结构体实现了某个接口内全部方法，我们说该接口就被结构体所支持，
   此时接口对象可以被结构体对象赋值（结构体对象可以作为接口对象的右值），
   此时 接口对象只能识别接口内的方法，由于其并未实现该方法，所以在调用接口内方法时，实际上要执行结构体实现的方法

- Go 语言通过接口支持多态

{% highlight ruby %}
// 定义一个接口

type Animal interface {
sleeping()
eating()
}

type Cat struct {
color string
}

func (self Cat) sleeping() {
fmt.Printf("%s's cat is sleeping\n", self.color)
}

func (self Cat) eating() {
fmt.Printf("%s's cat is eating\n", self.color)
}

type Dog struct {
color string
}

func (self Dog) sleeping() {
fmt.Printf("%s's dog is sleeping\n", self.color)
}

func (self Dog) eating() {
fmt.Printf("%s's dog is eating\n", self.color)
}

func doSomething(a Animal) {
a.eating()
a.sleeping()
}

func main() {

cat := Cat{"white"}
dog := Dog{"red"}

cat.eating()
dog.sleeping()

// 多态
var a1 Animal
a1 = cat
a1.eating()

a1 = dog
a1.sleeping()

doSomething(a1)
}
{% endhighlight %}

{% highlight ruby %}

// 自定义人类结构

type Human interface {
getName() string // 注意返回值
setAge(age uint)
}

// 接口的继承
type People interface {
Human
hahaha()
}

type Person struct {
name string
age uint
sex string
fight uint // 战斗力
}

func (this \*Person) setAge(age uint) {
this.age = age
}

func (p Person) getName() string {
return p.name
}

func (this Person) hahaha() {
fmt.Printf("%+v is hahaha\n", this)
}

type SuperMan struct {
Person // 内嵌，继承 Person 的全部属性

chaonengli string
name string
}

func doSomething(h People) {
h.getName()
h.setAge(25)
h.hahaha()
fmt.Printf("%+v\n", h)
}

func main() {

p1 := Person{
"tom",
30,
"male",
5,
}

// 由于 setAge 传入了指针参数，所以这里需要用取地址 &
doSomething(&p1)

s1 := SuperMan{
Person{
"tom",
30,
"male",
5,
},
"hide",
"litter",
}
doSomething(&s1)

}
{% endhighlight %}
