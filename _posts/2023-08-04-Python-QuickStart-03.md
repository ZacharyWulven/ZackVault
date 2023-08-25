---
layout: post
title: Python-QuickStart-03
date: 2023-08-04 16:45:30.000000000 +09:00
categories: [Python]
tags: [Python]
---


# 16 函数

## 16.1 函数调用参数传递
### 两种传参方式
1. 根据形参对应位置进行实参传递，即按实际传递顺序给形参
2. 关键字传递，类似 Swift 的参数标签


> 传参时，如果传入对象是可变对象，那么函数内对其进行修改会影响其实际对象的值。而不可变对象不会受影响。
{: .prompt-info }


## 16.2 函数返回值
* 函数返回多个值时，类型为元组
* 不写 `return` 默认返回 `None`

```python
def make_tuple():
    return 1, 2

t = make_tuple()
print(t, t[1])   # (1, 2) 2

def fn1():
    print('fn1')

ret = fn1()
print(ret) # None
```

## 16.3 参数默认值

```python
def fn2(a,b=10): # b 默认值是 10
    print('a =', a, 'b =', b)
    

fn2(100)         # a = 100 b = 10
fn2(100, 40)     # a = 100 b = 40
```


## 16.4 个数可变的形参

### 个数可变的位置形参
* 函数定义时，可能无法确定传递位置实参的个数，使用可变位置参数
* 使用 `*` 定义个数可变的位置形参，结果为一个`元组`

```python
def fn3(*args):
    print(args[0])
    print(args)

fn3(10)           # (10,)
fn3(10, 20)       # (10, 20)
fn3(10, 20, 30)   # (10, 20, 30)
```

### 个数可变的关键字形参
* 函数定义时，可能无法确定传递关键字实参的个数，使用个数可变的关键字形参
* 使用 `**` 定义个数可变的关键字形参，结果为一个`字典`

```python
def fn4(**kwargs):
    print(kwargs)

fn4(name='tom', age='10') # {'name': 'tom', 'age': '10'}
```

### 一个函数可同时定义，个数可变的位置形参和个数可变的关键字形参

```python
def fn5(*args1, *args2):     # 报错
    pass

def fn5(**kwargs, **kwargs): # 报错
    pass

def fn6(**kwargs, *args):    # 报错
    pass

def fn7(*args, **kwargs):    # 个数可变的位置形参必须在前边
    pass
```

> 一个函数中，个数可变的位置形参、个数可变的关键字形参，只能定义一个；如果既有个数可变的位置形参，又有个数可变的关键字形参，则个数可变的位置形参必须在前边（例如 fn7）；
{: .prompt-info }


## 16.5 参数总结
* 传参时使用 `*` 将列表中每个元素都转为位置实参传入
* 传参时使用 `**` 将字典中的元素转为关键字实参传入
* 形参中从 `*` 后的参数，强制必须使用关键字实参传递

```python
def fun(a, b, c):
    print(f'a={a},b={b},c={c}')

lst = [10, 20, 30]
# fun(lst) # error 因为 lst 被当成一个参数
fun(*lst) # *lst 将列表中每个元素都转为位置实参传入

dic = {'a': 11, 'b': 22, 'c': 33}
fun(**dic) # **dic 将字典中的元素转为关键字实参传入

def fun1(a, b, *,c, d): # 从 * 后的参数，强制必须使用关键字传递
    print(f'a={a},b={b},c={c}, d={d}')

fun1(1, 2, c=3, d=4) # a=1,b=2,c=3, d=4

# 形参定义顺序
def fun2(a, b, *,c, d, **kwargs):     # OK
    pass

# def fun3(a, b, *,c, d, *args):      # 报错
#     pass

def fun3(a, b, *,c=10, d, **kwargs):  # OK
    pass
```

## 16.6 变量的作用域

### 全局变量
* 在函数外定义的变量，可作用于函数内外

### 局部变量
* 在函数内定义的变量，只在函数内有效
* 局部变量使用 `global` 声明就会变成全局变量

```python
def vr():
    global name
    name = 'Jack'
    print('vr', name)

vr()           # vr Jack
name = 'Tom'
print(name)    # Tom
vr()           # vr Jack
```


# 17 异常

## 17.1 多异常结构
* 捕获异常顺序按照先子类后父类的顺序
* 兜底最后可加 BaseException

## 17.2 try...except...

```python
def try_01():
    try:
        a = int(input('请输入第一个整数'))
        b = int(input('请输入第二个整数'))
        result = a / b
        print('result is', result)
    except ZeroDivisionError:       # 捕获除数为 0 异常
        print('第二个数不能为 0')
    except ValueError:
        print('是能输入数字串')
    except BaseException as e:
        print(f'e is {e}')

try_01()
```
## 17.2 try...except...else
* 如果 try 中没有异常则执行 else 块
* 如果 try 中抛出异常则执行 except 块

```python
def try_02():
    try:
        a = int(input('请输入第一个整数'))
        b = int(input('请输入第二个整数'))
        result = a / b
    except BaseException as e:
        print(f'e is {e}')
    else:
        print('result is', result)

# try_02()
```

