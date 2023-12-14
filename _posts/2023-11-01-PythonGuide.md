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



