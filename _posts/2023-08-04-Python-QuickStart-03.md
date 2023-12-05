---
layout: post
title: Python-QuickStart-03
date: 2023-08-04 16:45:30.000000000 +09:00
categories: [Python, QuickStart]
tags: [Python, QuickStart]
---


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

### 单继承与方法 Override

```python
class Person(object): # 继承自 object，不写 object 也行
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def info(self):
        print(self.name, self.age)


class Student(Person):

    def __init__(self, name, age, score):
        super().__init__(name, age)     # 调用 super
        self.score = score
        
    def info(self):     # override 父类方法
        super().info()  # 调用 super
        print(f'分数是 {self.score}')



s = Student('Jack', 23, 60)
s.info()
```

### 多继承

```python
class A:
    pass


class B:
    pass


class C(A, B):
    pass
```


## 18.5 object 类的 __str__() 方法
* `object` 类是所有类的父类，如果一个类没有明确写继承自哪个类，那么它默认继承 `object` 类
* 可以使用内置函数 `dir()` 查看指定对象的所有属性
* `object` 有一个 `__str__()` 方法用于返回一个对象的描述（类似 swift 的 description 方法），对应与内置函数 `str()` 方法


```python
class Human:

    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __str__(self): # 类似 swift 的 description 方法
        return f'name is {self.name}, age is {self.age}'


h = Human('Tom', 22)
print(h) # 这里会默认调用对象的 __str__() 方法
```


## 18.6 多态
 
```python
class Animal:

    def eat(self):
        print('动物会吃')

class Dog(Animal):

    def eat(self):
        print('狗吃骨头')


class Cat(Animal):

    def eat(self):
        print('猫吃鱼')
        
        
class Man:

    def eat(self):
        print('人吃五谷杂粮')



def eat(obj):
    obj.eat()


eat(Dog())    # 狗吃骨头
eat(Cat())    # 猫吃鱼
eat(Animal()) # 动物会吃
eat(Man())    # 人吃五谷杂粮
```

## 18.7 特殊属性

* 即以 `__`    开头和结束的方法
* `__dict__`  获得类对象或实例对象所绑定的所有属性和方法的字典
* `__class__` 输出这个对象的类型
* `__bases__` 输出这个类的父类的元组
* `__base__`  输出第一个父类
* `__mro__`   输出类的层级结构
* `__subclasses__()` 输出子类列表
 

```python
class A:
    pass

class B:
    pass

class C(A, B):
    def __init__(self, name, age):
        self.name = name
        self.age = age


print('--------特殊属性----------------')
c = C('Jack', 20)
print(c.__dict__)  # 实例 c 的属性 {'name': 'Jack', 'age': 20} 字典
print(C.__dict__)  # 类 C 的属性 {'__module__': '__main__', '__init__': <function C.__init__ at 0x7fcd0815e5e0>, '__doc__': None}
print(c.__class__) # 输出这个对象的类型 <class '__main__.C'>
print(C.__bases__) # 输出 C 类的父类的元组，(<class '__main__.A'>, <class '__main__.B'>)
print(C.__base__)  # 输出第一个父类 <class '__main__.A'>
print(C.__mro__)   # 输出类的层级结构 (<class '__main__.C'>, <class '__main__.A'>, <class '__main__.B'>, <class 'object'>)
print(A.__subclasses__()) # 输出 A 的子类列表， [<class '__main__.C'>] 
```

## 18.8 特殊方法
* 通过重写 __add__ 使得对象有 + 的功能
* 通过重写 __len__ 输出对象的长度，内置函数 len() 会调用参数的 __len__ 方法


```python
class Spider:

    def __init__(self, name):
        self.name = name

    # 通过重写 __add__ 使得对象有 + 的功能
    def __add__(self, other):
        return self.name + other.name

    # 通过重写 __len__ 输出对象的长度，内置函数 len() 会调用参数的 __len__ 方法
    def __len__(self):
        return len(self.name)


s1 = Spider('Jack')
s2 = Spider('Tom')
s = s1 + s2
print(s)   # JackTom
s = s1.__add__(s2)
print(s)   # JackTom

print(len(s1))  # 会调用 s1 的 __len__ 方法
print(s1.__len__())
```

## 18.9 __new__() 和 __init__()
* __new__() 用于创建对象
* __init__() 对创建的对象进行初始化


