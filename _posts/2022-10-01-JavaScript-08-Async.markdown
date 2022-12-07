---
layout: post
title: JavaScript-08-Async
date: 2022-10-01 16:45:30.000000000 +09:00
tag: javascript
---

# 异步 JavaScript
* 本章将介绍三种重要语言特性，让编写异步代码更容易
* ES6 新增的 Promise 是一种对象，代表某个异步操作上不可用的结果
* async 和 await 允许开发者将基于 Promise 的异步代码写成同步的形式
* for/await 循环，允许在看起来同步的简单循环中操作异步事件流

## 0x01 使用回调的异步编程
* 在基本层面，JavaScript 异步编程是使用回调函数实现的

### 1 定时器
* setTimeout
* setInterval
{% highlight ruby %}
    // 只调用一次
    // 第一个参数是回调函数
    // 第二个参数是时间间隔，单位是毫秒
    setTimeout(() => {
        // 在 setTimeout 调用 600 毫秒后调用
        console.log("setTimeout ")
    }, 600);

    // 调用多次
    let i = 0
    let updateInterval = setInterval(() => {
        console.log("setInterval")
        i++;
        if (i > 3) {
            console.log("clearInterval")
            // clearInterval 用于停止重复调用
            clearInterval(updateInterval);
        }
    }, 600);
{% endhighlight %}

### 2 事件
* 回调函数叫事件处理程序或者事件监听器是通过 addEventListener() 注册的

{% highlight ruby %}
    // applyUpdate 是回调函数
    // addEventListener 第一个参数是字符串指定要注册的事件类型
    // 如果用户点击网页中指定元素就会调用 applyUpdate 函数
    let okay = document.querySelector('#confirmUpdateDialog button.okay');
    okay.addEventListener('click', applyUpdate);
{% endhighlight %}


### 3 网络事件
{% highlight ruby %}
    function getCurrentVersionNumber(versionCallback) {
        let request = new XMLHttpRequest();
        request.open("GET", "http://www.example.com/api/version");
        request.send();

        request.onload = function() {
            if (request.status === 200) {
                let currentVersion = parseFloat(request.responseText);
                versionCallback(null, currentVersion);
            } else {
                versionCallback(response.statusText, null);
            }
        };

        request.onerror = request.ontimeout = function(e) {
            versionCallback(e.type, null);
        };
    }
{% endhighlight %}

### 4 Node 中的回调与事件
* Node.js 服务器端 JavaScript 环境底层就是异步的，定义了很多使用回调和事件的 API
{% highlight ruby %}
读取文件
    const fs = require("fs"); // fs 模块由文件系统 API
    let options = {
        // 默认选项定义在这里
    };
    // 接收两个参数的回调作为最后一个参数, 异步读取指定文件，然后调用回调函数
    fs.readFile("config.js", "utf-8", (err, text) => {
        if (err) {
            // 如果有错误 显示一条警告消息 
            console.warn("Could not read config file:", err);
        } else {
            // 否则 解析文件内容并赋值给选项对象
            Object.assign(options, JSON.parse(text));
        }
        // 启动程序
        startProgram(options);
    });
{% endhighlight %}

通过 http 请求获取 URL 内容
{% highlight ruby %}
    const https = require("https");
    function getText(url, callback) {
        // 对 url 发送 get 请求
        request = https.get(url);

        // 注册一个 response 监听事件处理程序
        request.on("response", response => {
            let httpStatus = response.statusCode;

            response.setEncode("utf-8");
            let body = "";

            // 每个响应体块就绪时都会调用这个事件处理程序
            response.on("data", chunk => { body += chunk; });

            response.on("end", () => {
                if (httpStatus === 200) {
                    callback(null, body);
                } else {
                    callback(httpStatus, null);
                }

            });
        });

        // 注册一个错误监听事件处理程序
        request.on("error", (err) => {
            callback(err, null);
        });
    }
{% endhighlight %}

