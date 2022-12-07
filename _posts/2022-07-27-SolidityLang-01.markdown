---
layout: post
title: SolidityLang-01
date: 2022-07-27 16:45:30.000000000 +09:00
tag: solidity
---

## 0x01 A Simple Example
Solidity 是静态类型的，支持继承、库和复杂的用户定义类型等特性。所有标识符（合约名称、函数名称和变量名称）都仅限于 ASCII 字符集。可以将 UTF-8 编码的数据存储在字符串变量中。

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;

contract Coin {
    // The keyword "public" makes variables
    // accessible from other contracts
    address public minter;
    mapping (address => uint) public balances;

    // Events allow clients to react to specific
    // contract changes you declare
    event Sent(address from, address to, uint amount);

    // Constructor code is only run when the contract
    // is created
    constructor() {
        minter = msg.sender;
    }

    // Sends an amount of newly created coins to an address
    // Can only be called by the contract creator
    function mint(address receiver, uint amount) public {
        require(msg.sender == minter);
        balances[receiver] += amount;
    }

    // Errors allow you to provide information about
    // why an operation failed. They are returned
    // to the caller of the function.
    error InsufficientBalance(uint requested, uint available);

    // Sends an amount of existing coins
    // from any caller to an address
    function send(address receiver, uint amount) public {
        if (amount > balances[msg.sender])
            revert InsufficientBalance({
                requested: amount,
                available: balances[msg.sender]
            });

        balances[msg.sender] -= amount;
        balances[receiver] += amount;
        emit Sent(msg.sender, receiver, amount);
    }
}
{% endhighlight %}
* 第一行告诉您源代码是在 GPL 3.0 版下获得许可的。机器可读的许可证说明符在默认发布源代码的设置中很重要

* 关键字 Public 自动生成一个函数，允许您从合约外部访问状态变量的当前值。
如果没有这个关键字，其他合约就无法访问该变量。Public 类似 Getter

* send 会在最后一行发出一个 event（event Sent(address from, address to, uint amount)）。
以太坊客户端（例如 Web 应用程序）可以在无需太多成本的情况下侦听在区块链上发出的这些事件。一旦它发出，侦听器就会收到参数，这使得跟踪事务成为可能。
要监听此事件，您可以使用 web3.js 创建 Coin 合约对象，并且任何用户界面都会调用 balances 上面自动生成 balances 函数：
{% highlight ruby %}
Coin.Sent().watch({}, '', function(error, result) {
    if (!error) {
        console.log("Coin transfer: " + result.args.amount +
            " coins were sent from " + result.args.from +
            " to " + result.args.to + ".");
        console.log("Balances now:\n" +
            "Sender: " + Coin.balances.call(result.args.from) +
            "Receiver: " + Coin.balances.call(result.args.to));
    }
})
{% endhighlight %}

* 构造函数 constructor() 是一个特殊的函数，在合约创建时被调用，之后不能再调用。
* msg 变量（与 tx 和 block）都是特殊的全局变量，包含允许访问区块链的属性。msg.sender 始终是当前（外部）函数调用的来源地址。

* 错误允许您向调用者提供有关条件或操作失败原因的更多信息。Error 与 revert 语句一起使用 。
该 revert 语句无条件中止并恢复与该函数的所有更改，类似 require，但它还允许您提供错误的名称和将提供给调用者（并最终提供给前端应用程序或块资源管理器）的附加数据，以便故障可以更容易地被调试或响应。


## 0x02 以太坊虚拟机
以太坊出块一般 17 一个。以太坊虚拟机或 EVM 是以太坊中智能合约的运行时环境。

它不仅是沙盒，而且实际上是完全隔离的，这意味着在 EVM 内运行的代码无法访问网络、文件系统或其他进程。智能合约甚至对其他智能合约的访问权限有限。

### 账户
* 以太坊中有两种账户共享相同的地址空间：由公私钥对（即人类）控制的外部账户和由与账户一起存储的代码控制的合约账户
* 外部账户的地址由公钥确定，而合约的地址在合约创建时确定（它来源于创建者地址和从该地址发送的交易数量，即所谓的“随机数”）。
* 无论账户是否存储代码，EVM 都会平等对待这两种类型。