```python
class Person:
    def __new__(cls, *args, **kwargs):
        print(f'new is called, cls id = {id(cls)}')
        obj = super().__new__(cls)
        print(f'create obj id = {id(obj)}')
        return obj

    def __init__(self, name, age):
        print(f'init is called, self id = {id(self)}')
        self.name = name
        self.age = age


print(f'object 类对象 id = {id(object)}')
print(f'Person 类对象 id = {id(Person)}')

p = Person('Mike', 23)
print(f'p 的 id = {id(p)}')
```


> 先调用 `__new__()` 再调用 `__init__()`
{: .prompt-info }


## 18.10 浅拷贝与深拷贝
* Python 一般都是浅拷贝，拷贝时，对象包含的子对象不拷贝，原对象与拷贝对象引用同一个对象
* 使用 copy 模块的 `deepcopy` 函数，递归拷贝对象中的子对象，原对象与拷贝对象的子对象引用不同

```python
class CPU:
    pass

class Disk:
    pass

class Computer:

    def __init__(self, cpu, disk):
        self.cpu = cpu
        self.disk = disk

cup1 = CPU()
disk1 = Disk()
c1 = Computer(cup1, disk1)


import copy
c2 = copy.copy(c1)
print(c1, c1.cpu, c1.disk) # <__main__.Computer object at 0x7f7f780bcbe0> <__main__.CPU object at 0x7f7f780bcc40> <__main__.Disk object at 0x7f7f780bcc10>
print(c2, c2.cpu, c2.disk) # <__main__.Computer object at 0x7f7f780bcb20> <__main__.CPU object at 0x7f7f780bcc40> <__main__.Disk object at 0x7f7f780bcc10>

print('-------深拷贝-----------------')
c3 = copy.deepcopy(c1)
print(c1, c1.cpu, c1.disk) # <__main__.Computer object at 0x7fd7f008cbe0> <__main__.CPU object at 0x7fd7f008cc40> <__main__.Disk object at 0x7fd7f008cc10>
print(c3, c3.cpu, c3.disk) # <__main__.Computer object at 0x7fd7f008c9a0> <__main__.CPU object at 0x7fd7f008c5e0> <__main__.Disk object at 0x7fd7e00a7bb0>
```

> c2 为浅拷贝出的对象，c1 与 c2 的 id 不同，但子对象的 id 相同。c3 为深拷贝出的对象，本身以及其持有的对象都全部新 new 出来的
{: .prompt-info }


# 19 模块
* 一个 .py 文件就是一个模块

## 模块中可以包含
1. 类
2. 函数
3. 语句

## 19.1 导入模块
* import 模块名称 [as 别名]
* from 模块名称 import 函数/变量/类


```python
import math
print(id(math))    # 140227327370960
print(type(math))  # <class 'module'>
print(math)        # <module 'math' from '/Library/Frameworks/Python.framework/Versions/3.9/lib/python3.9/lib-dynload/math.cpython-39-darwin.so'>
print(math.pi)     # 3.141592653589793
print(dir(math))


# 自定义模块报错，可尝试模块所在目录右键->Make Directory as Sources Root 解决
import calc               # 导入模块
# from calc import add

print(calc.add(2, 3))
print(calc.div(10, 4))
```


## 19.2 以主程序方式运行

* 每个模块定义中都包含一个记录模块的变量 __name__，程序可以检测该变量，以确定它们在哪个模块运行
* 如果一个模块不被导入其他程序中运行，那么它可能在解释器顶级模块中运行
* 顶级模块 __name__ 变量值为 __main__

```python

def add(a, b):
    return a + b

def div(a, b):
    return a / b


if __name__ == '__main__':  # 只有点击运行 calc 时，这里才执行
    print('run calc----------')
    print(add(10, 15))
```


## 19.3 Python 中的包
* 包是一个分层次的目录结构，它将一组功能相近的模块组织在一起
* 创建包：PyCharm 右键->New->Python Package


### 包的作用
1. 代码规范
2. 避免命名冲突

### 包与目录的区别：
1. 包中含有 `__init__.py` 文件
2. 目录中通常不包含 `__init__.py` 文件

print('--------Python 中的包---------------------')

## 导入包

```python
# import 包名称.模块名称（只能导入包名或模块名称）
# import package1.moduleA
# print(package1.moduleA.a)


# 别名 import
# mb 是 package1.moduleB 模块的别名
import package1.moduleB as mb 
print(mb.a)


# from ... import ... 可以导入包、模块、函数、类型、变量等
from package1 import moduleA
print(moduleA.a)
```


