---
layout: post
title: Python 从入门到实践
date: 2023-11-01 16:45:30.000000000 +09:00
categories: [Python, 从入门到实践]
tags: [Python, 从入门到实践]
---

# 1 测试代码

## 安装 `pytest`

* 更新 pip

```shell
# 格式：
$ python3 -m pip install --upgrade [要更新的包名]

# 更新 pip
$ python3 -m pip install --upgrade pip


# 安装 pytest
# --user 让 Python 只为当前用户安装指定的包
$ python3 -m pip install --user pytest
```


* 当你让 `pytest` 运行测试时，它将查找以 `test_` 打头的文件，并运行其中所有的测试


## 测试用例

```python
# 创建测试文件 test_name_function.py

from name_function import get_formatted_name

def test_first_last_name():
    """ Janis Joplin """
    formatted_name = get_formatted_name('janis', 'joplin')
    assert formatted_name == 'Janis Joplin'
    
    


# name_function.py
def get_formatted_name(first, last):
    """               """
    full_name = f"{first} {last}"
    return full_name.title()
```

> 如果出现一条消息，提示没有找到命令 `pytest`， 请执行命令 `python3 -m test`
{: .prompt-info }


* 执行 `pytest`，会自动找 `test` 开头的文件里的用例，下边为测试通过的情况

```shell
(venv) ➜  pythonGuide pytest
============================================================= test session starts ==============================================================
platform darwin -- Python 3.9.13, pytest-7.4.3, pluggy-1.3.0
rootdir: /Users/zacharyah/Tech/Python/pythonGuide
collected 1 item                                                                                                                               

test_name_function.py .                                                                                                                  [100%]

============================================================== 1 passed in 0.02s ===============================================================
```

* 测试未通过时的输出，测试文件后边会有个 `F`

```shell
(venv) ➜  pythonGuide pytest
============================================================= test session starts ==============================================================
platform darwin -- Python 3.9.13, pytest-7.4.3, pluggy-1.3.0
rootdir: /Users/zacharyah/Tech/Python/pythonGuide
collected 1 item                                                                                                                               

test_name_function.py F                                                                                                                  [100%]

=================================================================== FAILURES ===================================================================
_____________________________________________________________ test_first_last_name _____________________________________________________________

    def test_first_last_name():
        """ Janis Joplin """
        formatted_name = get_formatted_name('janis', 'joplin')
>       assert formatted_name != 'Janis Joplin'
E       AssertionError: assert 'Janis Joplin' != 'Janis Joplin'

test_name_function.py:6: AssertionError
=========================================================== short test summary info ============================================================
FAILED test_name_function.py::test_first_last_name - AssertionError: assert 'Janis Joplin' != 'Janis Joplin'
============================================================== 1 failed in 0.06s ===============================================================
```


## 断言
* `assert a == b`：断言两个值相等
* `assert a != b`：断言两个值不相等
* `assert a`：断言 a 的布尔值为 True
* `assert not a`：断言 a 的布尔值为 False
* `assert element in list`：断言元素在列表中
* `assert element not in list`：断言元素不在列表中


## 测试类


```python
# survey.py
class AnonymousSurvey:
    """收集匿名调查问卷的答案"""

    def __init__(self, question):
        """ """
        self.question = question
        self.responses = []

    def show_question(self):
        """显示调查问卷"""
        print(self.question)

    def store_response(self, new_response):
        """存储单份调查问卷"""
        self.responses.append(new_response)

    def show_results(self):
        """ """
        print("Survey results:")
        for response in self.responses:
            print(f"- {response}")



# test_survey.py
from survey import AnonymousSurvey

def test_store_single_response():
    """ """
    question = "What language did you first learn to speak?"
    language_survey = AnonymousSurvey(question)
    language_survey.store_response('English')
    assert 'English' in language_survey.responses


def test_store_three_responses():
    """ """
    question = "What language did you first learn to speak?"
    language_survey = AnonymousSurvey(question)
    responses = ['English', 'Spanish', 'Mandarin']
    for response in responses:
        language_survey.store_response(response)
    for response in responses:
        assert response in language_survey.responses
```

* 指定要测试的文件名进行测试