## 0x02 Promise
* Promise 是一个对象，表示异步操作的结果。这个结果可能就绪也可能未就绪。
* Note：不能同步取 Promise 的值，只能要求 Promise 在值就绪时调用一个回调函数
* 调用者可以在 Promise 注册一个或多个回调函数，当异步计算完成时，它们会被调用
* Promise 可以让回调多层嵌套变为更线性的 Promise 链形式表达出来，更容易阅读和推断
* Promise 还使异步错误处理标准化了
* Note：不能使用 Promise 表示重复的异步计算，Promise 可以写一个 setTimeout 函数替代版，但不能写 setInterval 函数替代版，因为 setInterval 重复调用回调函数，
而 Promise 不能重复调用回调

### 1 使用 Promise
* Note：Promise 表示异步计算的未来结果，计算在返回 Promise 对象之后执行的。
* 基于 Promise 的异步计算在正常结束后，则会把计算结果传给作为 then 的第一个参数的函数
* 可以把 then 函数想象成客户端 JavaScript 中注册事件处理程序的 addEventListener() 函数
* 如果多次调用一个 Promise 的 then 方法，则指定的每个函数都会在预期计算完成后被调用
* Note：通过 then 注册的函数都只会被调用一次
* 即使调用 then 时异步计算已经完成，传给 then 的函数也会被异步调用
* then 是 Promise 独有的特性
* 基于 Promise 的异步计算把异常传给 then 的第二个参数的函数
{% highlight ruby %}
    getJSON(url).then(jsonData => {
        // callback func

    });
    
    
    // 第二个参数 handleProfileError 实现错误处理
    // 如果 getJSON 调用正常执行 displayUserProfile 函数，
    // 否则出错的话调用 handleProfileError 函数
    getJSON(url).then(displayUserProfile, handleProfileError);


    /*
    上边写法如果 displayUserProfile 出现异常无法处理
    改成这样
    如果 getJSON 调用正常，执行 displayUserProfile 函数，
    如果 getJSON 调用或 displayUserProfile 出现异常，则调用 handleProfileError

    catch 方法只是对调用 then 时以 null 作为第一个参数，以指定的错误处理函数作为第二个参数
    */
    getJSON(url).then(displayUserProfile).catch(handleProfileError);

{% endhighlight %}

#### Promise 相关术语
* 讨论 JavaScript Promise 时，术语是 ”兑现（fulfill）“ 和被”拒绝（reject）“
* 调用一个 Promise 的 then 方法时传入两个回调函数，如果第一个回调函数被调用，称为得到了”兑现“。而如果第二个回调函数被调用，称为被”拒绝“
* 如果 Promise 即未兑现，也未被拒绝，那它就是待定（pending）
* 而 Promise 一旦兑现或被拒绝，我们说它已经落定（settle），永远不会从兑现变为拒绝，vice verse

* Note：Promise 是一个对象，表示异步操作的结果
* Promise 被兑现则结果就是返回值，如果被拒绝结果就是 Error 或某个其他值
* Note：如果 Promise 被兑现结果会传给 then 第一个函数，如果被拒绝结果值会传给 catch 注册或作为 then 的第二个参数注册的回调函数

* Promise 有可能被解决（resolve）。它与兑现和落定不同，理解被解决是深刻理解 Promise 的关键

### 2 Promise 链
* Promise 一个最重要的优点就是以线性 then 方法调用链形式表达一连串异步操作，而无需把每个操作嵌套在前一个操作的回调内部
{% highlight ruby %}
    // 发送 HTTP 请求
    fetch(documentURL).then(response => response.json()) // 获取 JSON 格式的响应体
                      .then(document => {                // 在取得解析后的 JSON 时
                        return render(document);         // 把文档显示给用户
                      })
                      .then(rendered => {                // 在取得渲染的文档后
                        cacheInDatabase(rendered);       // 把它缓存在本地数据库中
                      })
                      .catch(error => handle(error));    // 处理发生的错误
{% endhighlight %}

