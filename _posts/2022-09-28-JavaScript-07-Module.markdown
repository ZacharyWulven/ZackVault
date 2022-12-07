---
layout: post
title: JavaScript-07-Module
date: 2022-09-28 16:45:30.000000000 +09:00
tag: javascript
---

# Modules

## 0x01 基于类、对象和闭包的模块

{% highlight ruby %}
    const stats = (function() {
        const sum = (x,y) => x + y;
        const square = x => x * x;

        function mean(data) {
            return data.reduce(sum) / data.length;
        }

        function stddev(data) {
            let m = mean(data);
            return Math.sqrt(
                data.map(x => x - m).map(square).reduce(sum) / (data.length - 1)
            );
        }
        return { mean, stddev };
    }());

    let a = stats.mean([1, 3, 5, 7, 9]); 
    console.log(a);    // 5
    a = stats.stddev([1, 3, 5, 7, 9]);
    console.log(a);    // Math.sqrt(10)
{% endhighlight %}

### 1 基于闭包的自动模块化
* 代码打包基本原理
{% highlight ruby %}
    const modules = {};
    function require(moduleName) { return modules[moduleName]; }

    modules["sets.js"] = (function(){
        const exports = {};
        // sets.js 文件内容在这里
        exports.BitSet = class BitSet { };
        return exports;
    }());

    modules["stats.js"] = (function(){
        const exports = {};

        // stats.js 文件内容在这里
        const sum = (x,y) => x + y;
        const square = (x,y) => x * y;
        exports.mean = function(data) { };
        exports.stddev = function(data) { };
        return exports;
    }());

    const stats = require("stats.js");
    const BitSet = require("sets.js").BitSet;

    let s = new BitSet(100);
    console.log(s);
{% endhighlight %}

## 0x02 Node 中的模块
* 在 Node 中每个文件都是一个拥有私有命名空间的独立模块，在文件中定义的常量、变量、函数和类对该文件而言都是私有的
* 除非该文件会导出它们，而被导出的值只有被另一个模块显式的导入后才会在该模块中可见
* Note：Node 模块使用 require() 函数导入其他模块，通过 Exports 对象的数学或完全替换 module.exports 对象来导出公共 API

### 1 Node 的导出
* Node 定义了一个全局 exports 对象，如果要导出多个值可以直接把这些值设置为 exports 对象的属性
* 即把想要导出的值直接赋值给 module.exports 即可
* module.exports 的默认值与 exports 引用的是同一个对象

{% highlight ruby %}
    const sum = (x,y) => x + y;
    const square = (x,y) => x * y;
    exports.mean = data => data.reduce(sum) / data.length;
    exports.stddev = function(d) {
        let m = exports.mean(d);
        return Math.sqrt(
            data.map(x => x - m).map(square).reduce(sum) / (data.length - 1)
        );
    };
    
    module.exports = class C {};

{% endhighlight %}

* 或在最后导出

{% highlight ruby %}
    const sum = (x,y) => x + y;
    const square = x => x * x;

    function mean(data) {
        return data.reduce(sum) / data.length;
    }

    function stddev(data) {
        let m = mean(data);
        return Math.sqrt(
            data.map(x => x - m).map(square).reduce(sum) / (data.length - 1)
        );
    }
    // 最后只导出公有函数
    module.exports = { mean, stddev };
{% endhighlight %}

### 2 Node 的导入
* Node 模块通过调用 require() 函数导入其他模块，参数是要导入模块的名字，返回值是该模块的导出值（通常是一个函数或类或对象）
* 导入内置系统模块或通过包管理器安装在系统的模块，使用不带“/”的字符模块名
{% highlight ruby %}
    // 这些都是内置模块
    const fs = require("fs");
    const http = require("http");

    // 导入自己的模块,可以省略 .js 后缀，但通常会带着
    const stats = require("./stats.js");
{% endhighlight %}
* 如果模块只导出一个类或函数，则只调用 require 获得返回值即可（例如方式一）
* 如果模块是一个导出多个属性的对象，则有两个选择
1. 导入整个对象（例如方式一）
2. 通过解构赋值只导出想要使用的属性
{% highlight ruby %}
    // 方式一：导入整个模块
    const stats = require("./stats.js");   
    stats.mean(data);  // 通过 模块名.方法名 调用

    // 方式二：通过解构赋值，只导入想用的模块
    const { stddev } = require("./stats.js");
    stddev(data)   // 直接通过解构名字调用
{% endhighlight %}


