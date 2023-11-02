---
layout: post
title: Rust 中级教程-接口设计建议-03-Obvious
date: 2023-10-08 16:45:30.000000000 +09:00
categories: [Rust, Rustaceans]
tags: [Rust, Rustaceans]
---


# 4 显而易见（Obvious）

## 4.1 前言

* 你接口的用户通常是不了解你接口实现的细节，所以他可能不会完全理解接口的所有规则和限制

> 因此，重要的一点是，要让用户容易理解你的接口，并难以用错，通过 Rust 文档和类型系统可以基本实现这个需求
{: .prompt-info }


## 4.2 文档

### 让接口透明化的第一步就是：写出好的文档，我们来看看下面几点

* 1 清楚的记录
  * 可能出现意外的情况，或它依赖于用户执行超出类型签名要求的操作
  * 例如 `panic` 如果你的代码会发生 `panic`，需要记录在文档中，说明什么情况下会发生 `panic`
  * 例如 返回错误，也要在文档中写明什么情况会返回错误
  * 例如 `unsafe` 函数，要写明用户要满足什么条件才能安全的调用这个函数
  * 看个例子
  

```rust
/// 除法运行，返回两个数的结果
/// 
/// # Panics
/// 
/// 如果除数为零，该函数会发生 panic。
/// 
/// # 示例
/// 
/// ````
/// let result = divide(10, 2);
/// assert_eq!(result, 5);
/// ````
pub fn divide(dividend: i32, divisor: i32) -> i32 {
    // Code...
    todo!()
}
```

* 2 要写出好的文档，需要在 `crate 或 module 级别`，包括端到端的用例
  * 而不是针对特定类型或方法，这么做的好处是让用户可以了解所有内容如何组合到一起
  * 并且让用户对接口的整体结构有一个相对清晰的理解
    * 从而让开发者快速了解到各方法和类型的功能，以及在哪使用它们
  * 你提供这个用例后，用户就可以把它们直接粘贴使用了，这就相当于提供了定制化使用的起点，让其结合自己的需求进行修改


* 3 组织好文档
  * 即利用模块来将语义相关的项进行分组
  * 使用内部的文档链接将这些项相互链接起来
  * 可以考虑使用 `#[doc(hidden)]` 标记那些不打算公开，但出于遗留原因需要的接口部分，避免弄乱文档
  * 下面是例子
  
  
```rust
// 一个简单的模块，包含一些用于内部使用的函数和结构体
pub mod internal {
    /// 一个拥有内部计算辅助函数 (因为标记了 #[doc(hidden)]，所以这个句不会出现在文档中)
    #[doc(hidden)]
    pub fn internal_helper() {
        // 内部计算实现...
    }

    /// 一个仅用于内部使用的结构体
    #[doc(hidden)]
    pub struct InternalStruct {
        // 字段和方法
    }
}

// 一个公共接口函数，调用了内部辅助函数
pub fn public_function() {
    // 调用内部辅助函数
    internal::internal_helper();
}
```

* 4 尽可能的丰富你的文档
  * 有时需要解释一些概念，就可以添加到外部资源的链接
    * 例如相关规范文档（RFC）、博客、白皮书等等
  * 可使用 `#[doc(cfg(..))]` 突出显示仅在特定配置下可用的项
    * 这样用户就能快速了解到，为什么在文档中列出的某个方法不可用，因为其只能在特定的条件下使用
  * 可使用 `#[doc(aliias = "...")]` 可以让用户以其他名称搜索到类型和方法
  * 在顶层文档中，引导用户了解常用的模块、Trait、类型、方法等等
  * 下面看两个例子

```rust
///! 这是一个用于处理图像的库
///!
///! 这个库提供了一些常用的图像处理功能，例如
///! - 读取和保存不同格式的图像文件 [`Image::load`] [`Image::save`]
///! - 调整图像的大小、旋转和裁剪 [`Image::resize`] [`Image::rotate`] [`Image::crop`]
///! - 应用不同的滤镜和效果 [`Filter`] [`Effect`]
///! 
///! 如果您想了解更多关于图像处理的原理和算法，您可以参考以下资源：
///! - [数字图像处理](https://book.xxx.com/subject/xxxx)，一本经典教科书，介绍图像处理的基本概念
///! - [Learn OpenCV](https://learnopencv.com)，一个网站，提供很多用 OpenCV 实现图像处理功能的教程和示例代码
///! - [Awsome Computer Vision](https://github.com/jbhuang0604/awesome-computer-vision)，一个仓库 

/// 一个表示图像的结构体
#[derive(Debug, Clone)]
pub struct Image {

}

impl Image {
    /// 从指定路径加载一个图像文件
    /// 
    /// 支持的格式有：PNG、JPEG、GIF、BMP 等
    /// 
    /// # 参数
    /// 
    /// - `path`: 图像文件的路径
    /// 
    /// # 返回值
    /// 
    /// 如果成功，返回一个 [`Image`] 实例；如果失败，返回一个 [`Error`]
    /// 
    /// # 示例
    /// 
    /// ```no_run
    /// use image::Image;
    /// 
    /// let img = Image::load("test.png")?;
    /// ```
    #[doc(alias = "读取")]
    #[doc(alias = "打开")]
    pub fn load<P: AsRef<Path>>(path: P) -> Result<Self, Error> {
        todo!()
    }
}
```
  
  
```rust



```
  

<!--![image](/assets/images/rust/web_server/teacher_aim.png)-->







