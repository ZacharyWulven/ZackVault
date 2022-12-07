---
layout: post
title: Solidity-Extra
date: 2022-09-21 16:45:30.000000000 +09:00
tag: 区块链
---

## 1 定长字节数组
字节数组 bytes 分两种，一种定长（byte, bytes8, bytes32），另一种不定长。定长的属于数值类型，不定长的是引用类型。
定长 bytes 可以存一些数据，消耗 gas 比较少。
{% highlight ruby %}
固定长度的字节数组
bytes32public _byte32 ="MiniSolidity"; 
bytes1public _byte =_byte32[0]; 

MiniSolidity 变量以字节的方式存储进变量 _byte32，
转换成 16 进制为：0x4d696e69536f6c69646974790000000000000000000000000000000000000000
_byte 变量存储_byte32的第一个字节，为0x4d。
{% endhighlight %}

## 2 在以太坊中，以下语句被视为修改链上状态：
1. 写入状态变量
2. 释放事件
3. 创建其他合同
4. 使用 selfdestruct
5. 通过调用发送以太币
6. 调用任何未标记 view 或 pure 的函数
7. 使用低级调用（low-level calls）
8. 使用包含某些操作码的内联汇编

## 3 常用的全局变量
* blockhash(uint blockNumber): (bytes32)给定区块的哈希值 – 只适用于256最近区块, 不包含当前区块。
* block.coinbase: (address payable) 当前区块矿工的地址
* block.gaslimit: (uint) 当前区块的gaslimit
* block.number: (uint) 当前区块的number
* block.timestamp: (uint) 当前区块的时间戳，为unix纪元以来的秒
* gasleft(): (uint256) 剩余 gas
* msg.data: (bytes calldata) 完整call data
* msg.sender: (address payable) 消息发送者 (当前 caller)
* msg.sig: (bytes4) calldata的前四个字节 (function identifier)
* msg.value: (uint) 当前交易发送的wei值
* now : (uint) 当前块的时间戳

{% highlight ruby %}
https://learnblockchain.cn/docs/solidity/units-and-global-variables.html#special-variables-and-functions
{% endhighlight %}