* 1 ether == 10 \*\* 18 wei

### 交易
* 交易是从一个帐户发送到另一个帐户的消息（可能相同或为空，见下文）。它可以包括二进制数据（称为“有效负载”）和以太币。
* 如果目标帐户包含代码，则执行该代码并将有效负载作为输入数据提供。
* 如果没有设置目标账户（交易没有收款人或收款人设置为null），则交易创建一个新合约。如前所述，该合约的地址不是零地址，而是从发送者及其发送的交易数量（“nonce”）派生的地址。这种合约创建交易的有效负载被视为 EVM 字节码并被执行。此执行的输出数据作为合约代码永久存储。这意味着，为了创建合约，您无需发送合约的实际代码，而是发送实际执行时返回该代码的代码。
* 在创建合约时，其代码仍然是空的。因此，在构造函数完成执行之前，您不应该回调正在构建的合约。

### Gas
* gas 费支付方是 tx.origin
* asset 会用光 gas，require 会当前执行多少就用多少 gas
* gas 机制为矿工和质押者的工作报酬
* 每一个块都有最大 gas 数量，这样限制了需要验证块的工作数量
* gas 价格是交易发起者设置的（gas_price * gas）用于支付给 EVM 执行者；如果交易正常执行没有用完的 gas 会返还给发起者；
如果发生 revert 异常，已用完的 gas 将不予退还
* EVM 执行者可选择是否包含一个交易，交易发起者不能通过设置低 gas 滥用系统

### Storage, Memory and the Stack
以太坊虚拟机有三个区域存储数据：
1. storage 
* 每个账户有一个数据存储区域叫 storage，它在函数调用和交易之间是持久性的
* storage 是一个 map， 256-bit words to 256-bit words
* 读取成本较高，初始化和修改成本更高，因此应将存储在 storage 内容最小化，在合约之外存储衍生计算、缓存和聚合数据
* 只能读写自己的 storage： 合约既不能读取也不能写入除自己的 storage 之外的任何 storage

2. memory
* 第二个存储区域为内存，其中合约为每个消息调用获取一个新的实例
* 内存是线性的，可以在字节基本寻址，但读取的宽度限制为 256 位，而写入宽度可以是 8 或 256 位的。
* 当访问（读或写）以前未触及的内存字（即字内任何偏移量）时，内存会扩展一个字（256 位）。扩张时，必须支付 gas 费，内存越大，费用越高。

3. stack
* EVM 不是寄存器机，而是堆栈机，所以所有计算都在称为堆栈的数据区域上执行。
* 它的最大大小为 1024 个元素并包含 256 位的字。
* 对堆栈的访问仅限于顶端：可以将最顶端的 16 个元素之一复制到堆栈的顶部，或者将最顶端的元素与它下面的 16 个元素之一交换。所有其他操作从堆栈中获取最顶部的两个（或一个或多个，取决于操作）元素并将结果压入堆栈。
* 当然，可以将堆栈元素移动到存储或内存中，以便更深入地访问堆栈，但在不首先移除堆栈顶部的情况下，不可能仅访问堆栈中更深的任意元素。

### 指令系统
EVM 的指令集保持最小化，以避免可能导致共识问题的不正确或不一致的实现。所有指令都对基本数据类型、256 位字或内存片（或其他字节数组）进行操作。存在通常的算术、位、逻辑和比较操作。有条件的和无条件的跳转是可能的。此外，合约可以访问当前区块的相关属性，例如其编号和时间戳