* Note：每个 then 方法调用都返回一个新 Promise 对象，这个 Promise 对象在传给 then 的函数执行结束才会”兑现“
{% highlight ruby %}
    // fetch() 传给他一个 URL，返回一个 Promise
    fetch("/api/user/profile").then(response => {
        // 在 Promise 解决时，可以访问 HTTP 状态和头部
        if (response.ok && response.headers.get("COntent-Type") === "application/json") {
            // 现在还没有得到响应体
        }
    })

    // 一个嵌套的幼稚写法
    fetch("/api/user/profile").then(response => {
        response.json().then(profile => {
            // 在响应体到达时，他会被自动被解析为 JSON 格式并传入这个函数
            displayUserProfile(profile);
        });
    })

    // Promise 应有的方式: Promise 链
    fetch("/api/user/profile")
    .then(response => {
        return response.json();
    })
    .then(profile => {
        displayUserProfile(profile);
    });

    // 可以理解为这样
    fetch(theURL)     // 任务 1，返回 Promise 1
    .then(callback1)  // 任务 2，返回 Promise 2，任务 2 在 callback1 被调用时开始
    .then(callback2); // 任务 3，返回 Promise 3 
Note：没有给 Promis 3 注册回调，则它落定时什么也不会发生
{% endhighlight %}

### 3 Note：解决 Promise
* 上边例子实际还有 Promise 4 对象

* 过程：当回调 c 传给 then 时，then 返回一个 p（Promise），并安排好在将来某时刻异步调用 c，届时回调 c 执行返回一个值 v，当回调返回 v 时，p 就以这个值得到了解决
1. 当 Promise 以一个非 Promise 的值解决时，就会立即以这个值兑现，即如果返回值 v 是一个非 Promise，则返回值就变成 p 的值，并且 p 兑现，结束 
2. 如果 v 是 Promise，则 p 会得到解决但并未兑现。如果 v 被拒绝，则 p 也会以相同理由被拒绝，这就是 Promise “解决” 状态的含义，即解决即一个 Promise 与另一个 Promise 发生了关联（或锁定了另一个 Promise）
3. 此时我们并不知道 p 将会兑现还是被拒绝，但回调 c 已经无法控制这个结果了
4. 说 p 得到了解决，即现在它的命运完全取决于 Promise 会怎么样

* 解决并不等同于兑现
{% highlight ruby %}
    fetch("/api/user/profile")    // Promise 1
    .then(response => {           // Promise 2
        return response.json(); // 第 4 个 Promise, 即表示 JSON 的 Promise 4 
    })
    .then(profile => {        // Promise 3
        displayUserProfile(profile);
    });
    
{% endhighlight %}

### 4 Promise 和 Error
* 同步代码中，如果不写异常处理逻辑，至少会看到异常和堆栈追踪，从而排查问题
* 异步代码，未处理异常往往不会得到报告，错误会静默发生，很难调试。catch 方法可以让 Promise Error 更容易处理
* Promise 的 catch 方法实际是对以 null 为第一个参数，以 Error 处理回调为第二个参数的 then 调用的简写
1. p.then(null, callback) 等价于 catch(callback)
* 同步代码中异常会沿着调用栈向上冒泡，直到碰到一个 catch 块
* 传统异常处理在异步代码中不适用
* 异步 Promise 链，类似沿着 Promise 链向下流淌，直到碰到一个 catch() 调用

#### .finally()
* ES2018加了 finally(),如果你在 Promise 链添加了 .finally()，则传给 .finally() 的回调会在 Promise 落地时被调用，无论兑现还是被拒绝 .finally() 回调都会被调用，
并且调用时不会给它传任何参数。
* .finally() 可用于做一些清理工作，比如关闭文件
* .finally() 也返回一个新的 Promise
* .finally() 返回值通常会被忽略，而解决或拒绝调用 .finally() 的 Promise 值一般也会用来解决或拒绝 .finally() 返回的 Promise，不过 .finally() 回调抛出异常，就会用
这个错误值拒绝 .finally() 返回的 Promise

* Promise 链最后加 catch 是惯例
{% highlight ruby %}
    fetch("/api/user/profile")
    .then(response => {         // 在状态和头部就绪时调用
        if (!response.ok) {  
            return null;
        }

        // 检查头部确保服务器发送的是 JSON
        let type = response.headers.get("content-type");
        if (type !== "application/json") {
            throw new TypeError(`Expected JSON, got ${type}`);
        }

        // 返回一个 Promise 
        return response.json();
    })
    .then(profile => {
        if (profile) {
            displayUserProfile(profile);
        } else {
            displayLoggedOutProfilePage();
        }
    })
    .catch(e => {
        if (e instanceof NetworkError) {
            displayErrorMessage("check your internet connection");
        } else if (e instanceof TypeError) {
            displayErrorMessage("something is wrong with our server!");
        } else {
            console.error(e);
        }
    });
{% endhighlight %}