## 0x03 ES6 中的模块
* ES6 为 JavaScript 添加了 import 和 export 关键字
* Note：每个文件本身都是模块，在文件中定义的常量、变量、函数和类对这个文件而言都是私有的，除非它们被显式导出
* Note：一个模块的导出值，只有在显式导入它们的模块中才能使用
* ES6 模块与常规 js 脚本区别
1. 在模块中，每个模块都有私有上下文可使用 import 和 export 关键字
2. ES6 模块中的代码自动为严格模式，即不用写 "use strict"，也意味着不能使用 with 语句和 arguments 对象和未声明的变量
3. 严格模式下函数调用中 this 的值是 undefined

* ES6 模块是通过 `<script type="module">` 标签添加到 HTML 中的
* Note：Node13 开始支持 ES6 模块，但大多数 Node 程序还是使用 Node 模块

### 1 ES6 导出
* 要从模块导出常量、变量、函数或类，只要在声明前加上 export 关键字即可
* 只导出一个值时使用 export default，这叫默认导出
* Note：使用 export 只对有名字声明的有效，而 export default 可以导出任何表达式，包括匿名函数表达式和匿名类表达式，因此 export default 后是实实在在的对象字面量
* Note： export 只能出现在代码顶层，不能在类、函数、循环、条件的内部出现
* ES6 模块特性用以支持静态分析，模块导出的值在每次运行时都相同，而导出的符号可以在模块实际运行前确定
* 一个模块即定义多个 export 和 export default 虽然合法但不常见
{% highlight ruby %}
    // 方式一：逐个声明 export
    export const PI = Math.PI;

    export function degreesToRadians(d) { return d * PI / 180;};

    export class Circle {
        constructor(r) { this.r = r; }
        area() { return PI * this.r * this.r; }
    }

    // 方式二：一起声明 export
    // 这里只是写在花括号里，不会定义对象字面量
    export { Circle, degreesToRadians, PI }
    
    -------------------------------------------
    // 只导出一个值
    export default class BitSet {
        
    }
{% endhighlight %}

### 2 ES6 导入
* 导入其他模块的值使用 import 关键字
* 获得导入值的标识符是一个常量，就像使用 const 关键字一样
* Note：导入与函数声明类似，会被“提升”到顶部，因此所有导入的值在模块代码运行时都是可用的
* from 的模块必须使用“/” 路径，类似一个 URL
{% highlight ruby %}
    import BitSet from './bitset.js';
    
    // 导入可以少于导出的
    import { mean, stddev } from './stats.js';


    // 导入 stats.js 所有值，并将其赋值给 stats 常量，每一个导出值都会成为 stats 的属性
    // 使用 stats.mean() 使用它们
    import * as stats from './stats.js';
    
    
    // 导入默认导出和非默认导出值
    import Histogram, { mean, stddev } from './histogram-stats.js';
{% endhighlight %}

* 默认导出在定义它们时没有名字，在导入时可以提个一个局部名
* 非默认导出在它们模块有名字，所以导入时需要通过名字引用它们

#### 导入没有任何导出的模块
{% highlight ruby %}
这种模块会在首次导入运行一次（之后再导入时则什么都不做）
Note：那些有导出的模块也可以使用这种什么也不导入的 import 语法
    import "./analytics.js";
{% endhighlight %}

### 3 导入和导出时重命名
* 使用 as 关键字对导入或导出值进行重命名