### Message Call
合约可以通过消息调用的方式调用其他合约或向非合约账户发送以太币。消息调用类似于交易，因为它们具有源、目标、数据有效负载、以太币、gas 和返回数据。实际上，每个交易都包含一个顶级消息调用，而该调用又可以创建进一步的消息调用。
合约可以决定应该通过内部消息调用发送多少剩余的gas，以及它想要保留多少。如果在内部调用（或任何其他异常）中发生气体不足异常，则会通过放入堆栈的错误值发出信号。在这种情况下，只有与调用一起发送的 gas 被用完。在 Solidity 中，调用合约在这种情况下默认会导致手动异常，因此异常会“冒泡”调用堆栈。 
调用被限制在 1024 的深度，这意味着对于更复杂的操作，循环应该优先于递归调用。此外，在消息调用中只能转发 63/64 的气体，这导致实际深度限制略小于 1000

### Delegatecall / Callcode and Libraries
存在一个消息调用的特殊变体，称为委托调用， 它与消息调用相同，除了目标地址处的代码在调用合约的上下文（即地址处）中执行msg.sender并且msg.value不会更改之外他们的价值观。
这意味着合约可以在运行时从不同的地址动态加载代码。存储、当前地址和余额仍然是调用合约，只是代码取自被调用地址。
这使得在 Solidity 中实现“库”特性成为可能：可重用的库代码，可以应用于合约的存储，例如，为了实现复杂的数据结构。

### Logs
可以将数据存储在一个特殊索引的数据结构中，该结构一直映射到块级别。Solidity 使用这个称为日志 的特性来实现事件。合约创建后无法访问日志数据，但可以从区块链外部有效访问。由于日志数据的一部分存储在布隆过滤器中，因此可以以一种有效且加密安全的方式搜索这些数据，因此不下载整个区块链的网络对等方（所谓的“轻客户端”）仍然可以找到这些日志

### Create
合约甚至可以使用特殊的操作码创建其他合约（即它们不像交易那样简单地调用零地址）。这些创建调用和普通消息调用之间的唯一区别是“执行有效负载数据并将结果存储为代码，调用者/创建者接收堆栈上新合约的地址”。

### Deactivate and Self-destruct
从区块链中删除代码的唯一方法是在该地址的合约执行selfdestruct操作时。存储在该地址的剩余以太币被发送到指定目标，然后从状态中删除存储和代码。理论上移除合约听起来是个好主意，但它有潜在的危险，就像有人将以太币发送给移除的合约一样，以太币将永远丢失。
{% highlight ruby %}
即使合约通过 selfdestruct 被删除 ，它仍然是区块链历史的一部分，并且可能被大多数以太坊节点保留。所以使用 selfdestruct 并不等同于从硬盘中删除数据。

即使合约的代码不包含 selfdestruct 调用，它仍然可以使用 delegatecallor 或 callcode 执行该操作。

如果你想停用你的合约，你应该通过改变一些导致所有功能恢复的内部状态来禁用它们。这使得无法使用合约，因为它会立即返回 Ether。
{% endhighlight %}


### 预编译合约
有一小部分特别的合约，地址为 1~8 包含“预编译合约”，可以像任何其他合约一样调用，但它们的行为（以及它们的 gas 消耗）不是由存储在的 EVM 代码定义的该地址（它们不包含代码）而是在 EVM 执行环境本身中实现。
不同的 EVM 兼容链可能使用不同的预编译合约集。未来也有可能将新的预编译合约添加到以太坊主链中，但您可以合理地期望它们始终在 1和0xffff（包括）之间的范围内。