* 如果 Promise 链某一环会因为错误而失败，而该错误是可恢复的，即后续环境还可以运行那么在中间插入一个 .catch() 是可以的
* 一个错误只要传给 catch 就会停止在 Promise 链中向下传递
* catch 的返回值会用于解决或兑现/拒绝与之关联的 Promise，如果 catch 返回正常则与之关联的 Promise 兑现或解决，从而停止错误的传播，异步链将会继续。
否则如果再抛出异常，后边的 then 代码不会被调用，错误会直接传给最后的 catch 的回调
{% highlight ruby %}
    startAsyncOperation()
    .then(callback1)
    .catch(recoverSth)     // 类似这样，如果 callback1 正常，则这个 catch 会被跳过
    .then(callback2)
    .then(callback3)
    .catch(logFinalErr)
    


    queryDatabase()
    .catch(e => wait(500).then(queryDatabase)) // 如果失败 等待并重试
    .then(displayTable)
    .catch(displayDatabaseError);
    
    
    Note：常见的错误
      .catch(e => wait(500).then(queryDatabase)) 返回 Promise
      .catch(e => { wait(500).then(queryDatabase) }) 加大括号就不行了，函数返回 undefined
{% endhighlight %}

### 5 并行 Promise
#### Promise.all()
* 并行执行多个异步操作，Promise.all() 可以做到这一点
* Promise.all() 接收一个 Promise 对象的数组作为输入，返回一个 Promise。如果数组中任意一个 Promise 拒绝，返回的 Promise 也将拒绝（会在第一个拒绝发生时立即发生）。
否则返回的 Promise 会以每个输入 Promise 兑现值的数组兑现

假如你想抓取多个 url 的内容
{% highlight ruby %}
    const urls = [];
    promises = urls.map(url => fetch(url).then(r => r.text()));
    Promise.all(promises)
    .then(bodies => {
        // 处理得到的字符串数组
    })
    .catch(e => console.error(e));
{% endhighlight %}

* Note：Promise.all() 的输入数组可以包含 Promise 兑现和非 Promise 对象值。如果某个元素不是 Promise，那么它就会当成一个已兑现 Promise 的值，被原封不动的复制到输出数组中

#### Promise.allSettled()
* Promise.allSettled() 也接收一个输入 Promise 数组，但 Promise.allSettled() 永远不拒绝返回的 Promise，而是等所有的输入 Promise 落定后兑现。
* 这个返回的 Promise 解决为一个对象数组，其中每个对象都对应一个输入 Promise，且都有一个 status 属性，值为 fulfilled 或 rejected。
1. 如果值为 fulfilled 还会有一个 value 属性，包含兑现的值。
2. 如果值为 rejected，则还会有一个 reason 属性，包含对应 Promise 的错误或拒绝理由
{% highlight ruby %}
    Promise.allSettled([Promise.resolve(1), Promise.reject(2), 3])
    .then(results => {
        results[0] // => { status: "fulfilled", value: 1 }
        results[1] // => { status: "rejected", reason: 2 }
        results[2] // => { status: "fulfilled", value: 3 }
    });
{% endhighlight %}

#### Promise.race() 
* 如果同时运行多个 Promise，但只关心第一个兑现的值，可以使用 Promise.race() 
* 返回一个 Promise，这个 Promise 会在输入数组中的 Promise 有一个兑现或拒绝时马上兑现或拒绝（或直接返回数组中非 Promise 的值）

### 6 创建 Promise
#### 基于其他 Promise 的 Promise
{% highlight ruby %}
    // Note：返回的 Promise 会解决为 response.json() 返回的 Promise
    // 当该 Promise 兑现时，getJSON 返回的 Promise 也会以相同的值兑现
    function getJSON(url) {
        return fetch(url).then(response => response.json()); // json 返回一个 Promise
    }
{% endhighlight %}

#### 基于同步值的 Promise
* 使用 Promise.resolve() 和 Promise.reject()