```shell
$ pytest test_survey.py
```

## 使用 Fixture
* Fixture 可帮助我们搭建测试环境。以便多个测试使用资源
* 在 `pytest` 中创建 Fixture，可编写一个使用装饰器 `@pytest.fixture` 装饰的函数
* 装饰器是放在函数定义前面的指令，在运行函数前，Python 将该指令应用于函数，以修改函数代码的行为。

```python
# test_survey.py

import pytest
from survey import AnonymousSurvey


@pytest.fixture
def language_survey():
    """一个可供所有测试函数使用的 AnonymousSurvey 实例"""
    question = "What language did you first learn to speak?"
    language_survey = AnonymousSurvey(question)
    return language_survey


def test_store_single_response(language_survey):
    """ """
    # question = "What language did you first learn to speak?"
    # language_survey = AnonymousSurvey(question)
    language_survey.store_response('English')
    assert 'English' in language_survey.responses


def test_store_three_responses(language_survey):
    """ """
    # question = "What language did you first learn to speak?"
    # language_survey = AnonymousSurvey(question)
    responses = ['English', 'Spanish', 'Mandarin']
    for response in responses:
        language_survey.store_response(response)
    for response in responses:
        assert response in language_survey.responses
```


# 2 数据可视化
* 一个流行的数据工具是 Matplotlib，它是一个数学绘图库
* 本节使用它来制作简单的绘图（plot），如折线图和散点图，还将基于随机游走概念（根据一系列随机决策生成图形）生成数据集
* 使用 Plotly 包来分析投骰子结果


## 安装 Matplotlib

```shell
$ python3 -m pip install matplotlib 
```


## 线图

```python
import matplotlib.pyplot as plt

# input_values 用于 x 轴
input_values = [1, 2, 3, 4, 5]

squares = [1, 4, 9, 16, 25]

# 设置内置样式, 可使用 print(plt.style.available) 进行查看
# 需要在 subplots() 前设置
plt.style.use('seaborn-v0_8')

# subplots 可在一个图形中绘制一个或多个绘图
# fig 表示由生成的一系列绘图构成的整个图形
# ax 表示图形中的绘图，大多数情况下，使用这个变量来定义和定制绘图
fig, ax = plt.subplots()

# 根据给定数据绘制图形, 设置线条粗细 linewidth
ax.plot(input_values, squares, linewidth=3)

# 设置图形并给坐标轴加标签
ax.set_title("Square Numbers", fontsize=24)    # 设置标题
ax.set_xlabel("Value", fontsize=14)
ax.set_ylabel("Square of Value", fontsize=14)

# 设置刻度标记样式
ax.tick_params(labelsize=14)

# 打开 matplotlib 查看器显示绘图
plt.show()
```

![image](/assets/images/python/guide/line.png)


## 点图

```python
import matplotlib.pyplot as plt


x_values = range(1, 1001)
y_values = [x ** 2 for x in x_values]

# 设置内置样式, 可使用 print(plt.style.available) 进行查看
# 需要在 subplots() 前设置
plt.style.use('seaborn-v0_8')
fig, ax = plt.subplots()

# s 为设置点的尺寸
# color 设置点的颜色
# ax.scatter(x_values, y_values, s=10, color='red')

# 使用 RGB 方式定制颜色, color 为一个元组,范围为 0~1
# ax.scatter(x_values, y_values, s=10, color=(0, 0.8, 0))

# 颜色映射（渐变色）
# pyplot 模块内置了一组颜色映射
# 参数 c 类似参数 color，但用于将一系列值关联到颜色映射
ax.scatter(x_values, y_values, s=10, c=y_values, cmap=plt.cm.Blues)



# 设置图形并给坐标轴加标签
ax.set_title("Square Numbers", fontsize=24)    # 设置标题
ax.set_xlabel("Value", fontsize=14)
ax.set_ylabel("Square of Value", fontsize=14)

# 设置刻度标记样式
# ax.tick_params(labelsize=14)



# 指定每个坐标轴的取值访问
# x 轴范围为 0~1100
# y 轴范围为 0~1_100_000
ax.axis([0, 1100, 0, 1_100_000])

# 定制刻度标记，这样当数足够大时，能够减少内存压力
ax.ticklabel_format(style='plain')


# 打开 matplotlib 查看器显示绘图
# plt.show()

# 如果要将图形保存到文件中，而不是 matplotlib 查看器中，可将 plt.show() 替换为 plt.savefig()
# bbox_inches='tight' 将绘图多余的空白区域裁剪掉
# 还可以使用 Path 对象，指定存储图片的位置
plt.savefig('scatter_plot.png', bbox_inches='tight')
```
![image](/assets/images/python/guide/scatter_plot.png)


