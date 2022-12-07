---
layout: post
title: SolidityLang-02
date: 2022-09-24 16:45:30.000000000 +09:00
tag: solidity
---

## 0x01 SPDX 许可证标识符
由于提供源代码总是涉及版权方面的法律问题，Solidity 编译器鼓励使用机器可读的 SPDX 许可证标识符。每个源文件都应以说明其许可证的注释开头：

{% highlight ruby %}
// SPDX-License-Identifier: MIT
{% endhighlight %}

编译器不会验证许可证是否是 SPDX 允许的列表的一部分，但它会在字节码元数据中包含提供的字符串。
编译器在文件级别的文件中的任何位置都可以识别注释，但建议将其放在文件的顶部。

## 0x02 编译指示
pragma 关键字用于启用某些编译器功能或检查

{% highlight ruby %}
pragma solidity ^0.5.2;
{% endhighlight %}
* 带有上述行的源文件不能用 0.5.2 之前的编译器编译，它也不能在 0.6.0 开始的编译器上工作（这第二个条件是通过 using 添加的^）
* 使用版本编译指示不会更改编译器的版本。它也不会启用或禁用编译器的功能。
* 它只是指示编译器检查其版本是否与 pragma 所需的版本匹配。如果不匹配，编译器会发出错误。
* 从 Solidity 0.7.4 开始， ABI coder v2 不在作为实验特性，而是可以通过 pragma abicoder v2 启用。



## 0x03 导入其他源文件
{% highlight ruby %}
import "filename";
{% endhighlight %}

* 该 filename 部分称为导入路径。此语句将所有全局符号从“文件名”（以及从那里导入的符号）导入当前全局范围（与 ES6 不同，但向后兼容 Solidity）。
* 不推荐使用这种形式，因为它会意外地污染命名空间。如果您在“文件名”中添加新的顶级项目，它们会自动出现在所有从“文件名”导入的文件中。最好显式地导入特定符号

以下示例创建一个新的全局符号symbolName，其成员是来自 的所有全局符号"filename"：
{% highlight ruby %}
import * as symbolName from "filename";
{% endhighlight %}

这导致所有全局符号都以 symbolName.symbol. 这相当于 import * as symbolName from "filename";

{% highlight ruby %}
import "filename" as symbolName;
{% endhighlight %}

如果存在命名冲突，您可以在导入时重命名符号。例如，下面的代码分别创建了新的全局符号alias和从内部symbol2引用 symbol1和。symbol2"filename"
{% highlight ruby %}
import {symbol1 as alias, symbol2} from "filename";
{% endhighlight %}

## 0x04 注释
{% highlight ruby %}

// This is a single-line comment.

/*
This is a
multi-line comment.
*/
{% endhighlight %}

## 0x05 合约结构
* Solidity 中的合约类似于面向对象语言中的类。每个合约都可以包含状态变量、函数、函数修饰符、事件、错误、结构类型和枚举类型的声明。
* 此外，合约可以继承自其他合约。

### 状态变量
{% highlight ruby %}

// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;

contract SimpleStorage {
    uint storedData; // State variable
    // ...
}
{% endhighlight %}

### 函数
函数可以在合约内定义，也可以在合约外定义