## 17.3 try...except...else...finally
* finally 块无论是否发生异常都会被执行，常用于释放 try 块中的资源

```python
def try_03():
    try:
        a = int(input('请输入第一个整数'))
        b = int(input('请输入第二个整数'))
        result = a / b
    except BaseException as e:
        print(f'e is {e}')
    else:
        print('result is', result)
    finally:
        print('谢谢使用')
```


## 17.4 常见的异常类型

```python
# 1 ZeroDivisionError：除 0 或模 0 导致
a = 1 % 0 # ZeroDivisionError: integer division or modulo by zero

# 2 IndexError
lst = [1, 3, 4]
print(lst[5])   # IndexError: list index out of range

# 3 KeyError  字典中没有这个 key
dic = {'name': 'Jack'}
print(dic['age'])   # KeyError: 'age'

# 4 NameError 没有声明/初始化对象
print(name)  # NameError: name 'name' is not defined

# 5 SyntaxError 语法错误
int a = 10  # SyntaxError: invalid syntax

# 6 ValueError 传入无效的参数
a = int('hello') # ValueError: invalid literal for int() with base 10: 'hello'
```

## 17.5 traceback 模块
* 手动调用 traceback 打印

```python
import traceback
def try_trace():
    try:
        print('------------------------')
        print(1/0)
    except:
        traceback.print_exc()

try_trace()
```

# 18 类与对象

> Python 中一切皆对象
{: .prompt-info }


## 18.1 类定义与方法

* 类属性：在类中的方法外定义的属性，被该类的所有对象所共享
* 类方法：使用 `@classmethod` 修饰的方法，使用类名直接访问
* 静态方法：使用 `@staticmethod` 修饰的方法，使用类名直接访问

* 实例对象中有一个类指针，指向其类对象

![image](/assets/images/python/cls.png)


```python
class Student:
    native_place = '云上'  # 直接写在类里的变量，称为类属性

    # 初始化方法
    def __init__(self, name, age):
        self.name = name   # self.name 是实例属性
        self.age = age


    # 实例方法
    def eat(self):
        print('学生在吃饭')

    # 静态方法, 使用 staticmethod 进行修饰
    @staticmethod
    def st_method():  # 静态方法中不能写 self
        print('我是静态方法')

    # 类方法，使用 classmethod 进行修饰
    @classmethod
    def c_method(cls):
        print('我是类方法', cls)

print(id(Student))    # Student 类对象的内存地址 140296517225136
print(type(Student))  # <class 'type'>
print(Student)        # <class '__main__.Student'>


# 创建对象
stu = Student('张三', 20)
print(stu.name)

# 调用 eat 方法 方式一
stu.eat()

print('-------------------')
# 调用 eat 方法 方式二
Student.eat(stu)


# 类属性的使用
print(Student.native_place)  # 云上
stu1 = Student('张三', 20)
stu2 = Student('李四', 25)
print(stu1.native_place)  # 云上
print(stu2.native_place)  # 云上

# 类方法使用
print('---------类方法使用------------------')
Student.c_method()

# 静态方法使用
print('---------静态方法使用------------------')
Student.st_method()
```

## 18.2 动态绑定属性和方法

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def eat(self):
        print(f'{self.name} 正在吃饭')

p1 = Person('张三', 22)
p2 = Person('李四', 25)
p1.eat()

# 动态绑定属性
print('-------只为 p2 增加 gender 属性，而 p1 没有---------------------')
p2.gender = '女'
print(p2.gender) # 女
# print(p1.gender) # AttributeError: 'Person' object has no attribute 'gender'

# 动态绑定方法
print('-------只为 p2 动态绑定方法，而 p1 没有---------------------')
def show():
    print('我是 show 方法')

p2.show = show # 绑定方法到 p2 对象上
p2.show()  # 我是 show 方法
# p1.show()  # 因为 p1 没有绑定 show 方法，AttributeError: 'Person' object has no attribute 'show'
```

> 动态绑定属性和方法只对绑定的那个对象生效
{: .prompt-info }


## 18.3 封装：私有属性
* 声明前加两个 `_`

```python
class Human:

    def __init__(self, name, age):
        self.name = name
        self.__age = age # __age 为私有属性，不希望外部可以访问

    def get_age(self):
        return self.__age

h = Human('Tom', 30)
print(h.get_age())

print('---------查看对象中都用哪些属性-------------------------')
print(dir(h))        # ['_Human__age', '__class__' ....
print(h._Human__age) # 通过 dir 获取的属性名，绕过进行使用私有属性
```

> 通过 dir 获取的属性名，可以达到绕过进而访问到私有属性。所以看到 `__` 开头的属性就不要访问了，全屏自觉。🤣
{: .prompt-info }


## 18.4 继承
* 如果一个类定义时没有继承任何类，则默认继承 `object` 
* Python 支持多继承
* 定义子类时，必须在其构造函数中调用父类的构造函数




