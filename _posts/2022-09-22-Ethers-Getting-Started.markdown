---
layout: post
title: Ethers-Getting-Started
date: 2022-09-22 16:45:30.000000000 +09:00
tag: 区块链
---

## 监听合约事件
这一讲，我们将介绍如何监听合约，并实现监听 USDT 合约的 Transfer 事件。

在ethersjs中，合约对象有一个contract.on的监听方法，让我们持续监听合约的事件：
{% highlight ruby %}
contract.on("eventName", function)
contract.on 有两个参数，一个是要监听的事件名称"eventName"，需要包含在合约abi中；另一个是我们在事件发生时调用的函数。
{% endhighlight %}

合约对象有一个contract.once的监听方法，让我们只监听一次合约释放事件，它的参数与contract.on一样：
{% highlight ruby %}
contract.once("eventName", function)
{% endhighlight %}

### 声明 provider
Alchemy 是一个免费的 ETH 节点提供商。需要先申请一个，后续会用到。