{% highlight ruby %}
    import { render as renderImage } from "./imageutils.js";
    import { render as renderUI } from "./ui.js";


    // 同时导入默认导出和命名导出
    // default 充当占位符，指明导入的是 default 导出
    import { default as Histogram, mean, stddev } from './histogram-stats.js';


    // 导出值时也可重命名，但仅限于 export 语句花括号
    // as 前需要标识符，而不是表达式
    export {
        layout as calculateLayout,
        render as renderLayout
    }
{% endhighlight %}

### 4 再导出
* 假如 ./stats/mean.js 中定义了 mean 函数， ./stats/stddev 中定义了 stddev 函数，有时会单独使用 mean 和 stddev，但有的程序可能同时用到它们俩，可以使用下面方法再导出
{% highlight ruby %}
    // 方式一：导入和导出分开
    import { mean } from "./stats/mean.js";
    import { stddev } from "./stats/stddev.js";
    export { mean, stddev };

    // 方式二：导入和导出合并
    export { mean } from "./stats/mean.js";
    export { stddev } from "./stats/stddev.js";


    // 方式三：导出所有值
    export * from "./stats/mean.js";
    export * from "./stats/stddev.js";

    // 方式四：导出并重命名
    export { mean, mean as average } from "./stats/mean.js";

    // 如果是默认导出
    export { default as mean } from "./stats/mean.js";

    //如果想将另一个模块命名符号再导出为默认导出
    export { mean as default } from "./stats.js";

    // 把一个模块的默认导出再导出为默认导出
    export { default } from "./stats/mean.js";
{% endhighlight %}

### 5 在网页中使用 JavaScript 模块
* 模块代码在严格模式下运行，this 不引用全局对象
* 如果想在浏览器中以原生方式使用 import 指令，必须以 `<script type="module">` 标签告诉浏览器你的代码是一个模块
* 语法：`<script type="module">import "./main.js";</script>`
* HTML 解析完成，脚本就会按照它们在 HTML 文档中出现顺序执行
* 添加了 async 属性的模块会在代码加载完备后立即执行，而不管 HTML 解析是否完成，同时也可能改变脚本执行的相对顺序
* 支持 `<script type="module">` 的浏览器必须支持 `<script nomodule>` 支持模块的浏览器会忽略带 nomodule 属性的脚本，不执行他们
* `<script type="module">` 增加了跨源加载限制，即只能从包含模块的 HTML 文档所在的域加载模块，除非服务器添加了适当的 CORS 头部允许跨源加载

### 6 通过 import() 动态导入
* 之前讲的 import 和 export 都是静态的。静态模块导入需要先加载完全部程序在执行

* 传给 import() 一个模块标识符，他就会返回一个 Promise 对象，表示加载和运行指定模块的异步过程。动态导入完成后，这个 Promise 对象会”兑现“并产生一个对象，与使用静态导入语句
import * as 得到的对象类似。

{% highlight ruby %}
    // 如果是静态导入 这么写
    import * as stats from "./stats.js";

    // 如果是动态导入 这么写
    import("./stats.js").then(stats => {
        let average = stats.mean(data);
    });
    // 或在 async 动态导入
    async analyzeData(data) {
        let stats = await import("./stats.js");
        return {
            average: stats.mean(data),
            stddev: stats.stddev(data),
        };
    }
动态 import() 看起来像函数调用，但其实并不是，实际上 import() 是一个操作符，而这个圆括号则是这个操作符语法必须的部分
{% endhighlight %}
* 通过有意识的使用动态 import() 可以将一个大文件拆分成多个小文件，实现按需加载

### 7 import.meta.url
* import.meta 引用一个对象，这个对象包含当前执行模块的元数据，其中这个对象的 url 属性是加载模块时使用的 URL
* import.meta.url 的主要使用场景是引用与模块位于同一或相对目录下的图片、数据文件或其他资源

{% highlight ruby %}
    // 加载本地字符串，l10n 是目录
    function localStringsURL(locale) {
        return new URL(`l10n/${locale}.json`, import.meta.url);
    }
{% endhighlight %}