## 随机游走


```python
# random_walk.py

from random import choice

class RandomWalk:
    """一个生成随机游走数据的类"""

    def __init__(self, num_points=5000):
        """初始化随机游走的属性"""
        self.num_points = num_points

        # 所有随机游走的始于 (0, 0)
        self.x_values = [0]
        self.y_values = [0]

    def fill_walk(self):
        """计算随机游走包含的所有点"""
        """告诉 Python 如何模拟四种游走策略"""

        # 不断游走，直到列表达到指定长度
        while len(self.x_values) < self.num_points:
            # 决定前进的方向以及沿这个方向前进的距离
            x_direction = choice([1, -1])        # 向左还是向右走
            x_distance = choice([0, 1, 2, 3, 4]) # 走的距离
            x_step = x_direction * x_distance
            y_direction = choice([1, -1])
            y_distance = choice([0, 1, 2, 3, 4])
            y_step = y_direction * y_distance

            # 拒绝原地踏步
            if x_step == 0 and y_step == 0:
                continue
            # 计算下一点的 x、y 坐标值
            x = self.x_values[-1] + x_step
            y = self.y_values[-1] + y_step
            self.x_values.append(x)
            self.y_values.append(y)
            
     

# rw_visual.py

import matplotlib.pyplot as plt
from random_walk import RandomWalk

rw = RandomWalk()
rw.fill_walk()

# 将所有点绘制出来
plt.style.use('classic')
fig, ax = plt.subplots()

ax.scatter(rw.x_values, rw.y_values, s=10)

# 默认情况 matplotlib 独立缩放每个轴，aspect 指定两条轴上刻度的间距必须相等
ax.set_aspect('equal')
plt.show()
```

![image](/assets/images/python/guide/random_walk.png)


### 定制随机游走图样式

```python
import matplotlib.pyplot as plt
from random_walk import RandomWalk

# rw = RandomWalk()
# rw.fill_walk()
#
# # 将所有点绘制出来
# plt.style.use('classic')
# fig, ax = plt.subplots()
#
# ax.scatter(rw.x_values, rw.y_values, s=10)
#
# # 默认情况 matplotlib 独立缩放每个轴，aspect 指定两条轴上刻度的间距必须相等
# ax.set_aspect('equal')
# plt.show()

while True:

    # rw = RandomWalk(50_000)
    rw = RandomWalk()
    rw.fill_walk()

    # 将所有点都绘制出来
    plt.style.use('classic')
    # figsize 设置输出尺寸 分辨率
    fig, ax = plt.subplots(figsize=(15, 9), dpi=128)

    point_numbers = range(rw.num_points)
    ax.scatter(rw.x_values, rw.y_values, s=10, c=point_numbers,
               cmap=plt.cm.Blues, edgecolors='none')

    # 默认情况 matplotlib 独立缩放每个轴，aspect 指定两条轴上刻度的间距必须相等
    ax.set_aspect('equal')

    # 突出起点和终点， 重新绘制第一个点和最后一个点
    ax.scatter(0, 0, c='green', edgecolors='none', s=100)
    ax.scatter(rw.x_values[-1], rw.y_values[-1], s=10, c='red', edgecolors='none')

    # 隐藏坐标轴
    ax.get_xaxis().set_visible(False)
    ax.get_yaxis().set_visible(False)

    plt.show()
    print('hello')
    keep_running = input('Make another walk?(y/n): ')
    if keep_running == 'n':
        break
```

![image](/assets/images/python/guide/random_walk_2.png)


## 使用 Plotly 模拟骰子



