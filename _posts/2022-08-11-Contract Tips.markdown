---
layout: post
title: Contract Tips
date: 2022-08-11 16:45:30.000000000 +09:00
tag: 区块链
---

本文旨在总结一些学习到的合约开发注意点，努力打造安全的合约开发心得

## 0x01 智能合约思想
1. 先判断条件
2. 改变条件
3. 再跟合约交互

比如先清零在执行交易

## 0x02 尽量使用 transfer or send
好处是给的汽油费很低，不足以让调用的地址再改其他事，最多也就是打 个 log

![](/assets/images/eth/contract4.png)

## 0x03 尽量使用 msg.sender 而不是 tx.origin
因为 tx.origin 为原始调用者，可能会给黑客以有利于的机会

## 0x04 保持小心
因为区块链上其他合约可能都是有恶意的，所以要想到你调用的合约的函数是否会反向调用你的合约并修改状态。

## 0x05 溢出问题
* 在进行数学运算时候一定要考虑溢出的可能性。Solidity 有一个 SafeMath 库叫 OpenZeppelin 会检测有没有溢出
* https://github.com/OpenZeppelin/openzeppelin-contracts
* 0.8.0 加入了 SafeMath

## 0x06 偶数检测
* Payable value 确保金额是偶数
{% highlight ruby %}
// Check via multiplication that it wasn't an odd number.
constructor() payable {
  seller = payable(msg.sender);
  value = msg.value / 2;
  if ((2 * value) != msg.value)
    revert ValueNotEven();
  }
}
{% endhighlight %}

## 0x07 乘除都有时候，应该总是先乘后除
{% highlight ruby %}
    function getConversionRate(uint256 ethAmount) public view returns (uint256) {
        uint256 ethPrice = getPrice();
        // 应该总是先乘后除
        uint256 ethAmountInUSD = (ethPrice * ethAmount) / 1e18;
        return ethAmountInUSD;
    }
{% endhighlight %}

