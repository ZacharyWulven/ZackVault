---
layout: post
title: 使用 Rust 构建微型游戏
date: 2023-10-14 16:45:30.000000000 +09:00
categories: [Rust, Rust Game]
tags: [Rust, Rust Game]
---


# 使用 Rust 构建微型游戏
* 以便理解游戏开发的关键概念


# 理解 `Game loop`
* 为了让游戏更流畅、顺滑的运行，需要使用 `Game loop`

## `Game loop` 作用
* 初始化窗口、图形和其他资源
* 每当屏幕刷新时（通常是每秒 30 次或 60 次，或更多次），`Game loop` 都会运行
* 每次通过循环，它都会调用游戏的 `tick()` 函数
  * 然后我们就可以在这个函数中做一些操作


* 流程图

![image](/assets/images/rust/game_loop.png)

# 游戏引擎
* 通常是需要使用游戏引擎的
* 游戏引擎可以用来处理平台特定的部分
* 它的目的是让开发者更专注于开发游戏，从而避免开发者把精力浪费在与游戏开发内容无关的一些事情上

## `Bracket-Lib`
* 本例使用 `Bracket-Lib`，它是一个 Rust 游戏编程库
* `Bracket-Lib` 是一个作为简单的教学工具
* 抽象了游戏开发很多复杂的东西，但是却保留了一些相关的概念
* `Bracket-Lib` 包括很多库：如随机数生成、几何、路径寻找、颜色处理、常用算法等

### `Bracket-terminal`
* 它是 `Bracket-Lib` 中负责显示的部分
* 它提供了模拟控制台
* 可以与多种渲染平台配合：
  * 从文本控制台到 `Web Assembly`
  * 渲染平台包括：`OpenGL、Vulkan、Metal`
* 它支持 `sprites` 和原生 `OpenGL` 开发

# `Codepage 437`
* 它是 IBM 扩展的 ASCII 字符集
* 它是来自 `Dos PC` 上的字符，用于终端输出，除了字母和数字，还提供了一些特殊符号
* `Bracket-Lib` 库会把字符翻译成图形 `sprites` 并提供一个有限的字符集，字符所展示的是相应的图片


# 游戏的模式
* 游戏通常在不同的模式中运行
* 每种模式会明确游戏在当前的 `tick()` 中应该做什么

## 本教程需要 3 种模式
* 菜单
* 游戏中
* 结束







> 
{: .prompt-info }