## 19.4 常用的内置模块
* sys 与 Python 解释器及其环境相关的标准库
* time 提供与时间相关的各种函数的标准库
* os 提供访问操作系统服务功能的标准库
* calendar 提供日期相关的各种函数的标准库
* urllib 用于读取来自服务器的数据标准库
* json 用于 JSON 序列化和反序列化
* re 用于字符串中执行正则表达式匹配和替换
* math 提供标准算术运算函数的库
* decimal 用于进行精确控制运算精度、有效位数和四舍五入操作十进制运算
* logging 提供了灵活记录事件、错误、警告、调试信息等日志功能


```python
import sys

# getsizeof 获取对象占的字节数
print(sys.getsizeof(24))    # 28
print(sys.getsizeof(60))    # 28
print(sys.getsizeof(True))  # 28
print(sys.getsizeof(False)) # 24

import time
print(time.time()) #
print(time.localtime(time.time())) # time.struct_time(tm_year=2023, tm_mon=8, tm_mday=30, tm_hour=18, tm_min=50, tm_sec=17, tm_wday=2, tm_yday=242, tm_isdst=0)

import math
print(math.pi)
```

## 19.5 第三方模块安装以及使用
* 安装命令：$ pip install 第三方模块名称
* 导入：import 模块名称

### 安装 schedule
* pip install schedule

```python
import schedule

def schedule_job():
    print('schedule_job')

schedule.every(3).seconds.do(schedule_job)
while True:
    schedule.run_pending()
    time.sleep(1)  # 每 3 秒休眠一秒
```


# 20 文件操作

## 20.1 编码格式

* Python 文件默认编码格式是 `UTF-8`

![image](/assets/images/python/decode.png)


## 20.2 读取文件内容

```python
f = open('a.txt', 'r')
print(f.readlines())  # ['Hello world！\n', 'Hi!']
f.close()
```


## 20.3 常用的文件打开模式

* r 以只读模式打开文件，文件指针放在文件开头，读取文件内容

```python
f = open('a.txt', 'r')
print(f.readlines())  # ['Hello world！\n', 'Hi!']
f.close()
```

* w 以只写模式打开文件，如果文件不存在会创建文件，如果文件存在则覆盖原内容，文件指针放在文件开头

```python
f = open('b.txt', 'w')
f.write('opp')
f.close()
```

* a 以追加模式打开文件，如果文件不存在会创建文件，文件指针放在文件开头。如果文件存在，则文件指针放在文件末尾

```python
f = open('b.txt', 'a')
f.write('python')
f.close()
```

* b 以二进制方式打开文件，不能单独使用，需要与其他模式一起使用，例如rb 或 wb

```python
# 拷贝图片操作
source = open('yin.jpeg', 'rb')

target = open('cp_yin.jpeg', 'wb')
target.write(source.read())
source.close()
target.close()
```

* + 以读写模式开发文件，不能单独使用，需要与其他模式一起使用，例如 a+

```python
f = open('c.txt', 'a+')
f.write('python')
f.seek(0)
print(f.readlines())
f.close()
```


## 20.4 文件常用方法

```python
f = open('a.txt', 'r')


# 1 read([size]): 从文件中读取 size 个字节或字符内容，若省略 size 则一直读到末尾
print(f.read())
f.close()

# 2 readline(): 读取一行
f = open('a.txt', 'r')
print('readline:', f.readline())

# 3 readlines() 把每一行作为一个对象，放到一个 list 中返回
print('readlines', f.readlines())
f.close()

f = open('a.txt', 'a')
# 4 write() 将字符串写入文件
f.write('Test')

# 5 writelines() 将字符串列表写入文件，不添加换行符
lst = ['Go', 'Rust', 'Python']
f.writelines(lst)
f.close()

# 6 seek(offset[,whence]) 把文件指针移动到新位置
# offset 表示相对于 whence 的位置。offset 为正往结束方向移动，为负往开始方向移动
# whence 0: 为默认值，从文件头开始计算
# whence 1: 从当前位置开始计算
# whence 2: 从文件末尾开始计算

f = open('a.txt', 'r')
f.seek(2) # seek 到前 2 个字节
print(f.readlines())

# 7 tell() 返回文件指针的当前位置字节数
print('tell is', f.tell())

# 8 flush() 把缓冲区的内容写入文件，但不关闭文件
# 9 close() 把缓冲区的内容写入文件，同时关闭文件，释放文件对象相关资源

f = open('d.txt', 'a')
f.write('hello')
f.flush()
f.write('world')
f.close()    # d.txt 内容是 helloworld
```


## 20.5 with 使用
* with 不用手动关闭，离开 with 作用域会自动释放资源，不管是否有异常
* 格式：with [上下文表达式] as [src_file]:
1. [上下文表达式] 的结果是一个上下文管理器
2. 什么是上下文管理器？即一个类实现了 `__enter__()` 和 `__exit__()` 方法，称为这个类遵守了上下文管理器协议