* Promise.resolve() 接收一个值作为参数，并返回一个会立即（但异步）以该值兑现的 Promise
* Promise.reject() 接收一个参数，并返回一个以该参数作为理由拒绝的 Promise 
* Note：这俩静态方法返回的 Promise 在被返回时并未兑现或拒绝，它们会在当前同步代码块运行结束后立即兑现或拒绝。通常在几毫秒后发生，除非有很多待定的异步任务等待运行
* 解决 Promise 并不等于兑现
* 如果把 p1 传给 Promise.resolve() 它会返回一个新 Promise p2，p2 会立即解决，但要等到 p1 兑现或被拒绝时才会兑现或被拒绝。

#### 从头开始创建 Promise
{% highlight ruby %}
    function wait(duration) {
        // 创建并返回新 Promise
        return new Promise((resolve, reject) => {
            // 如果参数无效
            if (duration < 0) {
                reject(new Error("Time travel not yet implemented"));
            }
            // 否则 异步等待，然后解决 Promise
            // setTimeout 调用 resolve 时未传参
            // 这意味着新 Promise 会以 undefined 值兑现
            setTimeout(resolve, duration);
        });
    }
    // 调用 Promise
    let a = wait(3000).then(a => {
        console.log("xxx");
    });
{% endhighlight %}

* Note：把一个 Promise 传给 Promise.resolve()，返回 Promise 会解决为该新 Promise，不过通常这里都会传一个非 Promise值，这个值会兑现返回的 Promise

{% highlight ruby %}
const { request } = require("http");

    function getJSON(url) {
        return new Promise((resolve, reject) => {
            // 向指定 URL 发送 GET 请求
            req = request.get(url, response => { //收到响应时调用
                // 如果 HTTP 状态码不对，拒绝这个 Promise
                if (response.statusCode !== 200) {
                    reject(new Error(`HTTP status ${response.statusCode}`));
                    response.resume(); // 这样不会导致内存泄露
                } else if (response.headers["content-type"] !== "application/json") {
                    reject(new Error("Invalid content-type"));
                    response.resume();
                } else {
                    // 注册事件处理程序读取响应体
                    let body = "";
                    response.setEncoding("utf-8");
                    response.on("data", chunk => { body += chunk; });
                    response.on("end", () => {
                        // 接收完全部响应体后解析它
                        try {
                            let parsed = JSON.parse(body);
                            // 如果成功解析 兑现 Promise
                            resolve(parsed);
                        } catch(e) {
                            // 解析失败，拒绝 Promise
                            reject(e);
                        }
                    });
                }
            });

            req.on("error", error => {
                reject(error);
            });
        });
    }
{% endhighlight %}

### 7 串行 Promise
假如有一个要抓取 URL 数组，你想一次只抓一个 URL，而数组是任意长度，写一个动态代码

* 方式一：fetchSequentially 先创建一个返回立即兑现的 Promise，然后基于这个初始 Promise构建一个线性的长 Promise 链并返回链中最后一个 Promise
类似多米诺骨牌
{% highlight ruby %}
    function fetchSequentially(urls) {
        //抓取 URL 时把响应体存这里
        const bodies = [];

        function fetchOne(url) {
            return fetch(url)
                .then(response => response.text())
                .then(body => {
                    // 把响应体保存到数组，这里故意省略了返回值（undefined）
                    bodies.push(body);
                });
        }
        // 从一个立即（以 undefined值）兑现的 Promise 开始
        let p = Promise.resolve(undefined)

        // 现在循环 urls 构建任意长度 Promise 链
        // 每次都取一个 URL 的响应体
        for (url of urls) {
            p = p.then(() => fetchOne(url));
        }

        /*
          Promise 链最后一个 Promise 兑现后，bodies 就绪
          因此可以将这个 bodies 数组通过过期 Promise 返回
          注意这里并未包含任何错误处理程序
          希望把错误传播给调用者处理
        */
       return p.then(() => bodies);
    }

调用代码
    fetchSequentially(urls)
    .then(bodies => {
        // 处理抓到的字符串 list
    })
    .catch(e => console.error(e));
{% endhighlight %}

* 方式二：不事先创建 Promise，而是让每个 Promise 的回调创建返回下一个 Promise

