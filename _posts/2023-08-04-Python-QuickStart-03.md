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