```python
# 自定义上下文管理器类
class CustomManager:

    def __enter__(self):
        print('call 上下文管理器 __enter__ ')
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print('call 上下文管理器 __exit__ ')

    def show(self):
        print('call show')


with CustomManager() as fp: # 相当于 fp=CustomManager()
    fp.show()


# 这里 open('d.txt', 'r') 是上下文管理器表达式
with open('d.txt', 'r') as fp:
    print(fp.readlines())

# 使用 with 实现图片复制
with open('yin.jpeg', 'rb') as src:
    with open('with_yin.jpeg', 'wb') as target:
        target.write(src.read())
```


# 21 os 常用函数
* os 模块是 Python 内置的与操作系统功能和文件系统功能相关的模块
* os 模块中语句的执行结果通常与操作系统有关，在不同操作系统运行，得到的结果可能不一样
* os 与 os.path 用于对目录或文件进行操作

```python
import os

# 1 执行系统命令
# os.system('ls')

# 2 打开应用程序
# Windows 系统
# os.startfile('~/Applications/GoLand.app')

# Mac 系统
import subprocess
# subprocess.call(['open', '/Applications/Xcode.app'])


# os 操作目录相关函数

# 3 getcwd() 返回当前目录
print(os.getcwd())

# 4 listdir(path) 返回指定路径下文件和目录信息
print(os.listdir('./'))

# 5 mkdir(path[,mode]) 创建目录, 如果目录已存在会报错
# os.mkdir('./test2') # 在当前目录创建 test2 目录

# 6 makedirs(path1/path2...[,mode]) 创建多级目录, 如果目录已存在会报错
# os.makedirs('./A/B/C')

# 7 rmdir(path) 删除目录，如果目录不存在会报错
# os.rmdir('./test2')

# 8 removedirs(path1/path2...) 删除多级目录，如果目录不存在会报错
# os.removedirs('./A/B/C')

# 9 chdir() 更改当前工作目录
os.chdir('./package1')
print(os.getcwd())
```


## os.path 模块操作目录相关函数


```python
import os.path

# 1 abspath(path) 获取文件或目录的绝对路径
print(os.path.abspath('./')) # 当前目录的绝对路径

# 2 exists(path) 判断文件或目录是否存在，存在返回 True，不存在返回 False
print(os.path.exists('d.txt'), os.path.exists('13_files.py'))

# 3 join(path, name) 将目录与目录或文件名拼接起来, 类似 OC 的 string appendComponentPath
print(os.path.join('./package1', 'calc.py'))  # ./package1/calc.py

# 4 split 分类文件名与文件名之前的 path
print(os.path.split('./package1/calc.py'))   # ('./package1', 'calc.py')

# 5 splitext 分类文件名称与扩展名
print(os.path.splitext('calc.py')) # ('calc', '.py')

# 6 basename() 从目录中提取文件名
print(os.path.basename('./calc.py'))  # calc.py

# 7 dirname() 从一个路径中提取文件路径，不包括文件名
print(os.path.dirname('./calc.py'))   # .

# 8 isdir() 判断是否为一个路径
print(os.path.isdir('./package1'))  # True
```

## 获取当前目录的所有 Python 文件

```python
import os

print('------- 获取当前目录的所有 Python 文件------------------')
path = os.getcwd()
lst = os.listdir(path)

for file in lst:
    if file.endswith('.py'):
        print(file)
```


## walk 遍历指定目录下所有文件和目录

```python
import os

path = os.getcwd()
lst = os.listdir(path)

print('-------获取当前目录的文件和子目录-------------------------')
lst_file = os.walk(path)
print(lst_file)
for dirpath, dirname, filename in lst_file:
    print('--------------------------------')
    print('dirpath', dirpath)
    print('dirname', dirname)
    print('filename', filename)
    
    print('for dir in dirname begin')
    for dir in dirname:
        print(os.path.join(dirpath, dir))
    print('for dir in dirname end')

    print('for file in filename begin')
    for file in filename:
        print(os.path.join(dirpath, file))
    print('for file in filename end')
```

# 22 打包

## 安装打包工具

```shell
$ /Library/Frameworks/Python.framework/Versions/3.9/bin
$  pip install PyInstaller
```

## 生成可执行文件

```shell
$ pyinstaller -F xxx/xxx.py 
```

可执行文件输出位置 `16311 INFO: Copying bootloader EXE to /xxx/dist/stusystem`