## 0x03 Voting 合约
{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;
/// @title Voting with delegation.
contract Ballot {
    // This declares a new complex type which will
    // be used for variables later.
    // It will represent a single voter.
    struct Voter {
        uint weight; // weight is accumulated by delegation
        bool voted;  // if true, that person already voted
        address delegate; // person delegated to
        uint vote;   // index of the voted proposal
    }

    // This is a type for a single proposal.
    struct Proposal {
        bytes32 name;   // short name (up to 32 bytes)
        uint voteCount; // number of accumulated votes
    }

    address public chairperson;

    // This declares a state variable that
    // stores a `Voter` struct for each possible address.
    mapping(address => Voter) public voters;

    // A dynamically-sized array of `Proposal` structs.
    Proposal[] public proposals;

    /// Create a new ballot to choose one of `proposalNames`.
    constructor(bytes32[] memory proposalNames) {
        chairperson = msg.sender;
        voters[chairperson].weight = 1;

        // For each of the provided proposal names,
        // create a new proposal object and add it
        // to the end of the array.
        for (uint i = 0; i < proposalNames.length; i++) {
            // `Proposal({...})` creates a temporary
            // Proposal object and `proposals.push(...)`
            // appends it to the end of `proposals`.
            proposals.push(Proposal({
                name: proposalNames[i],
                voteCount: 0
            }));
        }
    }

    // Give `voter` the right to vote on this ballot.
    // May only be called by `chairperson`.
    function giveRightToVote(address voter) external {
        // If the first argument of `require` evaluates
        // to `false`, execution terminates and all
        // changes to the state and to Ether balances
        // are reverted.
        // This used to consume all gas in old EVM versions, but
        // not anymore.
        // It is often a good idea to use `require` to check if
        // functions are called correctly.
        // As a second argument, you can also provide an
        // explanation about what went wrong.
        require(
            msg.sender == chairperson,
            "Only chairperson can give right to vote."
        );
        require(
            !voters[voter].voted,
            "The voter already voted."
        );
        require(voters[voter].weight == 0);
        voters[voter].weight = 1;
    }

    /// Delegate your vote to the voter `to`.
    function delegate(address to) external {
        // assigns reference
        Voter storage sender = voters[msg.sender];
        require(sender.weight != 0, "You have no right to vote");
        require(!sender.voted, "You already voted.");

        require(to != msg.sender, "Self-delegation is disallowed.");

        // Forward the delegation as long as
        // `to` also delegated.
        // In general, such loops are very dangerous,
        // because if they run too long, they might
        // need more gas than is available in a block.
        // In this case, the delegation will not be executed,
        // but in other situations, such loops might
        // cause a contract to get "stuck" completely.
        while (voters[to].delegate != address(0)) {
            to = voters[to].delegate;

            // We found a loop in the delegation, not allowed.
            require(to != msg.sender, "Found loop in delegation.");
        }

        Voter storage delegate_ = voters[to];

        // Voters cannot delegate to accounts that cannot vote.
        require(delegate_.weight >= 1);

        // Since `sender` is a reference, this
        // modifies `voters[msg.sender]`.
        sender.voted = true;
        sender.delegate = to;

        if (delegate_.voted) {
            // If the delegate already voted,
            // directly add to the number of votes
            proposals[delegate_.vote].voteCount += sender.weight;
        } else {
            // If the delegate did not vote yet,
            // add to her weight.
            delegate_.weight += sender.weight;
        }
    }

    /// Give your vote (including votes delegated to you)
    /// to proposal `proposals[proposal].name`.
    function vote(uint proposal) external {
        Voter storage sender = voters[msg.sender];
        require(sender.weight != 0, "Has no right to vote");
        require(!sender.voted, "Already voted.");
        sender.voted = true;
        sender.vote = proposal;

        // If `proposal` is out of the range of the array,
        // this will throw automatically and revert all
        // changes.
        proposals[proposal].voteCount += sender.weight;
    }

    /// @dev Computes the winning proposal taking all
    /// previous votes into account.
    function winningProposal() public view
            returns (uint winningProposal_)
    {
        uint winningVoteCount = 0;
        for (uint p = 0; p < proposals.length; p++) {
            if (proposals[p].voteCount > winningVoteCount) {
                winningVoteCount = proposals[p].voteCount;
                winningProposal_ = p;
            }
        }
    }

    // Calls winningProposal() function to get the index
    // of the winner contained in the proposals array and then
    // returns the name of the winner
    function winnerName() external view
            returns (bytes32 winnerName_)
    {
        winnerName_ = proposals[winningProposal()].name;
    }
}

{% endhighlight %}

## 0x04 拍卖合约
{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;
contract SimpleAuction {
    // Parameters of the auction. Times are either
    // absolute unix timestamps (seconds since 1970-01-01)
    // or time periods in seconds.
    address payable public beneficiary;
    uint public auctionEndTime;

    // Current state of the auction.
    address public highestBidder;
    uint public highestBid;

    // Allowed withdrawals of previous bids
    mapping(address => uint) pendingReturns;

    // Set to true at the end, disallows any change.
    // By default initialized to `false`.
    bool ended;

    // Events that will be emitted on changes.
    event HighestBidIncreased(address bidder, uint amount);
    event AuctionEnded(address winner, uint amount);

    // Errors that describe failures.

    // The triple-slash comments are so-called natspec
    // comments. They will be shown when the user
    // is asked to confirm a transaction or
    // when an error is displayed.

    /// The auction has already ended.
    error AuctionAlreadyEnded();
    /// There is already a higher or equal bid.
    error BidNotHighEnough(uint highestBid);
    /// The auction has not ended yet.
    error AuctionNotYetEnded();
    /// The function auctionEnd has already been called.
    error AuctionEndAlreadyCalled();

    /// Create a simple auction with `biddingTime`
    /// seconds bidding time on behalf of the
    /// beneficiary address `beneficiaryAddress`.
    constructor(
        uint biddingTime,
        address payable beneficiaryAddress
    ) {
        beneficiary = beneficiaryAddress;
        auctionEndTime = block.timestamp + biddingTime;
    }

    /// Bid on the auction with the value sent
    /// together with this transaction.
    /// The value will only be refunded if the
    /// auction is not won.
    function bid() external payable {
        // No arguments are necessary, all
        // information is already part of
        // the transaction. The keyword payable
        // is required for the function to
        // be able to receive Ether.

        // Revert the call if the bidding
        // period is over.
        if (block.timestamp > auctionEndTime)
            revert AuctionAlreadyEnded();

        // If the bid is not higher, send the
        // money back (the revert statement
        // will revert all changes in this
        // function execution including
        // it having received the money).
        if (msg.value <= highestBid)
            revert BidNotHighEnough(highestBid);

        if (highestBid != 0) {
            // Sending back the money by simply using
            // highestBidder.send(highestBid) is a security risk
            // because it could execute an untrusted contract.
            // It is always safer to let the recipients
            // withdraw their money themselves.
            pendingReturns[highestBidder] += highestBid;
        }
        highestBidder = msg.sender;
        highestBid = msg.value;
        emit HighestBidIncreased(msg.sender, msg.value);
    }

    /// Withdraw a bid that was overbid.
    function withdraw() external returns (bool) {
        uint amount = pendingReturns[msg.sender];
        if (amount > 0) {
            // It is important to set this to zero because the recipient
            // can call this function again as part of the receiving call
            // before `send` returns.
            pendingReturns[msg.sender] = 0;

            // msg.sender is not of type `address payable` and must be
            // explicitly converted using `payable(msg.sender)` in order
            // use the member function `send()`.
            if (!payable(msg.sender).send(amount)) {
                // No need to call throw here, just reset the amount owing
                pendingReturns[msg.sender] = amount;
                return false;
            }
        }
        return true;
    }

    /// End the auction and send the highest bid
    /// to the beneficiary.
    function auctionEnd() external {
        // It is a good guideline to structure functions that interact
        // with other contracts (i.e. they call functions or send Ether)
        // into three phases:
        // 1. checking conditions
        // 2. performing actions (potentially changing conditions)
        // 3. interacting with other contracts
        // If these phases are mixed up, the other contract could call
        // back into the current contract and modify the state or cause
        // effects (ether payout) to be performed multiple times.
        // If functions called internally include interaction with external
        // contracts, they also have to be considered interaction with
        // external contracts.

        // 1. Conditions
        if (block.timestamp < auctionEndTime)
            revert AuctionNotYetEnded();
        if (ended)
            revert AuctionEndAlreadyCalled();

        // 2. Effects
        ended = true;
        emit AuctionEnded(highestBidder, highestBid);

        // 3. Interaction
        beneficiary.transfer(highestBid);
    }
}

payable 关键字使能够接收 Ether。
{% endhighlight %}

## 0x05 盲拍合约
{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;
contract BlindAuction {
    struct Bid {
        bytes32 blindedBid;
        uint deposit;
    }

    address payable public beneficiary;
    uint public biddingEnd;
    uint public revealEnd;
    bool public ended;

    mapping(address => Bid[]) public bids;

    address public highestBidder;
    uint public highestBid;

    // Allowed withdrawals of previous bids
    mapping(address => uint) pendingReturns;

    event AuctionEnded(address winner, uint highestBid);

    // Errors that describe failures.

    /// The function has been called too early.
    /// Try again at `time`.
    error TooEarly(uint time);
    /// The function has been called too late.
    /// It cannot be called after `time`.
    error TooLate(uint time);
    /// The function auctionEnd has already been called.
    error AuctionEndAlreadyCalled();

    // Modifiers are a convenient way to validate inputs to
    // functions. `onlyBefore` is applied to `bid` below:
    // The new function body is the modifier's body where
    // `_` is replaced by the old function body.
    modifier onlyBefore(uint time) {
        if (block.timestamp >= time) revert TooLate(time);
        _;
    }
    modifier onlyAfter(uint time) {
        if (block.timestamp <= time) revert TooEarly(time);
        _;
    }

    constructor(
        uint biddingTime,
        uint revealTime,
        address payable beneficiaryAddress
    ) {
        beneficiary = beneficiaryAddress;
        biddingEnd = block.timestamp + biddingTime;
        revealEnd = biddingEnd + revealTime;
    }

    /// Place a blinded bid with `blindedBid` =
    /// keccak256(abi.encodePacked(value, fake, secret)).
    /// The sent ether is only refunded if the bid is correctly
    /// revealed in the revealing phase. The bid is valid if the
    /// ether sent together with the bid is at least "value" and
    /// "fake" is not true. Setting "fake" to true and sending
    /// not the exact amount are ways to hide the real bid but
    /// still make the required deposit. The same address can
    /// place multiple bids.
    function bid(bytes32 blindedBid)
        external
        payable
        onlyBefore(biddingEnd)
    {
        bids[msg.sender].push(Bid({
            blindedBid: blindedBid,
            deposit: msg.value
        }));
    }

    /// Reveal your blinded bids. You will get a refund for all
    /// correctly blinded invalid bids and for all bids except for
    /// the totally highest.
    function reveal(
        uint[] calldata values,
        bool[] calldata fakes,
        bytes32[] calldata secrets
    )
        external
        onlyAfter(biddingEnd)
        onlyBefore(revealEnd)
    {
        uint length = bids[msg.sender].length;
        require(values.length == length);
        require(fakes.length == length);
        require(secrets.length == length);

        uint refund;
        for (uint i = 0; i < length; i++) {
            Bid storage bidToCheck = bids[msg.sender][i];
            (uint value, bool fake, bytes32 secret) =
                    (values[i], fakes[i], secrets[i]);
            if (bidToCheck.blindedBid != keccak256(abi.encodePacked(value, fake, secret))) {
                // Bid was not actually revealed.
                // Do not refund deposit.
                continue;
            }
            refund += bidToCheck.deposit;
            if (!fake && bidToCheck.deposit >= value) {
                if (placeBid(msg.sender, value))
                    refund -= value;
            }
            // Make it impossible for the sender to re-claim
            // the same deposit.
            bidToCheck.blindedBid = bytes32(0);
        }
        payable(msg.sender).transfer(refund);
    }

    /// Withdraw a bid that was overbid.
    function withdraw() external {
        uint amount = pendingReturns[msg.sender];
        if (amount > 0) {
            // It is important to set this to zero because the recipient
            // can call this function again as part of the receiving call
            // before `transfer` returns (see the remark above about
            // conditions -> effects -> interaction).
            pendingReturns[msg.sender] = 0;

            payable(msg.sender).transfer(amount);
        }
    }

    /// End the auction and send the highest bid
    /// to the beneficiary.
    function auctionEnd()
        external
        onlyAfter(revealEnd)
    {
        if (ended) revert AuctionEndAlreadyCalled();
        emit AuctionEnded(highestBidder, highestBid);
        ended = true;
        beneficiary.transfer(highestBid);
    }

    // This is an "internal" function which means that it
    // can only be called from the contract itself (or from
    // derived contracts).
    function placeBid(address bidder, uint value) internal
            returns (bool success)
    {
        if (value <= highestBid) {
            return false;
        }
        if (highestBidder != address(0)) {
            // Refund the previously highest bidder.
            pendingReturns[highestBidder] += highestBid;
        }
        highestBid = value;
        highestBidder = bidder;
        return true;
    }
}

internal 关键字只能本合约或派生合约调用

calldata 关键字
对于函数中的数组参数，建议使用 calldata 而不是 memory，
因为 calldata 避免了不必要的 copy，这可以显着节省 gas
{% endhighlight %}

## 0x06 远程购买合约
* 如何在合约中使用类似状态机的构造
* Tips：构造时候，Payable value 确保金额是偶数

{% highlight ruby %}
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;
contract Purchase {
    uint public value;
    address payable public seller;
    address payable public buyer;

    enum State { Created, Locked, Release, Inactive }
    // The state variable has a default value of the first member, `State.created`
    State public state;

    modifier condition(bool condition_) {
        require(condition_);
        _;
    }

    /// Only the buyer can call this function.
    error OnlyBuyer();
    /// Only the seller can call this function.
    error OnlySeller();
    /// The function cannot be called at the current state.
    error InvalidState();
    /// The provided value has to be even.
    error ValueNotEven();

    modifier onlyBuyer() {
        if (msg.sender != buyer)
            revert OnlyBuyer();
        _;
    }

    modifier onlySeller() {
        if (msg.sender != seller)
            revert OnlySeller();
        _;
    }

    modifier inState(State state_) {
        if (state != state_)
            revert InvalidState();
        _;
    }

    event Aborted();
    event PurchaseConfirmed();
    event ItemReceived();
    event SellerRefunded();

    // Ensure that `msg.value` is an even number.
    // Division will truncate if it is an odd number.
    // Check via multiplication that it wasn't an odd number.
    constructor() payable {
        seller = payable(msg.sender);
        value = msg.value / 2;
        if ((2 * value) != msg.value)
            revert ValueNotEven();
    }

    /// Abort the purchase and reclaim the ether.
    /// Can only be called by the seller before
    /// the contract is locked.
    function abort()
        external
        onlySeller
        inState(State.Created)
    {
        emit Aborted();
        state = State.Inactive;
        // We use transfer here directly. It is
        // reentrancy-safe, because it is the
        // last call in this function and we
        // already changed the state.
        seller.transfer(address(this).balance);
    }

    /// Confirm the purchase as buyer.
    /// Transaction has to include `2 * value` ether.
    /// The ether will be locked until confirmReceived
    /// is called.
    function confirmPurchase()
        external
        inState(State.Created)
        condition(msg.value == (2 * value))
        payable
    {
        emit PurchaseConfirmed();
        buyer = payable(msg.sender);
        state = State.Locked;
    }

    /// Confirm that you (the buyer) received the item.
    /// This will release the locked ether.
    function confirmReceived()
        external
        onlyBuyer
        inState(State.Locked)
    {
        emit ItemReceived();
        // It is important to change the state first because
        // otherwise, the contracts called using `send` below
        // can call in again here.
        state = State.Release;

        buyer.transfer(value);
    }

    /// This function refunds the seller, i.e.
    /// pays back the locked funds of the seller.
    function refundSeller()
        external
        onlySeller
        inState(State.Release)
    {
        emit SellerRefunded();
        // It is important to change the state first because
        // otherwise, the contracts called using `send` below
        // can call in again here.
        state = State.Inactive;

        seller.transfer(3 * value);
    }
}
{% endhighlight %}