{% highlight ruby %}
    /*
    这个函数接收一个输入值数组和一个 promiseMaker 函数
    对输入数组中的任何值 x，promiseMaker 都应该返回一个
    兑现为输出值的 Promise，这个函数返回一个 Promise
    该 Promise 最终会兑现为一个包含计算得到的输出值的数组

    PromiseSequence() 不是一次创建所有 Promise 然后让他们并行运行
    而是每次只运行一个 Promise 直到上一个 Promise 兑现后，
    才会调用 promiseMaker() 计算下一个值
    */
   
    function promiseSequence(inputs, promiseMaker) {
        // 为数组创建一个可以修改的私有副本
        inputs = [...inputs];

        // 这是要用做 Promise 回调的函数
        // 它的伪递归魔术是核心逻辑
        function handleNextInput(outputs) {
            if (inputs.length === 0) {
                // 如果没有输入值了，则返回输出值数组
                // 这个数组最终兑现这个 Promise
                // 以及所有之前已经解决但尚未兑现的 Promise
                return outputs;
            } else {
                // 如果还有要处理的输入值，那么我们将返回
                // 一个 Promise 对象，把当前 Promise 解决为一个来自
                // 新 Promise 的未来值
                let nextInput = inputs.shift() // 取下一个输入值
                return promiseMaker(nextInput) // 计算下一个输出值
                    .then(output => outputs.concat(output)) // 创建一个新输出值数组
                    .then(handleNextInput) // 递归传入新的输出值数组
            }
        }
        // 从一个空数组兑现的 Promise 开始
        // 使用上面的函数作为它的回调
        return Promise.resolve([]).then(handleNextInput);
    }

    // call
    function fetchBody(url) { return fetch(url).then(r => r.text()); }
    promiseSequence(urls, fetchBody)
    .then(bodies => {
        // 处理字符串数组
    })
    .catch(console.error);
{% endhighlight %}

## 0x03 async 和 await
* 兑现 Promise 的值就像一个同步函数返回的值。而拒绝 Promise 就像一个同步函数抛出的值
* async 和 await 接收基于 Promise 的高效代码并隐藏 Promise，让你的代码像低效阻塞的同步代码一样容易理解

### 1 await 表达式
* await 关键字接收一个 Promise 并将其转换为`一个返回值或一个抛出的异常`
* 给定一个 Promise p，表达式 `await p` 会一直等到 p 落定，如果 p 兑现，那么 `await p` 的值就是兑现 p 的值。
如果 p 被拒绝，那么 `await p` 表达式就会抛出拒绝 p 的值
* 通常并不会使用 await 来接收一个保存 Promise 的变量，更多的是把它放在一个会返回 Promise 的函数调用前
1. let response = await fetch("/api/user/profile");
2. let profile = await response.json();
* await 并不会导致你的程序阻塞或在指定 Promise 落定前什么也不做，你的代码仍然是异步的，而 await 只是掩盖了这个事实，这意味着任何使用 await 的代码本身都是异步的

### 2 async 函数
* Note Note Note：`只能在以 async 关键字声明的函数内部使用 await 关键字`
* `把函数声明为 async 意味着该函数的返回值将是一个 Promise`。即使函数中不出现 Promise 相关代码
* 如果 async 函数会正常返回，那么作为该函数真正返回值的 Promise 对象将解决为整个显式的返回值
* 如果 async 函数会抛出异常，那么它返回的 Promise 对象将以该异常被拒绝
* `可以对任意函数使用 async 关键字`
{% highlight ruby %}
    async function getHighScore() {
        let response = await fetch("api/user/profile");
        let profile = await response.json();
        return profile.highScore;
    }

    // 在 async 函数内使用 await
    displayHighScore(await getHighScore());

    // 在非 async 函数内使用常规方式
    getHighScore().then(displayHighScore).catch(console.error);

{% endhighlight %}

### 3 等候多个 Promise
* 使用 async 重新了 getJSON 函数
{% highlight ruby %}
    async function getJSON(url) {
        let response = await fetch(url);
        let body = await response.json();
        return body;
    }
    // 这样写意味着必须等到抓取 url1 结果后，才会开始抓取 url2
    let value1 = await getJSON(url1);
    let value2 = await getJSON(url2);

    // 如果 url2 不依赖 url1，并行执行，同时抓取两个值
    let [val1, val2] = await Promise.all([getJSON(url1), getJSON(url2)]);
{% endhighlight %}

