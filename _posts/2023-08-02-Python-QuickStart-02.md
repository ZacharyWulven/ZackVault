---
layout: post
title: Python-QuickStart-02
date: 2023-08-02 16:45:30.000000000 +09:00
categories: [Python, QuickStart]
tags: [Python, QuickStart]
---


# 14 集合 Set
* 集合属于可变类型序列
* 集合中的元素不能重复

## 14.1 创建

### 方式 1： {}

```python
s = {2, 3, 4, 5, 5, 6, 7, 7}
print(s)
print(type(s)) # <class 'set'>
```

### 方式 2：内置函数 set()

```python
s = set(range(6))
print(s) # {0, 1, 2, 3, 4, 5}

# list -> set
s = set([1, 2, 3])
print(s) # {1, 2, 3}

s = set((22, False, 'hello'))
print(s) # {False, 22, 'hello'}

s = set('Python')
print(s) # {'n', 't', 'P', 'o', 'h', 'y'}

s = set({2, 3, 4, 5, 5, 6, 7, 7})
print(s) # {2, 3, 4, 5, 6, 7}

# 空 set
s = set()
print(s) # set()
```

## 14.2 判断
* in 判断是否在集合中
* not in 判断不在集合中

```python
s = {10, 20, 30, 40}

print(10 in s)     # True
print(11 not in s) # True
```

## 14.3 新增元素

* add() 方法，一次增加一个元素

```python
s.add(90)
print(s) # {40, 10, 20, 90, 30}
```

* update() 方法，至少添加一个元素

```python
s.update({11, 22})
print(s) # 传入 set {40, 10, 11, 20, 22, 90, 30}
s.update([33, 43])
print(s) # 传入 list {33, 40, 10, 11, 43, 20, 22, 90, 30}

s.update((78, 66))
print(s) # 传入 tuple {33, 66, 40, 10, 11, 43, 78, 20, 22, 90, 30}
```

## 14.4 删除操作

```python
# remove() 方法，删除指定元素，如果元素不存在抛出 KeyError
s.remove(66)
print(s) # {33, 40, 10, 11, 43, 78, 20, 22, 90, 30}

# discard() 方法，删除指定元素，如果元素不存在，不会抛出异常
s.discard(33)
s.discard(44) # 不会抛出异常
print(s) # {40, 10, 11, 43, 78, 20, 22, 90, 30}

# pop() 方法，随机删除一个元素，不能指定参数
s.pop()
print(s)

# clear() 删除所有元素
s.clear()
print(s)
```

## 14.5 集合之间的关系

```python
# 两个集合是否相等 使用 == 或 !=，
# 有相同的元素就是相等
s1 = {1, 2, 4, 3}
s2 = {2, 1, 3, 4}
print(s1 == s2) # True
print(s1 != s2) # False

# 一个集合是否是另一个的子集，调用 issubset 判断
s1 = {1, 2, 4, 3}
s2 = {2, 1}
s3 = {2, 1, 5}
print(s2.issubset(s1)) # True
print(s3.issubset(s1)) # False

# 一个集合是否是另一个的超集，调用 issuperset
print(s1.issuperset(s2)) # True
print(s1.issuperset(s3)) # False

# 两个集合 是否没有交集
print(s2.isdisjoint(s3)) # False 有交集
s4 = {6, 7}
print(s4.isdisjoint(s2)) # True 没有交集
```


## 14.6 数据操作

```python
s1 = {1, 2, 4, 3}
s2 = {2, 3, 4, 5, 6}
print('--------数据操作---------------------')
# 6.1 取交集
print(s1.intersection(s2)) # {2, 3, 4}
print(s1 & s2)             # 或者使用 & 也是取交集  {2, 3, 4}

# 6.2 取并集
print(s1.union(s2)) # {1, 2, 3, 4, 5, 6}
print(s1 | s2)      # 或者使用 | 也是取并集  {1, 2, 3, 4, 5, 6}

# 6.3 取差集 就是 A - B 集合
print('--------取差集---------------------')
print(s1.difference(s2)) # {1}
print(s1 - s2)           # 或者使用 - 取差集 {1}

# 6.4 取对称差集
print(s1.symmetric_difference(s2)) # {1, 5, 6}
print(s1 ^ s2)                     # 或使用 ^ 取对称差集 {1, 5, 6}
```

## 14.7 set 生成式
* 类似列表的生成式，就是把 [] 替换为 {}

```python
s = {x * 2 for x in range(3)}
print(s)
```


# 15 字符串
* 字符串是不可变序列
* 字符串有驻留机制：即只保存一份相同并且不可变的拷贝，不同的值被存放在驻流池中。相同的值在驻留池中只有一个备份，后续创建相同内容的字符串，不会开辟新空间，而是直接把其赋值给新变量

```python
a = 'Python'
b = "Python"
c = """Python"""
print(a, id(a)) # Python 140523277126896
print(b, id(b)) # Python 140523277126896
print(c, id(c)) # Python 140523277126896
```


