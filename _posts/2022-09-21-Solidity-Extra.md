---
layout: post
title: Solidity 补充
date: 2022-09-21 16:45:30.000000000 +09:00
categories: [区块链, Ethereum, Solidity]
tags: [区块链, Solidity, Ethereum] 
---

## 1 定长字节数组
字节数组 bytes 分两种，一种定长（byte, bytes8, bytes32），另一种不定长。定长的属于数值类型，不定长的是引用类型。
定长 bytes 可以存一些数据，消耗 gas 比较少。

```
固定长度的字节数组
bytes32public _byte32 ="MiniSolidity"; 
bytes1public _byte =_byte32[0]; 

MiniSolidity 变量以字节的方式存储进变量 _byte32，
转换成 16 进制为：0x4d696e69536f6c69646974790000000000000000000000000000000000000000
_byte 变量存储_byte32的第一个字节，为0x4d。
```

> bytes 比较特殊是数组，但是不用加 []。
另外，不能用 byte[] 声明单字节数组，可以使用 bytes 或 bytes1[]。
在 gas 费上，bytes 比 bytes1[] 便宜。因为 bytes1[] 在 memory 中要增加 31 个字节进行填充，会产生额外的 gas。但是在 storage 中，由于内存紧密打包，不存在字节填充。
{: .prompt-info }


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

```
https://learnblockchain.cn/docs/solidity/units-and-global-variables.html#special-variables-and-functions
```

## 4 Mapping
1. Mapping 使用 keccak256(key) 当成 offset 存取 value。
2. 因为 Ethereum 会定义所有未使用的空间为 0，所以未赋值（Value）的键（Key）初始值都是各个 type 的默认值，如 uint 的默认值是 0。

## 5 Event
### topics
每条日志记录都包含主题 topics 和数据 data 两部分。用于描述事件，长度不能超过 4 个元素。
它的第一个元素是事件的签名（哈希）。对于 ERC20 的 Transfer 事件，它的签名就是：`keccak256("Transfer(addrses,address,uint256)")`。
其他为 indexed 参数，事件中 indexed 参数最多可以有 3 个。indexed 参数可以直接检索。每个 indexed 参数的大小为固定的 256 比特，如果参数太大了（比如字符串），就会自动计算哈希存储在主题中。

### data
事件中不带 indexed 的参数会被存储在 data 部分中，data 部分的变量不能被直接检索，但可以存储任意大小的数据。
因此一般 data 部分可以用来存储复杂的数据结构，例如数组和字符串等，因为这些数据超过了 256 比特，即使存储在事件的 topic 部分中，也是以哈希的方式存储。
另外，data 部分的变量在存储上消耗的 gas 相比于 topic 更少