### 4 实现细节
* async 函数工作原理
{% highlight ruby %}
    async function f(x) { /* 函数体 */ }
    // 可以想象成一个返回 Promise 的包装函数
    function f(x) {
        return new Promise(function(resolve, reject) {
            try {
                resolve((function(x) {/* 函数体 */})(x));
            } catch(e) {
                reject(e);
            }
        })
    }
{% endhighlight %}
* await 可以想象成分隔代码体的记号，ES2017 解释器可以把函数体分割成一系列独立的子函数，每个子函数都将被传给位于前面以 await 标记的那个 Promise 的 then 方法

## 0x04 异步迭代
* Promise 只适合单次运行的异步计算，不适合与重复性异步事件来源一起使用，例如 setIntervnal()、浏览器的 click 事件或 Node 流的 dta 事件
* 所以 async 和 await 无法处理连续的异步事件；ES2018 产生了 `for/await` 循环，它是基于 Promise 的

### 1 for/await 循环
* 从一个流中读取连续的数据块
{% highlight ruby %}
const { fs } = require("fs");
    /*
       基于 Promise 的 for/await 循环 
       这里异步迭代器会产生一个 Promise，而 for/await 循环等待该 Promise 兑现
       将兑现值赋值给循环变量，然后在运行循环体
       之后再从头开始，从迭代器取得另一个 Promise 并等待这个新 Promise 兑现
    */
    async function parseFile(filename) {
        let stream = fs.createReadStream(filename, { encoding: "utf-8"});
        for await (let chunk of stream) {
            parseChunk(chunk); // 假设 parseChunk 是在其他地方定义的
        }
    }    

{% endhighlight %}

* 下边两种方式是等价的
* `for/await 也只能在 async 的函数内部使用`
{% highlight ruby %}
    const urls = [url1, url2, url3];
    const promises = urls.map(url => fetch(url));

    // 使用常规的 for/of 循环
    for(const promise of promises) {
        response = await promise;
        handle(response);
    }
    
    // 使用 for/await 循环
    for await (const response of promises) {
        handle(response);
    }
{% endhighlight %}

### 2 异步迭代器
#### 常规迭代器术语
* 可迭代对象是可以在 for/of 循环中使用的对象，它以一个符号名字 Symbol.iterator 定义了一个方法，该方法返回一个迭代器对象。
这个迭代器对象有一个 next() 方法，可以反复调用它获取可迭代对象的值，
* 迭代器对象的 next() 方法返回迭代结果对象，迭代结果对象有一个 value 属性或一个 done 属性

#### 异步迭代器
* 与常规迭代器十分类似，但有两个重要区别
1. 异步可迭代对象以符号名字 Symbol.asyncIterator 实现了一个方法。（for/await 与常规迭代器兼容，
但它更适合异步可迭代对象，因此会在 Symbol.iterator 前先尝试 Symbol.asyncIterator 方法）
2. 异步迭代器 next() 方法返回一个 Promise，解决为一个迭代器结果对象，而不直接返回一个迭代器结果对象

* 当我们对一个常规同步可迭代的 Promise 数组使用 for/await 时，操作是同步迭代器结果对象，其中 value 是一个 Promise 对象，但 done 属性是一个同步值。
* 真正的异步迭代器返回的是迭代结果对象的 Promise，其中 value 和 done 都是异步值。两者区别很微妙，对于异步迭代器，关于迭代何时结束的选择可以异步实现