### 函数修饰符 modifier
虽然声明在函数体外，但代码其实是放在函数最前边执行
{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.22 <0.9.0;

contract Purchase {
    address public seller;

    modifier onlySeller() { // Modifier
        require(
            msg.sender == seller,
            "Only seller can call this."
        );
        _;
    }

    function abort() public view onlySeller { // Modifier usage
        // ...
    }
}
{% endhighlight %}

### 事件
事件是与 EVM 日志记录工具的便利接口

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.21 <0.9.0;

contract SimpleAuction {
    event HighestBidIncreased(address bidder, uint amount); // Event

    function bid() public payable {
        // ...
        emit HighestBidIncreased(msg.sender, msg.value); // Triggering event
    }
}
{% endhighlight %}

### 错误
{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;

/// Not enough funds for transfer. Requested `requested`,
/// but only `available` available.
error NotEnoughFunds(uint requested, uint available);

contract Token {
    mapping(address => uint) balances;
    function transfer(address to, uint amount) public {
        uint balance = balances[msg.sender];
        if (balance < amount)
            revert NotEnoughFunds(amount, balance);
        balances[msg.sender] -= amount;
        balances[to] += amount;
        // ...
    }
}
{% endhighlight %}

### 结构体
{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;

contract Ballot {
    struct Voter { // Struct
        uint weight;
        bool voted;
        address delegate;
        uint vote;
    }
}
{% endhighlight %}

### 枚举
{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;

contract Purchase {
    enum State { Created, Locked, Inactive } // Enum
}
{% endhighlight %}

## 0x06 类型
Solidity 是一种静态类型语言，这意味着需要指定每个变量（状态和本地）的类型。Solidity 提供了几种基本类型，它们可以组合成复杂类型。

{% highlight ruby %}
Solidity 中不存在 “undefined”或“null” 的概念，但新声明的变量始终具有取决于其类型的默认值。
要处理任何意外值，您应该使用revert 函数来恢复整个事务，或者返回一个带有第二个bool值的元组，表示成功。
{% endhighlight %}

## 0x07 值类型
以下类型也称为值类型，因为这些类型的变量总是按值传递，即当它们用作函数参数或赋值时，它们总是被复制。
### 布尔值：true 或 false

* 运算符
1. \|\| 和 && 有短路规则 
2. This means that in the expression f(x) \|\| g(y), if f(x) evaluates to true, g(y) will not be evaluated even if it may have side-effects.

### Integers
1. uint8~256；int8~256
2. For an integer type X, you can use type(X).min and type(X).max 访问这个类型最大值和最小值.
3. Arithmetic operators: +, -, unary - (only for signed integers), *, /, % (模), ** (幂)

{% highlight ruby %}
Solidity中的整数是有取值范围的。 例如 uint32 类型的取值范围是 0 到 2 ** 32-1 。 
0.8.0 开始，算术运算有两个计算模式：一个是 “wrapping”（截断）模式或称 “unchecked”（不检查）模式，一个是”checked” （检查）模式。
默认情况下，算术运算在 “checked” 模式下，即都会进行溢出检查，如果结果落在取值范围之外，调用会通过 失败异常 回退。
你也可以通过 unchecked { ... } 切换到 “unchecked”模式，更多可参考 unchecked .
{% endhighlight %}

* 位操作
对数字的二进制补码表示执行位运算。这意味着，例如.~int256(0) == int256(-1)

* 移动操作
1. x << y 等价于数学表达式 x * 2**y.
2. x >> y 等价于数学表达式 x / 2\*\*y, 四舍五入到负无穷。.
3. 移位操作永远不会像算术运算那样执行溢出检查。相反，结果总是被截断。

* 加、减、乘
1. 如果有 int x = type(int).min;， 那 -x 将不在正数取值的范围内。 
这意味着这个检测 unchecked { assert(-x == x); } 是可以通过的（即这种情况下，不能假设它的负数会是正数），如果是 checked 模式，则会触发异常。

* 除法
1. 由于运算结果的类型始终是操作数之一的类型，因此整数除法始终产生整数。在 Solidity 中，除法向零舍入。这意味着 int256(-5) / int256(2) == int256(-2)
2. 除以0 会发生 Panic 错误 ， 而且这个检查，不可以通过 unchecked { ... } 禁用掉。

{% highlight ruby %}
表达式 type(int).min / (-1) 是仅有的整除会发生向上溢出的情况。 
在算术检查模式下，这会触发一个失败异常，在截断模式下，表达式的值将是 type(int).min 。
{% endhighlight %}

* 模
1. 模的结果与左操作数的符号相同
{% highlight ruby %}
int256(5) % int256(2) == int256(1)
int256(5) % int256(-2) == int256(1)
int256(-5) % int256(2) == int256(-1)
int256(-5) % int256(-2) == int256(-1)

模 0 会触发 Panic error. 不能通过禁用 unchecked { ... }.
{% endhighlight %}

* 幂
1. 求幂仅适用于指数中的无符号类型。求幂的结果类型始终等于基数的类型
2. 在检查模式下，求幂只对小基数使用相对便宜的exp操作码。对于 的情况x**3，表达式x*x*x可能更便宜。在任何情况下，gas 成本测试和优化器的使用都是可取的。
{% highlight ruby %}
Note that 0**0 is defined by the EVM as 1.
{% endhighlight %}

### Fixed Point Numbers 定长浮点型
Solidity 还没有完全支持定长浮点型。可以声明定长浮点型的变量，但不能给它们赋值或把它们赋值给其他变量。。
{% highlight ruby %}
fixed / ufixed：表示各种大小的有符号和无符号的定长浮点型。 
在关键字 ufixedMxN 和 fixedMxN 中，M 表示该类型占用的位数，N 表示可用的小数位数。 
M 必须能整除 8，即 8 到 256 位。 N 则可以是从 0 到 80 之间的任意数。 
ufixed 和 fixed 分别是 ufixed128x19 和 fixed128x19 的别名。
{% endhighlight %}

### Address
地址类型有两种风格，它们基本相同：
1. address ：保存一个 20 字节的值（以太坊地址大小）
2. address payable：与 address 相同，但具有 transfer 和 send 成员函数。

Note 类型转换
* 隐式从 address payable 转成 address 是允许的
* address 转为 address payable 需要显示使用 payable(<address>)；允许对 uint160，bytes20 合约类型转换
* 只有类型地址和合约类型的表达式可以通过显式转换 payable(<address>)
* 对于合约类型，仅当合约可以接收以太币时才允许这种转换，即合约具有接收或应付回退功能
* Note 请注意，payable(0) 是有效的，并且是该规则的一个例外

{% highlight ruby %}
如果将使用较大字节大小的类型转换为地址，例如 bytes32，则地址将被截断。
为了减少转换歧义，版本 0.4.24 及更高版本的编译器强制您在转换中显式截断。
以 32 字节值 0x111122223333444455556666777788889999AAAABBBBCCCCDDDDEEEEFFFFCCCC 为例。
可以使用address(uint160(bytes20(b)))，结果为0x111122223333444455556666777788889999aAaa，
也可以使用address(uint160(uint256(b)))，结果为0x777788889999AaAAbBbbCcccddDdeeeEfFFfc
{% endhighlight %}

{% highlight ruby %}
Note
address 与 address payable 的区别是 0.5.0 版引入的。
从该版本开始，合约不是从地址类型派生的，但如果它们具有 receive or payable fallback function，则仍可以显式转换为 address payable
{% endhighlight %}

### Address 成员
* balance：账户余额 
* transfer：转账

{% highlight ruby %}

如果当前合约的余额不足或者以太币转账被接收账户拒绝，transfer 功能就会失败。transfer 函数在失败时恢复

如果 x 是一个合约地址，它的代码（实现了 Receive Ether Function，或者实现了 Fallback Function）将与 transfer call 一起执行（这是 EVM 的一个特性，无法阻止）。如果该执行用完 gas 或以任何方式失败，则 Ether transfer 将被恢复，当前合约将异常停止
{% endhighlight %}

* send
{% highlight ruby %}
Send 是 transfer 相对低级的操作。如果执行失败，当前合约不会因为异常而停止，而是 send 会返回 false。
使用 send 存在一些危险：如果调用堆栈深度为 1024，则 transfer 失败（这总是由调用者强制执行的），
如果接收者耗尽 gas，transfer 也会失败。因此，为了进行安全的 Ether 转账，

Note：请始终检查 send 的返回值，使用 transfer 甚至更好（使用收款人取款的模式）
{% endhighlight %}

* call, delegatecall and staticcall
为了与不遵守 ABI 的合约交互，或者为了更直接地控制编码，提供了函数调用（call）、委托调用（delegatecall）和静态调用（staticcall）
。它们都采用单 bytes memory 并返回成功条件（作为布尔值）和返回的数据（bytes memory）。
函数 abi.encode、abi.encodePacked、abi.encodeWithSelector 和 abi.encodeWithSignature 可用于对结构化数据进行编码

Example
{% highlight ruby %}
bytes memory payload = abi.encodeWithSignature("register(string)", "MyName");
(bool success, bytes memory returnData) = address(nameReg).call(payload);
require(success);
{% endhighlight %}

此外，为了与不符合 应用二进制接口Application Binary Interface(ABI) 的合约交互，
于是就有了可以接受任意类型任意数量参数的 call 函数。 
这些参数会被打包到以 32 字节为单位的连续区域中存放。 
其中一个例外是当第一个参数被编码成正好 4 个字节的情况。 在这种情况下，这个参数后边不会填充后续参数编码，以允许使用函数签名。
{% highlight ruby %}

address nameReg = 0x72ba7d8e73fe8eb666ea66babc8116a41bfb10e2;
nameReg.call("register", "MyName");
nameReg.call(bytes4(keccak256("fun(uint256)")), a);
{% endhighlight %}


{% highlight ruby %}

所有这些函数都是低级函数，应该小心使用。具体来说，任何未知的合约都可能是恶意的，如果您调用它，您会将控制权移交给该合约，
该合约可能会反过来回调您的合约，因此请准备好在调用返回时更改您的状态变量。
与其他合约交互的常规方式是调用合约对象 (x.f()) 上的函数。

以前版本的 Solidity 允许这些函数接收任意参数，并且还会以不同的方式处理 bytes4 类型的第一个参数。这些 case 在 0.5.0 版中被删除
{% endhighlight %}

{% highlight ruby %}

可以使用气体调节器调整供应的气体：
address(nameReg).call{gas: 1000000}(abi.encodeWithSignature("register(string)", "MyName"));

同样，也可以控制提供的 Ether 值：
address(nameReg).call{value: 1 ether}(abi.encodeWithSignature("register(string)", "MyName"));

最后，可以组合这些修饰符。他们的顺序无关紧要：
address(nameReg).call{gas: 1000000, value: 1 ether}(abi.encodeWithSignature("register(string)", "MyName"));

{% endhighlight %}


* delegatecall
以类似的方式，可以使用函数 delegatecall：不同之处在于只使用给定地址的代码，
所有其他方面（存储，余额，等）都取自当前合约。 
delegatecall 的目的是使用存储在另一个合约中的库代码。用户必须确保两个合约中的存储布局都适合使用委托调用。

* staticcall
从以太坊拜占庭（byzantium）版本开始 提供了 staticcall ，它与 call 基本相同，但如果被调用的函数以任何方式修改状态变量，都将回退。

{% highlight ruby %}
Note：
所有三个函数 call、delegatecall 和 staticcall 都是非常低级的函数，只能作为最后的手段使用，因为它们破坏了 Solidity 的类型安全。
gas 选项适用于上边三种方法，而 value 选项仅在 call 中可用。

不管是读取状态还是写入状态，最好避免在合约代码中硬编码使用的 gas 值。这可能会引入”错误“，而且 gas 的消耗也是可能会改变的。
{% endhighlight %}

* code and codehash
您可以查询任何智能合约的部署代码。使用 .code 将 EVM 字节码作为字节内存获取，该内存可能为空。
使用 .codehash 获取该代码的 Keccak-256 哈希（作为 bytes32）。
{% highlight ruby %}
请注意，addr.codehash 比使用 keccak256(addr.code) 便宜。

所有合约都可以转换为 address 类型，因此可以使用 address(this).balance 查询当前合约的余额
{% endhighlight %}

### Contract Types
* 每个合约都定义了自己的类型。您可以将合约隐式转换为它们继承的合约类型。合同可以显式转换为 address 类型。
* 仅当合约类型具有 receive or payable fallback function 时，才能显式转换到地址应付类型。
转换仍然使用 address(x) 执行。如果合约 receive or payable fallback function，则可以使用 payable(address(x)) 转换为 address payable
* 在 0.5.0 版本之前，合约直接从 address 类型派生，没有 address 和 address payable 的区别。
* 您还可以实例化合约（即新创建一个合约对象），参考 ‘使用new创建合约’。
* 合约的数据表示与 address 类型的数据表示相同，并且这种类型也用于 ABI。
* 合约不支持任何运算符

合约类型的成员是合约的外部函数，包括任何标记为 public 的状态变量
对于合同 C，您可以使用 type(C) 访问有关合同的类型信息。


### 定长字节数组
* 值类型bytes1, bytes2, bytes3, ...bytes32 包含从 1 到最多 32 的字节序列。
* 该类型可以和作为右操作数的无符号整数类型进行移位运算（但返回结果的类型和左操作数类型相同），右操作数表示需要移动的位数。 进行有符号整数位移运算会引发运行时异常。
* .length 返回数组长度
* Note：bytes1[] 类型是一个字节数组，但是由于填充规则，它为每个元素浪费了 31 个字节的空间（存储空间除外）。最好使用 bytes 类型
* 在 0.8.0 版本之前，byte 曾经是 bytes1 的别名

### 动态大小的字节数组
* bytes：动态大小的字节数组，不是值类型
* string：动态大小的 UTF-8 编码字符串，不是值类型

### 地址字面量
通过地址校验和测试的十六进制文字，例如 0xdCad3a6d3569DF655070DEd06cb7A1b2Ccd1D3AF 属于地址类型。
长度在 39 到 41 位之间且未通过校验和测试的十六进制文字会产生错误。
您可以预先（对于整数类型）或附加（对于 bytesNN 类型）零来消除错误。
混合大小写地址校验和格式在 EIP-55 中定义

### 有理数和整数字面量
Solidity 中不存在八进制字面量。十进制小数字面常量带有一个 .，至少在后边有一个数字。 比如： .1， 和 1.3。

为了提高可读性可以在数字之间加上下划线。 例如，十进制 123_000，十六进制 0x2eff_abde，科学十进制表示 1_2e345_678都是有效的。 下划线仅允许在两位数之间，并且不允许下划线连续出现。添加到数字文字中下划线没有额外的语义，下划线会被编译器忽略。

数值字面常量表达式本身支持任意精度，直到被转换成了非常量类型（例如，在常量变量表达式之外有运算，或发生了显示转换）。 这意味着在数值常量表达式中, 计算不会溢出而除法也不会截断。
例如， (2**800 + 1) - 2**800 的结果是字面常量 1 （属于 uint8 类型），尽管计算的中间结果已经超过了 以太坊虚拟机Ethereum Virtual Machine(EVM) 的机器字长度。 此外， .5 * 8 的结果是整型 4 （尽管有非整型参与了计算）。


你可能认为像 255 + (true ? 1 : 0) 或 255 + [1, 2, 3][0] 这样的表达式等同于直接使用 256 字面常量。 但事实上，它们是在 uint8 类型中计算的，会溢出。

只要操作数是整型，任意整型支持的运算符都可以被运用在数值字面常量表达式中。 如果两个中的任一个数是小数，则不允许进行位运算。如果指数是小数的话，也不支持幂运算（因为这样可能会得到一个无理数）。

常量作为左（或基）操作数和整数类型的移位和幂运算时总是执行正确的（指数）操作，不管右（指数）操作数的类型如何。

* 不接收这样 uint128 a = 1; uint128 b = 2.5 + a + 0.5;

### 字符串字面量
* 字符串字面量用双引号或单引号
* "foo" "bar" 等于 to "foobar"，它们并不意味着 C 中的尾随零
* "foo" 代表三个字节，而不是四个
* 可以隐式转换为 bytes1, …, bytes32, 如果他们合适, to bytes and to string.
* 例如，对于 bytes32 samevar = "stringliteral"，当分配给 bytes32 类型时，字符串字面量会以其原始字节形式进行解释
* Until version 0.8.0 there were three additional escape sequences: \b, \f and \v. They are commonly available in other languages but rarely needed in practice. If you do need them, they can still be inserted via hexadecimal escapes, i.e. \x08, \x0c and \x0b, respectively, just as any other ASCII character.
* 以下示例中的字符串长度为 10 个字节。它以换行字节开头，后跟双引号，单引号和反斜杠字符，然后（不带分隔符）字符序列abcdef。
"\n\"\'\\abc\
def"
任何不是换行符的 Unicode 行终止符（即 LF、VF、FF、CR、NEL、LS、PS）都被视为终止字符串文字。换行符仅在字符串文字前面没有 . 时才终止\。

### unicode
* 常规字符串文字只能包含ASCII，而Unicode文字（以关键字unicode为前缀）可以包含任何有效的UTF-8序列。 它们还支持与转义序列完全相同的字符作为常规字符串文字。
* string memory a = unicode"Hello 😃";

### 十六进制
* 十六进制文字以 hex 关键字为前缀，并用双引号或单引号 ( hex"001122FF", hex'0011_22_FF') 括起来。它们的内容必须是十六进制数字。
* Multiple hexadecimal literals separated by whitespace are concatenated into a single literal: hex"00112233" hex"44556677" is equivalent to hex"0011223344556677"

### Note 枚举
* 枚举是在 Solidity 中创建用户定义类型的一种方式。
* 它们可以显式转换为所有整数类型，但不允许隐式转换。整数的显式转换在运行时检查该值是否在枚举范围内， 否则会导致 Panic 错误。
* Note：枚举至少需要一个成员，声明时它的默认值是第一个成员。枚举不能有超过 256 个成员。
* 数据表示与C中的枚举相同：选项从“0”开始的无符号整数值表示。
* 使用 type(NameOfEnum).min 和 type(NameOfEnum).max 你可以获得给定枚举的最小值和最大值。
* 枚举也可以在文件级别声明，在合约或库定义之外。
* Note 枚举默认是 uint8，超过会自动适配
* Note 枚举转 int，直接 uint8(ActionChoices.GoRight)
{% highlight ruby %}

// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.8;

contract test {
    enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill }
    ActionChoices choice;
    ActionChoices constant defaultChoice = ActionChoices.GoStraight;

    function setGoStraight() public {
        choice = ActionChoices.GoStraight;
    }

    // Since enum types are not part of the ABI, the signature of "getChoice"
    // will automatically be changed to "getChoice() returns (uint8)"
    // for all matters external to Solidity.
    function getChoice() public view returns (ActionChoices) {
        return choice;
    }

    function getDefaultChoice() public pure returns (uint) {
        return uint(defaultChoice);
    }

    function getLargestValue() public pure returns (ActionChoices) {
        return type(ActionChoices).max;
    }

    function getSmallestValue() public pure returns (ActionChoices) {
        return type(ActionChoices).min;
    }
}
{% endhighlight %}

### Note 用户自定义类型 alias
* 用户定义的值类型使用类型 C 定义为 V，其中 C 是新引入类型的名称，V 必须是内置值类型（“基础类型”）。
* 函数 C.wrap 用于从底层类型 -> 转换自定义类型
* 函数 C.unwrap 用于将自定义类型 -> 转换为底层类型
* C 类型没有任何运算符或绑定的成员函数。特别是，即使是运算符 == 也没有定义。不允许与其他类型进行显式和隐式转换。
这种类型的值的数据表示是从底层类型继承的，底层类型也在 ABI 中使用


下面的例子说明了一个自定义类型 UFixed256x18，代表了一个有18位小数的十进制定点类型，并有一个库来对该类型进行算术操作。
{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.8;

// Represent a 18 decimal, 256 bit wide fixed point type using a user defined value type.
type UFixed256x18 is uint256;

/// A minimal library to do fixed point operations on UFixed256x18.
library FixedMath {
    uint constant multiplier = 10**18;

    /// Adds two UFixed256x18 numbers. Reverts on overflow, relying on checked
    /// arithmetic on uint256.
    function add(UFixed256x18 a, UFixed256x18 b) internal pure returns (UFixed256x18) {
        return UFixed256x18.wrap(UFixed256x18.unwrap(a) + UFixed256x18.unwrap(b));
    }
    /// Multiplies UFixed256x18 and uint256. Reverts on overflow, relying on checked
    /// arithmetic on uint256.
    function mul(UFixed256x18 a, uint256 b) internal pure returns (UFixed256x18) {
        return UFixed256x18.wrap(UFixed256x18.unwrap(a) * b);
    }
    /// Take the floor of a UFixed256x18 number.
    /// @return the largest integer that does not exceed `a`.
    function floor(UFixed256x18 a) internal pure returns (uint256) {
        return UFixed256x18.unwrap(a) / multiplier;
    }
    /// Turns a uint256 into a UFixed256x18 of the same value.
    /// Reverts if the integer is too large.
    function toUFixed256x18(uint256 a) internal pure returns (UFixed256x18) {
        return UFixed256x18.wrap(a * multiplier);
    }
}
{% endhighlight %}
Note：UFixed256x18.wrap 和 FixedMath.toUFixed256x18 的签名相同，但执行的是两个完全不同的操作：``UFixed256x18.wrap 函数返回一个与输入的数据表示相同的 UFixed256x18， 而 toUFixed256x18``则返回一个具有相同数值的 ``UFixed256x18 。

### 函数类型
函数分为 internal 和 external 函数

internal 函数
* Note 内部函数只能在当前合约内部调用（更具体地说，在当前代码单元内部，还包括内部库函数和继承函数），
* 调用内部函数是通过跳转到它的入口标签来实现的，就像在内部调用当前合约的函数一样。

external 函数
* 外部函数由地址和函数签名组成，它们可以通过外部函数调用传递和返回。


函数定义格式
{% highlight ruby %}
function (<parameter types>) {internal|external} [pure|view|payable] [returns (<return types>)]
{% endhighlight %}

* 与参数类型相比，返回类型不能为空——如果函数类型不应返回任何内容，则 必须省略整个 returns 部分
* 默认情况下，函数类型是内部的，因此 internal 可以省略关键字


### 函数的转换
* 如果一个函数类型变量没有初始化就调用会宕机，如果使用 delete 后再调用函数也会发生宕机
* 如果在 Solidity 的上下文之外使用外部函数，它们将被视为 function 类型，它将地址和函数标识符一起编码为一个 bytes24 类型。
* 当前合约的 public 函数可以作为 internal 和 external 函数使用，使用 f 作为 internal 函数，使用 this.f 作为 external 函数
* 一个内部函数可以被分配给一个内部函数类型的变量，无论定义在哪里，包括合约和库的私有、内部和public函数，以及自有函数。 另一方面，外部函数类型只与public和外部合约函数兼容。库是不可以的，因为库使用 delegatecall，并且 他们的函数选择器有不同的 ABI 转换 。 接口中声明的函数没有定义，所以指向它们也没有意义。

* 当且仅当参数类型相同，返回类型相同，内外部属性相同并且 A 的状态可变性比 B 的状态可变性更具限制性时，函数 A 才能隐私转换为 B，就三种情况
1. pure 函数可转为 view 或 non-payable 函数
2. view 函数可转为 non-payable 函数
3. payable 函数可转为 non-payable 函数，反之不行



### 函数的成员
Note：external 或 public 函数有如下成员
1. .address 返回函数的合约的地址
2. .selector 返回 ABI 函数 selector

External (or public) 函数有额外成员 .gas(uint) 和 .value(uint) ），，在 0.6.2 版本后 deprecated，0.7.0 中去掉了，
用 {gas:...} 和 {value:...} 替代，用于指定发送到函数的  gas 数量或 wei 数量


### Example that shows how to use the members:

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.4 <0.9.0;

contract Example {
    function f() public payable returns (bytes4) {
        assert(this.f.address == address(this));
        return this.f.selector;
    }

    function g() public {
        this.f{gas: 10, value: 800}();
    }
}
{% endhighlight %}


### Example that shows how to use internal function types:

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.16 <0.9.0;

library ArrayUtils {
    // internal functions can be used in internal library functions because
    // they will be part of the same code context
    function map(uint[] memory self, function (uint) pure returns (uint) f)
        internal
        pure
        returns (uint[] memory r)
    {
        r = new uint[](self.length);
        for (uint i = 0; i < self.length; i++) {
            r[i] = f(self[i]);
        }
    }

    function reduce(
        uint[] memory self,
        function (uint, uint) pure returns (uint) f
    )
        internal
        pure
        returns (uint r)
    {
        r = self[0];
        for (uint i = 1; i < self.length; i++) {
            r = f(r, self[i]);
        }
    }

    function range(uint length) internal pure returns (uint[] memory r) {
        r = new uint[](length);
        for (uint i = 0; i < r.length; i++) {
            r[i] = i;
        }
    }
}


contract Pyramid {
    using ArrayUtils for *;

    function pyramid(uint l) public pure returns (uint) {
        return ArrayUtils.range(l).map(square).reduce(sum);
    }

    function square(uint x) internal pure returns (uint) {
        return x * x;
    }

    function sum(uint x, uint y) internal pure returns (uint) {
        return x + y;
    }
}
{% endhighlight %}

Another example that uses external function types:


{% highlight ruby %}

// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.22 <0.9.0;


contract Oracle {
    struct Request {
        bytes data;
        function(uint) external callback;
    }

    Request[] private requests;
    event NewRequest(uint);

    function query(bytes memory data, function(uint) external callback) public {
        requests.push(Request(data, callback));
        emit NewRequest(requests.length - 1);
    }

    function reply(uint requestID, uint response) public {
        // Here goes the check that the reply comes from a trusted source
        requests[requestID].callback(response);
    }
}


contract OracleUser {
    Oracle constant private ORACLE_CONST = Oracle(address(0x00000000219ab540356cBB839Cbe05303d7705Fa)); // known contract
    uint private exchangeRate;

    function buySomething() public {
        ORACLE_CONST.query("USD", this.oracleResponse);
    }

    function oracleResponse(uint response) public {
        require(
            msg.sender == address(ORACLE_CONST),
            "Only oracle can call this."
        );
        exchangeRate = response;
    }
}
{% endhighlight %}

## 0x08 引用类型
* 目前，引用类型包括结构、数组和映射
* 如果使用引用类型，则必须显式提供存储该类型的数据区域：
1. 内存（其生命周期仅限于外部函数调用）
2. 存储（存储状态变量的位置，生命周期所在的位置）仅限于合约的生命周期）
3. calldata（包含函数参数的特殊数据位置）

* 更改数据位置的赋值或类型转换将始终引发自动复制操作，而同一数据位置内的赋值仅在某些情况下复制存储类型

### 1 数据位置
* 每个引用类型都有一个附加注释，即“数据位置”，关于它的存储位置。共有三个数据位置：memory、storage 和 calldata。 calldata 是存储函数参数的不可修改、非持久性区域，其行为类似于内存。
* Note：如果可以，请尝试使用 calldata 作为数据位置，因为它可以避免复制，并且可以确保数据不能被修改。具有 calldata 数据位置的数组和结构也可以从函数返回，但不能分配此类类型
* Now memory and calldata 可用在所有函数，无论它们的可见性是什么

#### 数据位置与赋值行为
数据位置不仅与数据的持久性有关，还与分配的语义有关
1. 在 storage 或 memory (calldata) 直接赋值总是创建独立的拷贝
2. 从 memory 到 memory 赋值是引用传递
3. 从 storage 赋值给一个本地的 storage 也是引用传递
4. 从 storage 赋值给非 storage 总是 copy

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.5.0 <0.9.0;

contract C {
    // The data location of x is storage.
    // This is the only place where the
    // data location can be omitted.
    uint[] x;

    // The data location of memoryArray is memory.
    function f(uint[] memory memoryArray) public {
        x = memoryArray; // works, copies the whole array to storage
        uint[] storage y = x; // works, assigns a pointer, data location of y is storage
        y[7]; // fine, returns the 8th element
        y.pop(); // fine, modifies x through y
        delete x; // fine, clears the array, also modifies y
        // The following does not work; it would need to create a new temporary /
        // unnamed array in storage, but storage is "statically" allocated:
        // y = memoryArray;
        // Similarly, "delete y" is not valid, as assignments to local variables
        // referencing storage objects can only be made from existing storage objects.
        // It would "reset" the pointer, but there is no sensible location it could point to.
        // For more details see the documentation of the "delete" operator.
        // delete y;
        g(x); // calls g, handing over a reference to x
        h(x); // calls h and creates an independent, temporary copy in memory
    }

    function g(uint[] storage) internal pure {}
    function h(uint[] memory) public pure {}
}
{% endhighlight %}

### 2 Arrays
* 声明定长数组 T[k]
* 声明动态数组 T[]
* uint[][5] 表示一个包含 5 个 array 元素的数组，Note 语法与 C 是相反的
* mapping 只能存储在 storage 位置
* .push() 不传参数会新加一个 0 值元素

#### 2.1 bytes and string 作为数组
* `bytes` 和 `string` 类型的变量是特殊的数组。 `bytes` 类似于 `bytes1[]`，但它在 `calldata` 和 `memory` 中会被“紧打包”（将元素连续地存在一起，不会按每 32 字节一单元的方式来存放）。 `string` 与 `bytes` 相同，但不允许用长度或索引来访问。
* Solidity 没有字符串操作函数，但是可以使用第三方字符串库，可以比较两个字符串通过计算他们的 `keccak256-hash` ，可使用 `keccak256(abi.encodePacked(s1)) == keccak256(abi.encodePacked(s2))` 和使用 `string.concat(s1, s2)` 来拼接字符串

* 更多时候应该使用 bytes 而不是 bytes1[] ，因为Gas 费用更低, 在 内存 memory 中使用 bytes1[] 时，会在元素之间添加 31 个填充字节。 而在 storage 中，由于紧密包装，这没有填充字节
* 作为一个基本规则，对任意长度的原始字节数据使用 bytes，对任意长度字符串（UTF-8）数据使用 string 。
* 如果使用一个长度限制的字节数组，应该使用一个 bytes1 到 bytes32 的具体类型，因为它们便宜得多。

* Note：如果想要访问以字节表示的字符串 s，请使用 bytes(s).length / bytes(s)[7] = 'x';。 注意这时你访问的是 UTF-8 形式的低级 bytes 类型，而不是单个的字符。

#### 2.2 函数 bytes.concat 和 string.concat
* 可以使用 string.concat 连接任意数量的 string 字符串。该函数返回一个 string memory ，包含所有参数的内容，无填充方式拼接在一起
* bytes.concat 函数可以连接任意数量的 bytes 或 bytes1 ... bytes32 值。该函数返回一个 bytes memory ，包含所有参数的内容，无填充方式拼接在一起
* 如果你调用不使用参数调用 string.concat 或 bytes.concat 将返回空数组。

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.12;

contract C {
    string public s = "Storage";
    function f(bytes calldata bc, string memory sm, bytes16 b) public {
        string memory concatString = string.concat(s, string(bc), "Literal", sm);
        s = concatString;
        assert((bytes(s).length + bc.length + 7 + bytes(sm).length) == bytes(concatString).length);

        bytes memory concatBytes = bytes.concat(bytes(s), bc, bc[:2], "Literal", bytes(sm), b);
        assert((bytes(s).length + bc.length + 2 + 7 + bytes(sm).length + b.length) == concatBytes.length);
    }
}
{% endhighlight %}

#### 2.3 创建内存数组
* Note：使用 new 关键字创建 memory 的动态数组。与 storage 数组相反，你不能通过 .push 改变 memory 数组的大小。
{% highlight ruby %}
contract CC {
    function f(uint len) public pure {
        uint[] memory a = new uint[](7);
        bytes memory b = new bytes(len);
        assert(a.length == 7);
        assert(b.length == len);
        //a.push();  // error
        a[6] = 8;
    }
}
{% endhighlight %}

#### 2.4 数组字面量
* 它总是一个静态大小的内存数组，其长度为表达式的数量。
* 数组的基本类型是列表上的第一个表达式的类型，以便所有其他表达式可以隐式地转换为它。如果不可以转换，将出现类型错误。

{% highlight ruby %}
[1, 2, 3] 的类型是 uint8[3] memory

contract LBC {
    function f() public pure {
        g([uint(1), 2, 3]);
    }
    function g(uint[3] memory) public pure {
        // ...
    }
}
{% endhighlight %}

{% highlight ruby %}
contract C {
    function f() public pure returns (uint24[2][4] memory) {
        uint24[2][4] memory x = [[uint24(0x1), 1], [0xffffff, 2], [uint24(0xff), 3], [uint24(0xffff), 4]];
        // The following does not work, because some of the inner arrays are not of the right type.
        // 类型错误 因为第一个值的类型不匹配
        // uint[2][4] memory x = [[0x1, 1], [0xffffff, 2], [0xff, 3], [0xffff, 4]];
        return x;
    }
}
{% endhighlight %}

* 定长数组不能赋值给动态数组

{% highlight ruby %}
// This will not compile.
contract C {
    function f() public {
        // The next line creates a type error because uint[3] memory
        // cannot be converted to uint[] memory.
        uint[] memory x = [uint(1), 3, 4];
    }
}
{% endhighlight %}

#### 2.5 数组成员
* length：数组长度
* push()：动态 storage 数组 和 bytes (not string) 都有。它用来添加新的零初始化元素到数组末尾，并返回元素引用． 因此可以这样：　 x.push().t = 2 或 x.push() = b.
* push(x)：动态 storage 数组 和 bytes (not string) 都有。它用来添加新的元素到数组末尾，什么也不返回
* pop()：动态 storage 数组 和 bytes (not string) 都有。弹出数组最后一个元素，同时会在移除的元素上隐含调用 delete ，这个函数没有返回值

* 通过调用 push() 增加存储数组的长度具有恒定的 gas 成本，因为存储是零初始化的，而通过调用 pop() 减少长度的成本取决于被删除元素的“大小”。如果该元素是一个数组，它可能会非常昂贵
* 如果需要在外部（external）函数中使用多维数组，这需要启用ABI coder v2。 公有（public）函数中是支持的使用多维数组。
* 如果要调用返回动态数组的函数，请确保 EVM 在拜占庭分叉上运行。

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

contract ArrayContract {
    uint[2**20] aLotOfIntegers;
    /*

    注意下面不是一对动态数组而是一个动态对数组（即长度为 2 的固定大小数组）。
    在 Solidity 中，T[k] 和 T[] 总是包含 T 类型元素的数组，
    即使 T 本身是一个数组。
    因此，bool[2][] 是元素为 bool[2] 的动态数组，
    即 [] 是行，2 是列，与 C 语言是反着的
    */
    bool[2][] public pairsOfFlags;

    // newPairs is stored in memory - the only possibility
    // for public contract function arguments
    function setAllFlagPairs(bool[2][] memory newPairs) public {
        // assignment to a storage array performs a copy of ``newPairs`` and
        // replaces the complete array ``pairsOfFlags``.
        pairsOfFlags = newPairs;
    }

    struct StructType {
        uint[] contents;
        uint moreInfo;
    }
    StructType s;

    function f(uint[] memory c) public {
        // stores a reference to ``s`` in ``g``
        StructType storage g = s;
        // also changes ``s.moreInfo``.
        g.moreInfo = 2;
        // assigns a copy because ``g.contents``
        // is not a local variable, but a member of
        // a local variable.
        g.contents = c;
    }

    function setFlagPair(uint index, bool flagA, bool flagB) public {
        // access to a non-existing index will throw an exception
        // Note：赋值时候 与 C 类似
        pairsOfFlags[index][0] = flagA;
        pairsOfFlags[index][1] = flagB;
    }

    function changeFlagArraySize(uint newSize) public {
        // using push and pop is the only way to change the
        // length of an array
        if (newSize < pairsOfFlags.length) {
            while (pairsOfFlags.length > newSize)
                pairsOfFlags.pop();
        } else if (newSize > pairsOfFlags.length) {
            while (pairsOfFlags.length < newSize)
                pairsOfFlags.push();
        }
    }

    function clear() public {
        // these clear the arrays completely
        delete pairsOfFlags;
        delete aLotOfIntegers;
        // identical effect here
        pairsOfFlags = new bool[2][](0);
    }

    bytes byteData;

    function byteArrays(bytes memory data) public {
        // byte arrays ("bytes") are different as they are stored without padding,
        // but can be treated identical to "uint8[]"
        byteData = data;
        for (uint i = 0; i < 7; i++)
            byteData.push();
        byteData[3] = 0x08;
        delete byteData[2];
    }

    function addFlag(bool[2] memory flag) public returns (uint) {
        pairsOfFlags.push(flag);
        return pairsOfFlags.length;
    }

    function createMemoryArray(uint size) public pure returns (bytes memory) {
        // Dynamic memory arrays are created using `new`:
        uint[2][] memory arrayOfPairs = new uint[2][](size);

        // Inline arrays are always statically-sized and if you only
        // use literals, you have to provide at least one type.
        arrayOfPairs[0] = [uint(1), 2];

        // Create a dynamic byte array:
        bytes memory b = new bytes(200);
        for (uint i = 0; i < b.length; i++)
            b[i] = bytes1(uint8(i));
        return b;
    }
}
{% endhighlight %}

#### 2.6 数组元素的野指针
* 指一个指向不再存在的东西的引用，或者是对象被移除而没有更新引用。

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;

contract C {
    uint[][] s;

    function f() public {
        // Stores a pointer to the last array element of s.
        uint[] storage ptr = s[s.length - 1];
        // Removes the last array element of s.
        s.pop();
        // Writes to the array element that is no longer within the array.
        ptr.push(0x42);
        // 现在添加元素到 ``s`` 不会添加一个空元素, 而是数组长度为 1， ``0x42`` 作为其元素。
        s.push();
        assert(s[s.length - 1][0] == 0x42);
    }
}
{% endhighlight %}

ptr.push(0x42) 中的写入不会恢复，尽管 ptr 不再引用 s 的有效元素。
由于编译器假定未使用的存储始终为零，因此后续的 s.push() 不会将零显式写入存储，因此该 push() 之后的 s 的最后一个元素的长度为 1 并包含 0x42 作为其第一个元素。

Note：Solidity 不允许在存储中声明对值类型的引用。

这些明确的悬空引用被限制在嵌套引用类型中。然而，悬空引用也可能在元组赋值中使用复杂表达式时临时发生

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;

contract C {
    uint[] s;
    uint[] t;

    constructor() {
        // Push some initial values to the storage arrays.
        s.push(0x07);
        t.push(0x03);
    }

    function g() internal returns (uint[] storage) {
        s.pop();
        return t;
    }

    function f() public returns (uint[] memory) {
        // 下面会先执行 ``s.push()`` 获得到元素 1 的引用。
        // 然后, 调用 ``g`` pop 出这个新的引用，结果左边的元组元素成了悬空引用。
        // 赋值仍然会发生，会被写入 ``s`` 之外的数据区域。
        (s.push(), g()[0]) = (0x42, 0x17);
        // A subsequent push to ``s`` will reveal the value written by the previous
        // statement, i.e. the last element of ``s`` at the end of this function will have
        // the value ``0x42``.
        // 下面向 ``s`` push 使用上一个语句的值， ``s`` 的最后一个元素将是 ``0x42``

        s.push();
        return s;
    }
}
{% endhighlight %}

Note Note Note：每个语句只分配一次存储总是更安全，并避免在分配的左侧使用复杂的表达式。

{% highlight ruby %}
// 会有警告
contract C2 {
    bytes x = "012345678901234567890123456789";

    function test() external returns(uint) {
        (x.push(), x.push()) = (0x01, 0x02);
        return x.length;
    }
}

在处理对字节数组元素的引用时需要特别小心，因为字节数组上的 .push() 可能会在存储中从短布局切换到长布局
这里，当第一个 x.push() 被求值时，x 仍然存储在短布局中，因此 x.push() 返回对 x 的第一个存储槽中的元素的引用。
但是，第二个 x.push() 将字节数组切换到长布局。现在 x.push() 引用的元素在数组的数据区域中，而引用仍然指向其原始位置，
现在是长度字段的一部分，并且赋值将有效地混淆 x 的长度。

Note Note Note：为了安全起见，在单个赋值期间最多只能将字节数组扩大一个元素，并且不要在同一语句中同时对数组进行索引访问。
{% endhighlight %}


### 数组切片
* 数组切片是数组连续部分。它们写为 x[start:end]，其中 start 和 end 是导致 uint256 类型（或隐式转换为它）的表达式。
* 切片的第一个元素是 x[start]，最后一个元素是 x[end - 1]。
* 如果 start 大于 end 或者 end 大于数组的长度，则抛出异常。
* start 和 end 都是可选的：start 默认为 0，end 默认为数组的长度。
* 数组切片没有任何成员。它们可以隐式转换为其基础类型的数组并支持索引访问。索引访问在底层数组中不是绝对的，而是相对于切片的开头。
* 数组切片没有类型名称，这意味着没有变量可以将数组切片作为类型，它们只存在于中间表达式中
* 到目前为止，数组切片仅针对 calldata 数组实现
* 数组切片对于 ABI 解码函数参数中传递的辅助数据很有用

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.5 <0.9.0;
contract Proxy {
    /// @dev Address of the client contract managed by proxy i.e., this contract
    address client;

    constructor(address client_) {
        client = client_;
    }

    /// Forward call to "setOwner(address)" that is implemented by client
    /// after doing basic validation on the address argument.
    function forward(bytes calldata payload) external {
        bytes4 sig = bytes4(payload[:4]);
        // Due to truncating behaviour, bytes4(payload) performs identically.
        // Note：bytes4 sig = bytes4(payload);
        if (sig == bytes4(keccak256("setOwner(address)"))) {
            address owner = abi.decode(payload[4:], (address));
            require(owner != address(0), "Address of owner cannot be zero.");
        }
        (bool status,) = client.delegatecall(payload);
        require(status, "Forwarded call failed.");
    }
}
{% endhighlight %}