## 15.1 驻留机制的几种情况(交互模式)
* 使用交互模式进行演示

> `>>>` 为交互模式；不要使用 Pycharm 因为它对字符串有优化
{: .prompt-info }


### Case 1：字符串的长度是 0 或 1 时

### Case 2：符合标识符的字符串
* abc% 不驻留
* abc 驻留

```python
>>> s1 = 'abc%'
>>> s2 = 'abc%'
>>> s1 is s2
False
>>>

>>> a = 'abc'
>>> b = 'abc'
>>> a is b
True
>>>
```


### Case 3：只在编译时驻留，而非运行时

```python
a = 'abc'
b = 'ab' + 'c'           # 在编译就完成了，所以有驻留
c = "".join(['ab', 'c']) # 运行时，没有驻留
print(a, id(a))          # abc 140602733975088
print(b, id(b))          # abc 140602733975088
print(c, id(c))          # abc 140601928621616
```

### Case 4：[-5, 256] 之间的整数数字


```python
>>> a = '-6'
>>> b = '-6'
>>> a is b
False
>>>
```

### 可以强制驻留

```python
>>> a = '-6'
>>> b = '-6'
>>> a is b
False
>>> a = sys.intern(b)
>>> a is b
True
>>>
```


> 驻留机制的好处：Python 中一切皆对象，这样可以提升效率，避免频繁的创建和销毁对象。拼接字符串、修改字符串是比较消耗性能的。拼接字符串时建设使用 `str` 的 `join()` 方法，而不是 `+`。因为 `join` 方法会先计算所有字符串的长度，然后再拷贝，只 new 一次对象，效率比 `+` 高
{: .prompt-info }


## 15.2 字符串查询操作
* `index()` 查找子串第一次出现的起始位置，如果查找子串不存在，会抛出 ValueError
* `rindex()` 查找子串最后一次出现的起始位置，如果查找子串不存在，会抛出 ValueError
* `find()` 查找子串第一次出现的起始位置，如果查找子串不存在，则返回 -1
* `rfind()` 查找子串最后一次出现的起始位置，如果查找子串不存在, 则返回 -1

```python
s = 'hello,hello!'
print(s.index('lo'))  # 3
print(s.find('lo'))   # 3

print(s.rindex('lo')) # 9
print(s.rfind('lo'))  # 9
```


## 15.3 字符串大小写转换操作

```python
s = 'hello,python'
# upper() 全部转为大写
print(s.upper())

# lower() 全部转为大写
print(s.lower())

# swapcase() 把原来大小转为小写，把小写转为大写
s = 'Hello,wORLD'
print(s.swapcase())

# capitalize() 把第一个字符转为大写，把其余的转为小写
print(s.capitalize())
# title() 把每个单词首字母转为大写，其他字符转为小写
print(s.title())
```

## 15.4 字符串内容对齐操作

```python
## center() 居中对齐
### 第一个参数指定宽度
### 第二个参数指定填充符，参数可选，默认是空格
### 如果设置的宽度小于实际宽度，则返回原字符串
s = 'hello, Python'
print(s.center(20, '*')) # ***hello, Python****

## ljust() 左对齐
### 第一个参数指定宽度
### 第二个参数指定填充符，参数可选，默认是空格
### 如果设置的宽度小于实际宽度，则返回原字符串
print(s.ljust(20, '*'))  # hello, Python*******

## rjust() 右对齐
### 第一个参数指定宽度
### 第二个参数指定填充符，参数可选，默认是空格
### 如果设置的宽度小于实际宽度，则返回原字符串
print(s.rjust(20, '*'))  # *******hello, Python

## zfill() 右对齐，左侧用 0 填充，
# 只有一个参数就是宽度，如果设置的宽度小于实际宽度，则返回原字符串
print(s.zfill(20))                 # 0000000hello, Python
print('-8910'.zfill(10))           # -000008910
```


## 15.5 字符串拆分

```python
## split() 从字符串左边开始分割，默认按'空格'分割字符串，返回一个 list
## 参数 sep 指定分割符
## 参数 maxsplit 指定分割次数，经过最大分割次数后，剩余的字符串作为单独一个部分
s = 'hello world python'
lst = s.split()
print(lst) # ['hello', 'world', 'python']
s = 'hello, world,python'
lst = s.split(sep=',')
print(lst) # ['hello', ' world', 'python']
lst = s.split(sep=',', maxsplit=1)
print(lst)  # ['hello', ' world,python']

print('----------rsplit---------------------')

## rsplit() 与 split 相反，从字符串右边开始分割，其他功能一样
s = 'hello world python'
lst = s.rsplit()
print(lst)  # ['hello', 'world', 'python']
s = 'hello-world-python'
lst = s.rsplit(sep='-')
print(lst)  # ['hello', 'world', 'python']
lst = s.rsplit(sep='-', maxsplit=1)
print(lst)  # ['hello-world', 'python']
```

## 15.6 判断字符串方法