### 3 异步生成器
* 实现迭代器最简单方式通常是使用生成器
* 使用声明为 async 的生成器函数来实现异步迭代器，声明为 async 的异步生成器同时有异步函数和生成器的特性。
即可以像常规异步函数中一样使用 await，也可以像常规生成器中一样使用 yield。但通过 yield 生成的值会自动包装到 Promise
* `async function *` 表示异步生成器
{% highlight ruby %}
    // 一个基于 Promise 包装 setTimeout 实现返回一个 Promise
    // 这个 Promise 在指定秒数后兑现
    function elapsedTime(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    // 一个异步迭代器函数，按照固定时间间隔
    // 递增并生成指定（或无穷）个计数器
    async function* clock(interval, max=Infinity) {
        for (let i = 1; i <= max; i++) {
            await elapsedTime(interval);  // 等待时间流逝
            yield i;      // 生成计数器
        }
    }

    // 使用 async 声明 以便里边可以使用 for/await
    async function test() {
        for await (let tick of clock(300, 100)) { // 循环 100 次，每次间隔 300ms
            console.log(tick);
        }
    }
{% endhighlight %}

### 4 实现异步迭代器
* 除了使用异生成器实现异步迭代器，还可以直接实现异步迭代器。这个需要定义个包含 Symbol.asyncIterator() 方法的对象，该方法要返回一个包含 next() 方法的对象，
而这个 next() 方法要返回解决为迭代器结果对象的 Promise。
{% highlight ruby %}
    function clock(interval, max=Infinity) {
        // 一个 setTimeout 的 Promise 版本
        // 注意参数是一个绝对时间而非时间间隔
        function until(time) {
            return new Promise(resolve => setTimeout(resolve, time - Date.now()));
        }

        // 返回一个异步可迭代对象
        return {
            startTime: Date.now(),  // 记住开始时间
            count: 1,               // 记住第几次迭代
            async next() {          // 方法使其为迭代器
                if (this.count > max) {   // 是否结束？
                    return { done: true }; // 表示结束的迭代结果
                }
                let targetTime = this.startTime + this.count * interval;
                // 等待该时间到来
                await until(targetTime);
                return { value: this.count++ };
            },
            [Symbol.asyncIterator]() { return this; }
        };
    }
{% endhighlight %}

#### AsyncQueue 例子
{% highlight ruby %}
    /*
        一个异步可迭代队列类，使用 enqueue() 添加值
        使用 dequeue() 移除值，dequeue() 返回一个 Promise
        这意味着，值可以在入队之前出队.
        这个类实现了 [Symbol.asyncIterator] 和 next(),
        因此可以与 for/await 循环一起使用

    */
    class AsyncQueue {
        constructor() {
            // 已经入队尚未出队的值保存在这里
            this.values = [];

            // 如果 Promise 出队时它们对应的值尚未入队
            // 就把那些 Promise 解决方法保存在这里
            this.resolves = [];

            // 一旦关闭，任何值都不能再入队，
            // 也不会再返回任何未兑现的 Promise
            this.closed = false;
        }

        enqueue(value) {
            if (this.closed) {
                throw new Error("AsyncQueue closed")
            }
            if (this.resolves.length > 0) {
                // 如果这个值已经有对应的 Promise 则解决该 Promise
                const resolve = this.resolves.shift();
                resolve(value);
            } else {
                // 否则，让它去排队
                this.values.push(value);
            }
        }

        dequeue() {
            if (this.values.length > 0) {
                // 如果有一个排队的值，为它返回一个解决 Promise
                const value = this.values.shift();
                return Promise.resolve(value);
            } else if (this.closed) {
                // 如果没有排队的值，并且队列已经关闭
                // 返回一个解决为 EOS(流终止) 标记的 Promise
                return Promise.resolve(AsyncQueue.EOS);
            } else {
                // 否则，返回一个未解决的 Promise
                // 将解决方法排队，以便后面使用
                return new Promise((resolve) => {
                    this.resolves.push(resolve);
                });
            }
        }

        close() {
            // 一旦关闭，任何值都不允许再入队
            // 因此以 EOS 标记解决所有待解决的 Promise
            while (this.resolves.length > 0) {
                this.resolves.shift()(AsyncQueue.EOS);
            }
            this.closed = true;
        }

        // 定义这个方法让这个类成为异步可迭代对象
        [Symbol.asyncIterator]() { return this; }

        // 定义这个方法让这个类成为异步迭代器
        // dequeue() 返回的 Promise 会解决为一个值，
        //或在关闭时解决为 EOS 标记，这里我们需要返回一个解决为
        // 迭代器结果对象的 Promise
        next() {
            return this.dequeue().then(value => (value === AsyncQueue.EOS)
            ? { value: undefined, done: true }
            : { value: value, done: false });
        }
    }
    // dequeue() 方法返回的标记值 在关闭时表示“终止流”
    AsyncQueue.EOS = Symbol("end-of-stream");
{% endhighlight %}