```python
s = 'hello, python'
# isidentifier() 判断字符串是否是合法的标识符
print(s.isidentifier())        # False
print('hello'.isidentifier())  # True

# isspace() 判断字符串是否全部由空白字符组成（回车、tab，换行等）
print(s.isspace())       # False
print('    '.isspace())  # True
print('\t'.isspace())    # True

# isalpha() 判断字符串是否全部由字母组成
print('abc'.isalpha())   # True
print('哈哈'.isalpha())   # True
print('哈哈1'.isalpha())   # False

# isdecimal() 判断字符串是否全部由 10 进制数字组成
print('123'.isdecimal())  # True
print('123a'.isdecimal()) # False
print('1.2'.isdecimal())  # False

# isnumeric() 判断字符串是否全部由数字组成
print('13'.isnumeric())  # True
print('13四'.isnumeric()) # True
print('Ⅵ'.isnumeric()) # True

# isalnum() 是否全部由字母和数字组成
print('abc1'.isalnum())   # True
print('哈哈123'.isalnum()) # True
print('abc$'.isalnum()) # False
```


## 15.7 其他方法

```python
## replace() 方法
# 参数 1 指定被替换的子串
# 参数 2 指定替换的的子串
# 参数 3 指定最大替换次数
# 该方法返回替换后的字符串，原字符串不变
s = 'hello,Python'
print(s.replace('Python', 'Rust')) # hello,Rust
# 只替换 2 次
s = 'hello,Python,Python,Python'
print(s.replace('Python', 'Ruby', 2)) # hello,Ruby,Ruby,Python


## join() 方法
# 将列表或元组中的字符串合并成一个字符串
lst = ['hello', 'Rust', 'Python']
print('|'.join(lst)) # hello|Rust|Python
print(''.join(lst))  # helloRustPython
t = ('hello', 'Rust', 'Python')
print("".join(t))    # helloRustPython
print('-'.join('Python')) # P-y-t-h-o-n
```

## 15.8 字符串比较操作

```python
# 比较规则：
# 首先比较两个字符串的第一个字符，如果相等则继续依次比较下一个字符，直到遇到不相等的字符，
# 其结果作为字符串的比较结果，后续的字符不再比较了

# 比较原理：
# 两个字符比较时，比较的是其原始值（ordinal value），调用内置函数 ord 可以得到指定字符的原始值
# 与 ord 对应的内置函数是 chr，调用 chr 时指定原始值返回对应的字符
print('apple' > 'app')    # True
print('ape' > 'bye')      # False, 97 小于 98 所以结果为 False
print(ord('a'), ord('b')) # 97 98
print(chr(97), chr(98))   # a b
print(ord('黑'))          # 40657
print(chr(40657))         # 黑

## == 与 is 区别
# == 比较的是 value 是否相等
# is 比较的是 id(内存地址) 是否相等
print('== 与 is 区别')
a = b = 'Python'
print(a == b)  # True
print(a is b)  # True
c = ''.join(['P','ython'])
print(a == c)  # True
print(a is c)  # False
print(id(a), id(c))  # 140533209238768 140533479055024
```


## 15.9 字符串切片

```python
print('字符串切片')
s = 'hello,Python'
s1 = s[:5]
s2 = s[6:]
print(s1)  # hello
print(s2)  # Python
print(s[::2]) # hloPto  步长为 2
print(s[::-1]) # nohtyP,olleh
print(s[-6:])  # Python
```

> 字符串是不可变对象，切片将产生新对象
{: .prompt-info }


## 15.10 格式化字符串

```python
# 方式 1：%
# %s：表示字符串
# %i、%d：表示整数
# %f：表示浮点数

name = 'tom'
age = 20
print('My name is %s, %d years old' % (name, age))

# 方式 2：{}
# 所有 {0} 用 name 替换
print('My name is {0}, {1} years old, nickname is {0}'.format(name, age))
# My name is tom, 20 years old, nickname is tom

# 方式 3：f-string
print(f'My name is {name}, {age} years old, nickname is {name}')
# My name is tom, 20 years old, nickname is tom


## 详细
print('%10d' % 99) # 总共占 10 个字符位置
print('%.3f' % 3.1415926)  # 保留 3 位小数
print('%10.2f' % 3.1415926)  # 保留 2 位小数 同时宽度为 10

print('{0:10.3}'.format(3.1415926)) # .3 表示一共 3 位数
print('{0:10.3f}'.format(3.1415926)) # .3f 表示 3 位小数，0 表示占位符的索引
```


## 15.11 字符串编码 解码

```python
s = '多蓝古雷格'

# 编码
byte = s.encode(encoding='utf-8')
# b 表示二进制
# b'\xe5\xa4\x9a\xe8\x93\x9d\xe5\x8f\xa4\xe9\x9b\xb7\xe6\xa0\xbc'
print(byte) # utf-8 一个中文占 3 个字节

# 解码
# t = byte.decode(encoding='BGK') # error 编解码方式必须一样
t = byte.decode(encoding='utf-8')
print(t) # 多蓝古雷格
```


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
