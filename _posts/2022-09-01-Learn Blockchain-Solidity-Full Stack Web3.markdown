---
layout: post
title: Learn Blockchain-Solidity-Full Stack Web3
date: 2022-09-01 16:45:30.000000000 +09:00
tag: 区块链
---

## 0x01 基本介绍

### 预言机

预言机是任何可以向区块链输入数据或进行链下计算的设备。我们需要一个去中心化的预言机网络，正如去中心化的区块链网络一样。

将链上去中心化逻辑和链下去中心化数据和计算相结合，这叫混合型智能合约，比如 chainlink。

chainlink 是一个组件化，去中心化的预言机网络，它不仅可以给智能合约提供外部数据，还可以提供链下计算。
chainlink 可以让我们获取数据，合约自动化执行，随机数，以任何有意义的方式自定义你的智能合约

### 水管

水管连接 https://faucets.chain.link，或者使用 MetaMask 的水管 

参考 https://github.com/smartcontractkit/full-blockchain-solidity-course-js 里边推荐的测试网络

### Gas 介绍
* Gas Price：是交易中每个执行单元的花费，价格越高被写入链的几率越高

* Gas Limit & Usage by Txn: 60,000 | 21,000 (35%)
1. Gas Limit 是上限（这里是 60,000），Usage by Txn 是使用了的量（这里是 21,000）
2. Transaction Fee = Gas Price * Usage by Txn

* 在 etherscan.io 上查看交易

https://goerli.etherscan.io/tx/0x98233d8f3d15086318ef423dd2f2ef986074bf9c655e22a14f502aa101bfe913

### 区块链运作机制

{% highlight ruby %}
https://andersbrownworth.com/blockchain/blockchain
{% endhighlight %}

以太坊使用哈希函数 Keccak256

### 签名交易
比特币和以太坊都是 ECDSA（它是 DSA 电子签名算法的一种），可以根据私钥创建出公钥

### Gas-区块奖励 & EIP1559
* 在 etherscan 上，Transaction Fee = (区块基础费 + 最大优先费) * gas 使用数量

* 根据 EIP-1559 以太坊每一笔交易都有 base fee
{% highlight ruby %}
https://eth-converter.com/
{% endhighlight %}

* Max Fee：指我们愿意为这个交易付的最高 gas price，它会比 Gas Price 多一点
* Max Priority Fee：它是 Max gas fee 加上我们愿意支付给矿工的小费
* Base Fee 会被 burnt。Burnt 值 = Base Fee * Usage by Txn。这意味着每次交易，一部分以太币会永远不再流通，或说被烧掉。
现在以太坊上，你的交易手续费的一部分被烧掉了，另一部分直接给矿工

* 给矿工的钱 = Transaction Fee - Burnt Fee
* Others 里显示 Txn Type: 2 (EIP-1559) 表示这个交易版本是 EIP-1559
* Base Fee 会根据有多少正在发送交易而调节，如果有很多人想写入区块，那么就有大量的 gas 会被烧掉
* Base Fee 调整是期望每个区块被写入 50%，如果超过 50% Base Fee 就会上涨，反之下降
* Txn Savings Fees：是我们实际支付费用和费用上限之间的差额
* Nonce: 表示是这个账户第几个交易，从 0 开始

### EIP 1559


### 区块链概念
* 中本聪共识：包括 PoW 和最长合法链规则
* 矿工：在 PoW 中叫矿工，在 PoS 中叫验证者
* 区块链要有抗女巫攻击的能力，比特币通过 PoW 避免，而以太坊 PoS 通过质押

### 分片（Sharding）
* 因为区块的存储空间有限，节点存储交易也有限，所以 gas price 就会上涨，这时有了分片
* 分片：一个可扩展性的解决方案，指多个区块链的区块链，有一个主链会协调一些不同的链，将它们连接在一起
* 这意味着人们可以在多个链上发送交易，很有效率的提供了区块链的空间，分片可以极大的增加在 Layer1 上发送的交易数量

### Layer 1 & Layer 2
* Layer 1 是区块链实现的基础层，比特币，以太坊， avalanche 都是  Layer 1
* Layer 2 是加在 Layer 1 和区块链上的任何应用，比 Chainlink，Arbitrum，Optimism。
* Arbitrum，Optimism 也是 Rollup，把它们自己的交易集中起来，然后写入以太坊这样的 Layer 1，
* Rollup 有点像一个分片的链（Sharding chain），它继承了以太坊这样的基础链也就是 Layer 1 的安全性，它们把交易发给 Layer 1 ，解决扩展性问题，
解决方式是做为一条链，人们在它上面发交易就相当于在以太坊上发交易，
* Layer 2 与侧链不同，因为侧链的安全性来自于自身的协议，而 Rollup 的安全性来自于基础层，所以 Arbitrum，Optimism 和以太坊一样安全

### 小结
* Sharding 和 Rollup 是 Layer 1 的可扩展性解决方案
* 可扩展性问题是因为区块空间不足导致写入的交易有限

{% highlight ruby %}
教程 https://github.com/smartcontractkit/full-blockchain-solidity-course-js
{% endhighlight %}


## 0x02 Solidity

### 版本
* pragma solidity >=0.8.0 <0.9.0; 任何大于等于 0.8.0 小于 0.9.0 版本都适用
* pragma solidity ^0.8.0 <0.9.0; 至少是 0.8.0 最新版本 不能超过 0.8

### License
* // SPDX-License-Identifier: MIT
* MIT 是限制最少的 License 之一，

### 数据类型
* bytes 是一种更底层的数据结构
* 声明有默认值 uint256 默认初始化为 0
* 方法默认是 internal，即只对本合约和继承合约可见
* 状态变量加 public 就会自动创建 getter 函数

{% highlight ruby %}
1 Solidity 有两个关键字，标识函数调用不需要消耗 gas，它们是 view 和 pure
2 调用 view 和 pure 是免费的，除非你在消耗 gas 的函数中调用它们
3 只有修改状态的时候才需要支付 gas，发交易
{% endhighlight %}

* 合约第一个状态变量是第一个插槽，索引是 0

### Memory & Storage & Calldata
EVM can access and store information in six places

1. Stack
2. Memory
3. Storage
4. Calldata
5. Code
6. Logs

{% highlight ruby %}
Calldata 和 Memory 表示是临时的，变量只存在于函数过程中
Calldata 是不可以修改的临时变量，如果一个函数参数没有修改可以用 Calldata
Memory 是可以修改的临时变量
{% endhighlight %}

### 部署合约
在 Remix 选择 Environment 为 Injected Provider-MetaMask
{% highlight ruby %}
https://docs.chain.link/docs/link-token-contracts/
{% endhighlight %}

在 Remix 的 Compile 中选 Compilation Details 可以查看 ABI

### StorageFactory
* 通过工厂合约实现与其他合约交互
* 合约引入
* 继承和 override
{% highlight ruby %}
// I'm a comment!
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;
// pragma solidity ^0.8.0;
// pragma solidity >=0.8.0 <0.9.0;

contract SimpleStorage {

    uint256 favoriteNumber;

    struct People {
        uint256 favoriteNumber;
        string name;
    }
    // uint256[] public anArray;
    People[] public people;

    mapping(string => uint256) public nameToFavoriteNumber;

    function store(uint256 _favoriteNumber) public virtual {
        favoriteNumber = _favoriteNumber;
    }
    
    function retrieve() public view returns (uint256){
        return favoriteNumber;
    }

    function addPerson(string memory _name, uint256 _favoriteNumber) public {
        people.push(People(_favoriteNumber, _name));
        nameToFavoriteNumber[_name] = _favoriteNumber;
    }
}


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./SimpleStorage.sol";

// 继承使用 is
contract ExtraStorage is SimpleStorage {

    function store(uint256 _favoriteNumber) public virtual override {
        favoriteNumber = _favoriteNumber + 5;
    }

}


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./SimpleStorage.sol";

contract StorageFactory {

    SimpleStorage[] public simpleStorageArray;
    function createSimpleStorageContract() public {
        SimpleStorage simpleStorage = new SimpleStorage();
        simpleStorageArray.push(simpleStorage);

    }
    
    function sfStore(uint256 _simpleStoreageIndex, uint256 _simpleStorageNumber) public {

        // Address
        // ABI
       // SimpleStorage simpleStorage = SimpleStorage(simpleStorageArray[_simpleStoreageIndex]);
       SimpleStorage simpleStorage = simpleStorageArray[_simpleStoreageIndex];
       simpleStorage.store(_simpleStorageNumber);
    }

    function sfGet(uint256 _simpleStoreageIndex) public view returns (uint256) {
        SimpleStorage simpleStorage = simpleStorageArray[_simpleStoreageIndex];
        return simpleStorage.retrieve();
    }

}

{% endhighlight %}

### Transactions Fields

{% highlight ruby %}
Note
1. Nonce: tx count for the account
2. Gas Price: price per unit of gas (in wei)
3. Gas Limit: max gas that this tx can use
4. To: address that th tx is sent to
5. Value: amount of wei to send
6. Data: what to send to the To Address
7. v,r,s: components of tx signature
{% endhighlight %}

### Fund Me 合约
{% highlight ruby %}

function fund() public payable  {
    // 1e18 = 1* 10 ** 18
    // 要求金额必须大于 1 eth
    require(msg.value > 1e18, "need more than 1 eth");
}
{% endhighlight %}

## 0x03 chainlink
 
### 1 获取 ETH 对美元价格
通过 chainlink 获取链外数据
{% highlight ruby %}
https://data.chain.link/ethereum/mainnet/crypto-usd/eth-usd


https://docs.chain.link/docs/get-the-latest-price/
获取 ETH-USD 合约 PriceConsumerV3，通过 MetaMask 部署合约
调用 getLatestPrice 获取 USD 价格，获得数会比较大，小数部分有 8 位，例如 $1263.70000000
https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol

https://docs.chain.link/docs/ethereum-addresses/
其他数据合约地址
{% endhighlight %}

### 2 chainlink VRF 
* chainlink VRF 连接可验证的随机功能
* 区块链是一个确定性的系统，这意味着它们不能有随机数，所以需要通过查看链外数据获取随机数
{% highlight ruby %}
如何获得可证明的随机数
https://docs.chain.link/docs/vrf/v2/examples/get-a-random-number/
{% endhighlight %}

### 3 chainlink Keepers
* 解决每 10 分钟或一周 do something，或某个币涨到多少 do something
* 你注册事件，然后 chainlink 节点会不断监听让这些事件触发

{% highlight ruby %}
https://docs.chain.link/docs/chainlink-keepers/compatible-contracts/
{% endhighlight %}

### 4 Make a Get Request
chainlink 发送 http 请求

{% highlight ruby %}
https://docs.chain.link/docs/any-api/get-request/examples/single-word-response/

APIConsumer 合约地址
https://remix.ethereum.org/#url=https://docs.chain.link/samples/APIRequests/APIConsumer.sol&optimize=false&runs=200&evmVersion=null&version=soljson-v0.8.7+commit.e28d00a7.js
首先发送 requestVolumeData 请求
第二步 fulfill 保存数据
{% endhighlight %}

## 0x04 FundMe
* 当我们需要与合约交互时候需要 abi 和 address

{% highlight ruby %}
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// import from gihub or use npm
// https://www.npmjs.com/package/@chainlink/contracts
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

import "./PriceConverter.sol";

// get fund from users
// withdraw funds
// set a minimum funding value in USD

error NotOwner();

contract FundMe {
    using PriceConverter for uint256;

    // 50 / $1270 = 0.04 最少要 0.04
    // constant 更节省 gas
    // 常量要大写
    uint256 public constant MINIMUM_USD = 10 * 1e18;

    address[] public funders;
    mapping(address => uint256) public addressToAmountFunded;

    // immutable 更节省 gas
    // constant 和 immutable 省油的原因是直接写入 bytecode
    address public immutable i_owner;
    // constant immutable 不存储在状态变量里，而跟合约代码一起

    modifier onlyOwner() {
        //require(msg.sender == owner, "only owner can do it");
        if (msg.sender != i_owner) {
            // 这样更省 gas 因为不需要字符串
            revert NotOwner();
        }
        _;
    }

    constructor() {
        i_owner = msg.sender;
    }

    function fund() public payable {
        // 1e18 = 1* 10 ** 18
        // 要求金额必须大于 1 eth
        //require(msg.value > 1e18, "need more than 1 eth");
        //require(getConversionRate(msg.value) >= minimumUSD, "balance not enough");
        require(
            msg.value.getConversionRate() >= MINIMUM_USD,
            "balance not enough"
        );
        funders.push(msg.sender);
        addressToAmountFunded[msg.sender] += msg.value;
    }

    function withdraw() public payable onlyOwner {
        for (uint256 i = 0; i < funders.length; ++i) {
            address funder = funders[i];
            addressToAmountFunded[funder] = 0;
        }
        // reset the array
        // 一个新的数组，没有任何元素
        funders = new address[](0);

        // transfer
        payable(msg.sender).transfer(address(this).balance);

        // send
        bool sendSuccess = payable(msg.sender).send(address(this).balance);
        require(sendSuccess, "send failed");

        // call
        // 如果调用 call 返回数据 则保存在 bytes callData 中
        // 不调用任何方法 就 ("")
        // (bool callSuccess, bytes memory callData) =
        (bool callSuccess, ) = payable(msg.sender).call{
            value: address(this).balance
        }("");
        require(callSuccess, "call failed");
    }

    // 如果用户没有使用 fund
    // 那么内部调用实现 fund
    receive() external payable {
        fund();
    }

    fallback() external payable {
        fund();
    }

    // to library
    // function getPrice() public view returns (uint256) {
    //     // https://docs.chain.link/docs/ethereum-addresses/
    //     // 0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e
    //     //AggregatorV3Interface(0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e)
    //     AggregatorV3Interface priceFeed = AggregatorV3Interface(0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e);

    //     // ETH in terms of USD
    //     // price 有多少为是小数，可以调用 AggregatorV3Interface 的 decimals 方法查看
    //     // 返回乘以 1*10**8 所以最后乘以 1e10，才有正确的单位
    //     (, int256 price,,,) = priceFeed.latestRoundData();

    //     return uint256(price * 1e10); // 1**10
    // }

    // function getVersion() public view returns (uint256) {
    //    AggregatorV3Interface priceFeed = AggregatorV3Interface(0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e);
    //    return priceFeed.version();
    // }

    // function getConversionRate(uint256 ethAmount) public view returns (uint256) {
    //     uint256 ethPrice = getPrice();
    //     // 应该总是先乘后除
    //     uint256 ethAmountInUSD = (ethPrice * ethAmount) / 1e18;
    //     return ethAmountInUSD;
    // }
}


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

library PriceConverter {
    function getPrice() internal  view returns (uint256) {
        // https://docs.chain.link/docs/ethereum-addresses/
        // 0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e
        //AggregatorV3Interface(0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e)
        AggregatorV3Interface priceFeed = AggregatorV3Interface(0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e);

        // ETH in terms of USD
        // price 有多少为是小数，可以调用 AggregatorV3Interface 的 decimals 方法查看
        // 返回乘以 1*10**8 所以最后乘以 1e10，才有正确的单位
        (, int256 price,,,) = priceFeed.latestRoundData();
    
        return uint256(price * 1e10); // 1**10 
    }

    function getVersion() internal view returns (uint256) {
       AggregatorV3Interface priceFeed = AggregatorV3Interface(0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e);
       return priceFeed.version();
    }

    // 第一个参数就是调用者
    function getConversionRate(uint256 ethAmount) internal view returns (uint256) {
        uint256 ethPrice = getPrice();
        // 应该总是先乘后除
        uint256 ethAmountInUSD = (ethPrice * ethAmount) / 1e18;
        return ethAmountInUSD;
    }
}

{% endhighlight %}


使用 unchecked 更节省 gas
{% highlight ruby %}
contract SafeMathTester {

    uint8 public bigNumber = 255;
    // 使用 unchecked 更节省 gas
    function add() public {
        unchecked { bigNumber = bigNumber + 1;}
    }
}
{% endhighlight %}

### 如何查找问题的解决方案
1. Stack Overflow
2. github
3. reddit
4. dc

## 0x04 本地环境安装
### 题外话
* vs code 可安装 remote development
* gitpod 是一个云开发环境，可以在远程服务器运行您的代码
* typescript 是 javascript 的安全版本

### 一、vs code 配置
* 1 vs code solidity 高亮，搜索 solidity + hardhat 安装

* 2 vs code 搜索安装 Prettier。可以在 vs code 中搜 `defaultformat` 设置成 Prettier

* 3 vs code 快捷键：command + shift + p 打开命令面板
{% highlight ruby %}
1. 输入 setting 选择 worksapce JSON settings，打开 worksapce JSON settings
2. 编辑添加 
{
    "[solidity]": {
        "editor.defaultFormatter": "NomicFoundation.hardhat-solidity"
    }, 
    "[javascript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    }, 
}
{% endhighlight %}

#### 二、js 异步编程
{% highlight ruby %}
// Setup Movie Night

// Cook popcorn
// Pour Drinks
// Start Movie
/*
    没有 async 说明代码是同步的 比如 solidity 是同步编程的

    async 说明是异步编程

    sync 返回一个 promise，promise 值可能是 
    Pending
    Fulfilled
    Rejected

 */
async function setupMovieNight() {
  /*
    await 关键字告诉任何基于 Promise 的函数，等待
    Promise 结果为 Fulfilled 或 Rejected
    await 只能用于 async 函数
   */
  await cookPopcorn();
  await pourDrinks();
  // 等待 cookPopcorn 和 pourDrinks 结果，方可 startMovie
  startMovie();
}

// 基于 Promise 的函数
function cookPopcorn() {
  return Promise();
}

function pourDrinks() {
  return Promise();
}

function startMovie() {
  return Promise();
}

{% endhighlight %}

### 三、use solc-js
https://github.com/ethereum/solc-js

{% highlight ruby %}
$ node --version
$ npm --version
$ corepack --version

安装 yarn
旧版本使用 npm 安装， 现在可以使用 yarn
$ npm i -g corepack 
$ npm i -g yarn 安装 yarn （过时了）

1 应该
$ corepack enable 
$ yarn --version
->1.22.19
之后通过 yarn 安装其他

2 安装 solc
$ yarn add solc
issue：如果出现 package-lock.json found. your project contains lock files generated by tools other than yarn，
需要删除工程目录或父级目录的 yarn.lock 文件 再 yarn add solc 进行安装
issue link：https://stackoverflow.com/questions/45690202/how-to-have-yarn-not-issue-a-warning-for-the-license-field

安装成功后，工程目录应该多了 package.json 和 yarn.lock 和 node_modules 
package.json：它告诉我们关于项目的信息和依赖项目，solidity 版本 0.8.17
"dependencies": {
    "solc": "^0.8.17"
}

yarn.lock：说明一些库的版本（类似 podfile.lock）

node_modules：是节点模块文件夹

如果想按照特定版本 solidity 可以，$ yarn add solc@0.8.7-fixed 安装成功后 package.json 的依赖版本变为 0.8.7-fixed
可以查看 https://github.com/ethereum/solc-js 的 release 查看版本


Tips：或全局使用 0.8.7-fixed 版本
$ yarn global add solc@0.8.7-fixed
但工程目录有优先使用 package.json 的 solc 版本
{% endhighlight %}


### 四、solcjs 编译
{% highlight ruby %}
$ yarn solcjs --help
$ yarn solcjs --version

编译
$ yarn solcjs --bin --abi --include-path node_modules --base-path . -o . SimpleStorage.sol
--bin 最终获得 bin
--abi 最终获得 abi
. 表示当前目录
-o . 表示输出到当前目录
SimpleStorage.sol 表示要编译的文件


后续我们多次编译 SimpleStorage.sol 就要输入大堆命令，我们可以做一个 script 简化这不
操作如下：编辑 package.json，添加 scripts 字段如下
{
  "dependencies": {
    "solc": "^0.8.17"
  },
  "scripts": {
    "compile": "yarn solcjs --bin --abi --include-path node_modules --base-path . -o . 
    SimpleStorage.sol"
  }
}
然后在终端输入
$ yarn compile 即可编译 SimpleStorage.sol，这样编译跟 Remix 中编译一样

{% endhighlight %}

### 五、ganache
ganache 类似于 Remix 中的虚拟机，相当于一个假的区块链测试网环境

{% highlight ruby %}
https://trufflesuite.com/ganache/

https://github.com/ethereum/go-ethereum

https://ethereum.github.io/execution-apis/api-documentation/ rpc 规范
{% endhighlight %}

## 0x05 ethers.js 部署合约  (Lesson 5)
ethers.js 是最流行的基于 JavaScript 的工具包，另一个流行的包时 web3.js

{% highlight ruby %}
https://docs.ethers.io/v5/getting-started/

安装 ethers
$ yarn add ethers
安装后可以看到 package.json 里 dependencies 多了 ethers
  "dependencies": {
    "ethers": "^5.7.1",
    "solc": "^0.8.17"
  },

安装 fs-extra
$ yarn add fs-extra


部署合约
$ node deploy.js
{% endhighlight %}

#### 部署方式一
{% highlight ruby %}

  // deploy a contract ? Wait for it to be deployed，so need main function be async
  // ganache http://127.0.0.1:7545

  // 1 将连接到 http://127.0.0.1:7545 这个网址
  // 我们的部署脚本连接到 ganache 区块链
  const provider = new ethers.providers.JsonRpcProvider(process.env.RPC_URL!);

  // 2 paste private key from ganache accounts
  // 正常不要直接把私钥写这里，但这里是测试所以没有风险
  // 这里提供了对区块链的交互，我们可以通过私钥对交易进行签名等
  // Note 重启 ganache 私钥可能变化，记得替换私钥，否则部署不成功
  console.log(`private key is ${process.env.PRIVATE_KEY!}`);
  // get wallet 方式一
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!, provider);

  // get wallet 方式二
  // const encryptedJson = fs.readFileSync("./.encryptedKey.json", "utf8");
  // let wallet = new ethers.Wallet.fromEncryptedJsonSync(
  //   encryptedJson,
  //   process.env.PRIVATE_KEY_PASSWORD
  // );
  // wallet = await wallet.connect(provider);

  // 3 部署合约需要 abi 和 bin
  const abi = fs.readFileSync("./SimpleStorage_sol_SimpleStorage.abi", "utf8");
  const binary = fs.readFileSync(
    "./SimpleStorage_sol_SimpleStorage.bin",
    "utf8"
  );

  // 部署方式一
  const contractFactory = new ethers.ContractFactory(abi, binary, wallet);
  console.log("Deploying, please wait...");
  // STOP here! Wait for contract to deployed
  const contract = await contractFactory.deploy();
  console.log(`contract address is ${contract.address}`);

  // 指定 gasLimit 进行部署
  // const contract = await contractFactory.deploy({ gasLimit: 1000000000000000 });
  // STOP here！Wait for contract to deploy！

  // 等待 1 个区块确认，确保写入链上
  // transactionReceipt 是等后续区块确认的
  // 而 deploy response 是部署合约就得到的 它们是不同的
  const transactionReceipt = await contract.deployTransaction.wait(1);
  console.log("Here is the deployment transaction(transaction response): ");
  console.log(contract.deployTransaction);
  console.log("Here is the deployment receipt: ");
  console.log(transactionReceipt);
{% endhighlight %}

#### 部署 log
{% highlight ruby %}
Deploying, please wait...
Here is the deployment transaction(transaction response): 
{
  nonce: 2,
  gasPrice: BigNumber { _hex: '0x04a817c800', _isBigNumber: true },
  gasLimit: BigNumber { _hex: '0x08956d', _isBigNumber: true },
  to: null,
  value: BigNumber { _hex: '0x00', _isBigNumber: true },
  data: '0x608060405234801561001057600080fd5b5061093b806100206000396000f3fe608060405234801561001057600080fd5b50600436106100575760003560e01c80632e64cec11461005c5780636057361d1461007a5780636f760f41146100965780638bab8dd5146100b25780639e7a13ad146100e2575b600080fd5b610064610113565b60405161007191906102b2565b60405180910390f35b610094600480360381019061008f919061030d565b61011c565b005b6100b060048036038101906100ab9190610480565b610126565b005b6100cc60048036038101906100c791906104dc565b6101af565b6040516100d991906102b2565b60405180910390f35b6100fc60048036038101906100f7919061030d565b6101dd565b60405161010a9291906105a4565b60405180910390f35b60008054905090565b8060008190555050565b6001604051806040016040528083815260200184815250908060018154018082558091505060019003906000526020600020906002020160009091909190915060008201518160000155602082015181600101908161018591906107e0565b5050508060028360405161019991906108ee565b9081526020016040518091039020819055505050565b6002818051602081018201805184825260208301602085012081835280955050505050506000915090505481565b600181815481106101ed57600080fd5b906000526020600020906002020160009150905080600001549080600101805461021690610603565b80601f016020809104026020016040519081016040528092919081815260200182805461024290610603565b801561028f5780601f106102645761010080835404028352916020019161028f565b820191906000526020600020905b81548152906001019060200180831161027257829003601f168201915b5050505050905082565b6000819050919050565b6102ac81610299565b82525050565b60006020820190506102c760008301846102a3565b92915050565b6000604051905090565b600080fd5b600080fd5b6102ea81610299565b81146102f557600080fd5b50565b600081359050610307816102e1565b92915050565b600060208284031215610323576103226102d7565b5b6000610331848285016102f8565b91505092915050565b600080fd5b600080fd5b6000601f19601f8301169050919050565b7f4e487b7100000000000000000000000000000000000000000000000000000000600052604160045260246000fd5b61038d82610344565b810181811067ffffffffffffffff821117156103ac576103ab610355565b5b80604052505050565b60006103bf6102cd565b90506103cb8282610384565b919050565b600067ffffffffffffffff8211156103eb576103ea610355565b5b6103f482610344565b9050602081019050919050565b82818337600083830152505050565b600061042361041e846103d0565b6103b5565b90508281526020810184848401111561043f5761043e61033f565b5b61044a848285610401565b509392505050565b600082601f8301126104675761046661033a565b5b8135610477848260208601610410565b91505092915050565b60008060408385031215610497576104966102d7565b5b600083013567ffffffffffffffff8111156104b5576104b46102dc565b5b6104c185828601610452565b92505060206104d2858286016102f8565b9150509250929050565b6000602082840312156104f2576104f16102d7565b5b600082013567ffffffffffffffff8111156105105761050f6102dc565b5b61051c84828501610452565b91505092915050565b600081519050919050565b600082825260208201905092915050565b60005b8381101561055f578082015181840152602081019050610544565b60008484015250505050565b600061057682610525565b6105808185610530565b9350610590818560208601610541565b61059981610344565b840191505092915050565b60006040820190506105b960008301856102a3565b81810360208301526105cb818461056b565b90509392505050565b7f4e487b7100000000000000000000000000000000000000000000000000000000600052602260045260246000fd5b6000600282049050600182168061061b57607f821691505b60208210810361062e5761062d6105d4565b5b50919050565b60008190508160005260206000209050919050565b60006020601f8301049050919050565b600082821b905092915050565b6000600883026106967fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff82610659565b6106a08683610659565b95508019841693508086168417925050509392505050565b6000819050919050565b60006106dd6106d86106d384610299565b6106b8565b610299565b9050919050565b6000819050919050565b6106f7836106c2565b61070b610703826106e4565b848454610666565b825550505050565b600090565b610720610713565b61072b8184846106ee565b505050565b5b8181101561074f57610744600082610718565b600181019050610731565b5050565b601f8211156107945761076581610634565b61076e84610649565b8101602085101561077d578190505b61079161078985610649565b830182610730565b50505b505050565b600082821c905092915050565b60006107b760001984600802610799565b1980831691505092915050565b60006107d083836107a6565b9150826002028217905092915050565b6107e982610525565b67ffffffffffffffff81111561080257610801610355565b5b61080c8254610603565b610817828285610753565b600060209050601f83116001811461084a5760008415610838578287015190505b61084285826107c4565b8655506108aa565b601f19841661085886610634565b60005b828110156108805784890151825560018201915060208501945060208101905061085b565b8683101561089d5784890151610899601f8916826107a6565b8355505b6001600288020188555050505b505050505050565b600081905092915050565b60006108c882610525565b6108d281856108b2565b93506108e2818560208601610541565b80840191505092915050565b60006108fa82846108bd565b91508190509291505056fea264697066735822122088b471e3f349cdf59973b52669ea0389d906b58d0d3eae4b53170c96b03bdeb864736f6c63430008110033',
  chainId: 1337,
  v: 2709,
  r: '0x94482043a9c21d699febb0465c405e051fac1d3023cb99fd710a5379f24f037e',
  s: '0x08278eafcc07a19bef925b08c7cedfce5f4be5598925328448d7be9bebb7df4b',
  from: '0x32dbd6b6e50F87272BF35b6dc4D4edDA22CE03FC',
  hash: '0x6c4e98f8bbce8c72a79e983d30c914af0bcc172216a589af2b0e49f440ccca20',
  type: null,
  confirmations: 0,
  wait: [Function (anonymous)]
}
Here is the deployment receipt: 
{
  to: null,
  from: '0x32dbd6b6e50F87272BF35b6dc4D4edDA22CE03FC',
  contractAddress: '0x1BF18FaA2DB65Ba0e27F96b09c8f467f699EF1c8',
  transactionIndex: 0,
  gasUsed: BigNumber { _hex: '0x08956d', _isBigNumber: true },
  logsBloom: '0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000',
  blockHash: '0x53efee1dbf6a7d59fb0f5f78f419cd0ae57ab4205efa9fcca4b0fc49b9b23d84',
  transactionHash: '0x6c4e98f8bbce8c72a79e983d30c914af0bcc172216a589af2b0e49f440ccca20',
  logs: [],
  blockNumber: 3,
  confirmations: 1,
  cumulativeGasUsed: BigNumber { _hex: '0x08956d', _isBigNumber: true },
  status: 1,
  type: 0,
  byzantium: true,
  events: []
}
{% endhighlight %}

### Sending a “raw” transaction in ethersjs
{% highlight ruby %}
  // 部署方式二
  // Note 一般不会用这个方式，还是用方式一，tx 相关内容 hardhat 和 ethers 会帮我们做
  console.log("Let's deploy with only transaction data!");
  // data 是 SimpleStorage_sol_SimpleStorage.bin
  // data 是合约的 bin，就是部署操作
  // chainId 是 ganache Network id
  const nonce = await wallet.getTransactionCount();
  const tx = {
    nonce: nonce,
    gasPrice: 20000000000,
    gasLimit: 1000000,
    to: null,
    data: "0x608060405234801561001057600080fd5b5061093b806100206000396000f3fe608060405234801561001057600080fd5b50600436106100575760003560e01c80632e64cec11461005c5780636057361d1461007a5780636f760f41146100965780638bab8dd5146100b25780639e7a13ad146100e2575b600080fd5b610064610113565b60405161007191906102b2565b60405180910390f35b610094600480360381019061008f919061030d565b61011c565b005b6100b060048036038101906100ab9190610480565b610126565b005b6100cc60048036038101906100c791906104dc565b6101af565b6040516100d991906102b2565b60405180910390f35b6100fc60048036038101906100f7919061030d565b6101dd565b60405161010a9291906105a4565b60405180910390f35b60008054905090565b8060008190555050565b6001604051806040016040528083815260200184815250908060018154018082558091505060019003906000526020600020906002020160009091909190915060008201518160000155602082015181600101908161018591906107e0565b5050508060028360405161019991906108ee565b9081526020016040518091039020819055505050565b6002818051602081018201805184825260208301602085012081835280955050505050506000915090505481565b600181815481106101ed57600080fd5b906000526020600020906002020160009150905080600001549080600101805461021690610603565b80601f016020809104026020016040519081016040528092919081815260200182805461024290610603565b801561028f5780601f106102645761010080835404028352916020019161028f565b820191906000526020600020905b81548152906001019060200180831161027257829003601f168201915b5050505050905082565b6000819050919050565b6102ac81610299565b82525050565b60006020820190506102c760008301846102a3565b92915050565b6000604051905090565b600080fd5b600080fd5b6102ea81610299565b81146102f557600080fd5b50565b600081359050610307816102e1565b92915050565b600060208284031215610323576103226102d7565b5b6000610331848285016102f8565b91505092915050565b600080fd5b600080fd5b6000601f19601f8301169050919050565b7f4e487b7100000000000000000000000000000000000000000000000000000000600052604160045260246000fd5b61038d82610344565b810181811067ffffffffffffffff821117156103ac576103ab610355565b5b80604052505050565b60006103bf6102cd565b90506103cb8282610384565b919050565b600067ffffffffffffffff8211156103eb576103ea610355565b5b6103f482610344565b9050602081019050919050565b82818337600083830152505050565b600061042361041e846103d0565b6103b5565b90508281526020810184848401111561043f5761043e61033f565b5b61044a848285610401565b509392505050565b600082601f8301126104675761046661033a565b5b8135610477848260208601610410565b91505092915050565b60008060408385031215610497576104966102d7565b5b600083013567ffffffffffffffff8111156104b5576104b46102dc565b5b6104c185828601610452565b92505060206104d2858286016102f8565b9150509250929050565b6000602082840312156104f2576104f16102d7565b5b600082013567ffffffffffffffff8111156105105761050f6102dc565b5b61051c84828501610452565b91505092915050565b600081519050919050565b600082825260208201905092915050565b60005b8381101561055f578082015181840152602081019050610544565b60008484015250505050565b600061057682610525565b6105808185610530565b9350610590818560208601610541565b61059981610344565b840191505092915050565b60006040820190506105b960008301856102a3565b81810360208301526105cb818461056b565b90509392505050565b7f4e487b7100000000000000000000000000000000000000000000000000000000600052602260045260246000fd5b6000600282049050600182168061061b57607f821691505b60208210810361062e5761062d6105d4565b5b50919050565b60008190508160005260206000209050919050565b60006020601f8301049050919050565b600082821b905092915050565b6000600883026106967fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff82610659565b6106a08683610659565b95508019841693508086168417925050509392505050565b6000819050919050565b60006106dd6106d86106d384610299565b6106b8565b610299565b9050919050565b6000819050919050565b6106f7836106c2565b61070b610703826106e4565b848454610666565b825550505050565b600090565b610720610713565b61072b8184846106ee565b505050565b5b8181101561074f57610744600082610718565b600181019050610731565b5050565b601f8211156107945761076581610634565b61076e84610649565b8101602085101561077d578190505b61079161078985610649565b830182610730565b50505b505050565b600082821c905092915050565b60006107b760001984600802610799565b1980831691505092915050565b60006107d083836107a6565b9150826002028217905092915050565b6107e982610525565b67ffffffffffffffff81111561080257610801610355565b5b61080c8254610603565b610817828285610753565b600060209050601f83116001811461084a5760008415610838578287015190505b61084285826107c4565b8655506108aa565b601f19841661085886610634565b60005b828110156108805784890151825560018201915060208501945060208101905061085b565b8683101561089d5784890151610899601f8916826107a6565b8355505b6001600288020188555050505b505050505050565b600081905092915050565b60006108c882610525565b6108d281856108b2565b93506108e2818560208601610541565b80840191505092915050565b60006108fa82846108bd565b91508190509291505056fea264697066735822122088b471e3f349cdf59973b52669ea0389d906b58d0d3eae4b53170c96b03bdeb864736f6c63430008110033",
    chainId: 1337,
    value: 0,
  };
  // 只是签名
  // const signedTxRepsonse = await wallet.signTransaction(tx);
  // console.log("Here is the signedTxRepsonse: ");
  // console.log(signedTxRepsonse);

  // sendTransaction 部署
  const sendTxRepsonse = await wallet.sendTransaction(tx);
  await sendTxRepsonse.wait(1); // 等 1 个区块确认
  console.log("Here is the sendTxRepsonse: ");
  console.log(sendTxRepsonse);
{% endhighlight %}

### interacting with Contracts in Ethersjs
让 abi 更可读
{% highlight ruby %}
1 可将 SimpleStorage_sol_SimpleStorage.abi 改成 SimpleStorage_sol_SimpleStorage.json
2 然后 command + shift + P，搜索 format，选择格式化文档选择 Prettier
3 将 SimpleStorage_sol_SimpleStorage.json 改成 SimpleStorage_sol_SimpleStorage.abi
{% endhighlight %}

反编译工具
https://ethervm.io/decompile 

{% highlight ruby %}
  // 4.1 Get FavoriteNumber
  // BigNumber 是一个自带的库
  const currentFavoriteNumber = await contract.retrieve();
  console.log(currentFavoriteNumber);
  console.log(`currentFavoriteNumber is: ${currentFavoriteNumber.toString()}`);
  // 传值最好用字符串
  // 当调用合约的方法时候得到一个 transactionResponse
  // 当 transaction 结束时 得到一个 transactionReceipt
  const transactionResponseStore = await contract.store("7");
  const transactionReceiptStore = await transactionResponseStore.wait(1);
  const updateFavoriteNumber = await contract.retrieve();
  console.log(`updateFavoriteNumber is: ${updateFavoriteNumber.toString()}`);
{% endhighlight %}

### 环境变量 Environment Variable
到目前为止，我们将私钥直接写在代码中，这样比较危险，可以通过 Environment Variable File，管理私钥
{% highlight ruby %}
1 在 deploy.js 同级创建 .env 文件
2 编辑 .env 添加私钥，PRIVATE_KEY=xxxx
3 $ yarn add dotenv
参考 https://www.npmjs.com/package/dotenv
4 在 deploy.js 最上边加上 require("dotenv").config();
5 修改 const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);

6 add .env node_modules to gitignore 文件

{% endhighlight %}

### Better Private Key Management
* 方式一：如果你不想使用 env 文件可以运行时候输入私钥
{% highlight ruby %}
$ RPC_URL=http://127.0.0.1:7545 PRIVATE_KEY=你的私钥 node deploy.js
{% endhighlight %}

* 方法二：将私钥加密，私钥只用一次，生成 encryptedKey.json 后删除
{% highlight ruby %}
1 创建 encryptKey.js 用于加密私钥
2 编写 encryptKey.js
const ethers = require("ethers");
const fs = require("fs-extra");
require("dotenv").config();

async function main() {
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY);
  // 返回加密的秘钥
  const encryptedJsonKey = await wallet.encrypt(
    process.env.PRIVATE_KEY_PASSWORD,
    process.env.PRIVATE_KEY
  );
  // 别人需要知道密码才行
  console.log(encryptedJsonKey);
  fs.writeFileSync("./.encryptedKey.json", encryptedJsonKey);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.log(error);
    process.exit(1);
  });
  
3 node encryptKey.js
4 add encryptedKey.json to gitignore
5 删除 .env 中的 PRIVATE_KEY 和 PRIVATE_KEY_PASSWORD


6  修改 get wallet 代码
  const encryptedJson = fs.readFileSync("./.encryptedKey.json", "utf8");
  let wallet = new ethers.Wallet.fromEncryptedJsonSync(
    encryptedJson,
    process.env.PRIVATE_KEY_PASSWORD
  );
  wallet = await wallet.connect(provider);


{% endhighlight %}

通过方式二，修改 deploy.js，
{% highlight ruby %}
  get wallet 方式二 （推荐）
  部署命令行输入：PRIVATE_KEY_PASSWORD=password node deploy.js
  const encryptedJson = fs.readFileSync("./.encryptedKey.json", "utf8");
  let wallet = new ethers.Wallet.fromEncryptedJsonSync(
    encryptedJson,
    process.env.PRIVATE_KEY_PASSWORD
  );
  wallet = await wallet.connect(provider);
  
命令行重新部署输入：PRIVATE_KEY_PASSWORD=password node deploy.js
这样 PRIVATE_KEY_PASSWORD 也不需要存在 .env 文件中
{% endhighlight %}

Tips：
{% highlight ruby %}
你可以 $ env_变量名称=xxx node deploy.js
只要 env_变量名称 是通过 process.env.{env_变量名称} 定义的
{% endhighlight %}

### Optional Prettier Formating
1 $ yarn add prettier prettier-plugin-solidity
2 与 .env 文件同级创建 .prettierrc 文件
{% highlight ruby %}
.prettierrc 内容
{
  "tabWidth": 2,
  "semi": true,
  "useTabs": false,
  "singleQuote": false
}

Tips: command+shift+v README.md 预览快捷键
{% endhighlight %}



### Deploying to Testnet or Mainnet

{% highlight ruby %}
可以使用以太坊连接
https://github.com/ethereum/go-ethereum

或使用
https://infura.io/zh
{% endhighlight %}

这里演示使用 alchemy，alchemy 有个节点服务可连接到任何区块链
{% highlight ruby %}
1 注册账号
https://www.alchemy.com/

2 创建 app 这里选择 Goerli 测试网，然后在 dashboard 可以看到 node 的相关信息
https://dashboard.alchemy.com/
可以选择 view keys 查看 keys，这里我们只关心 http key，即我们连接到 Goerli 测试网的 RPC URL

以下以 Goerli 测试网为例子
3 copy http key，修改 .env 文件，
修改 RPC_URL=RPC_URL=http://127.0.0.1:7545 -> RPC_URL=你申请的 Goerli URL

4 copy metamask 私钥到 .env 文件
PRIVATE_KEY=Your private key

5 $ node deploy.js 部署即可
由于是测试网而非 ganache 本地区块链网络，所以需要等待出块，需要等一会
成功后输出合约地址
Deploying, please wait...
contract address is 0x55AaC750Dca0a46C462553F07D0664b97528d781
然后可以在 https://goerli.etherscan.io/address/0x55AaC750Dca0a46C462553F07D0664b97528d781 看到
{% endhighlight %}

Tips
{% highlight ruby %}
换链部署其实就是换下 RPC URL 就行了
{% endhighlight %}

### Verify on Block Explorer from the UI （Verify Contrat）
一般部署后没有 Verify 会显示 bytecode，你可以使用反编译器编译 bytecode，但需要花费一些时间
点击 https://goerli.etherscan.io/address/0x55AaC750Dca0a46C462553F07D0664b97528d781 中 Contrat Tab 的
Verify and Publish，根据提示选择并粘贴 solidity 合约代码

### Alchemy Dashboard & The MemPool（内存池）
内存池就类似候诊室，就是节点本地维护的内存交易池，如果有等待交易的会放到内存池

Alchemy Dashboard 有个 MemPool Tab


### 使用 Typescript

{% highlight ruby %}
$ yarn add typescript ts-node

1 修改 deploy.js -> deploy.ts
注释 
// const ethers = require("ethers");
// const fs = require("fs-extra");
// require("dotenv").config();
改为
import {ethers} from "ethers"
import * as fs from "fs-extra"
import "dotenv/config"

2 修改 encryptKey.js -> encryptKey.ts
注释 
// const ethers = require("ethers");
// const fs = require("fs-extra");
// require("dotenv").config();
改为
import {ethers} from "ethers"
import * as fs from "fs-extra"
import "dotenv/config"

3 
3.1 install typescript
$ yarn add typescript ts-node 或 $ npm install -g ts-node
ts-node 是 typescript 一个版本

3.2
$ yarn add @types/fs-extra  

4 $ts-node deploy.ts 部署合约

{% endhighlight %}


## 0x06 How to use Hardhat (Lesson 6)
Hardhat 是一个开发环境允许基于 JavaScript 的开发

### 6.1 Setup Hardhat 

{% highlight ruby %}
https://hardhat.org/

1 
$ mkdir hardhat-simple-storage-fcc

2 $yarn init  然后一路回车

3 删除 package.json 中 "main": "index.js"

4 $yarn add --dev hardhat
 
5 在安装 hardhat 同一目录中, $yarn hardhat 或 npm hardhat 也行,
全部选 y 回车，然后 hardhat 项目就创建好了
安装好后有 contracts、node_modules、scripts、test 文件夹

在 node_modules 中，比如 @metamask 由 metamask 团队开发，@xxx 这称为 scope package
scripts 是我们写 deploy.js 的地方
test 文件夹用于测试合约，创建好项目后会有一些测试代码
hardhat.config 是所有脚本的入口点，这是配置文件，决定了我们的代码如何与区块链互动
你可以再次运行 $ yarn hardhat 运行 hardhat 一些命令


6 编译合约与用 ethers.js 或 slope.js 编译类似
$ yarn hardhat compile
编译合约后会生成一个 cache 和 artifacts 文件夹
cache 用于快速访问合约
artifacts 包含所有我们编译合约的代码信息
{% endhighlight %}

### 6.2 解决 hardhat 相关问题

* 常见问题一是 hardhat.config.js 配置文件引起的，需要找到它进行删除。
如果运行 yarn hardhat 遇到了问题，可能是当前目录没有 hardhat.config.js，可以回退一级目录试试
{% highlight ruby %}
$ ride % npx hardhat --verbose
{% endhighlight %}

* 常见问题二：别忘记 yarn init

### 6.3 Deploying Contract with Hardhat 
{% highlight ruby %}
1 copy SimpleStorage 合约代码到这个项目

2 编译合约
$ yarn hardhat compile 
Note：合约版本要和 hardhat.config.js 中要求的合约版本兼容才行

3 $yarn add --dev prettier prettier-plugin-solidity
创建 .prettierrc 文件，告诉 prettier 格式化那些
创建  .prettierignore 文件，告诉 prettier 不格式化那些文件

4 $ yarn hardhat run scripts/deploy.js 


async function main() {
    const SimpleStorageFactory = await ethers.getContractFactory(
        "SimpleStorage"
    );
    console.log("Deploying contract...");
    const SimpleStorage = await SimpleStorageFactory.deploy();
    await SimpleStorage.deployed();
    console.log(`Deployed contract to: ${SimpleStorage.address}`);
}
Deploying contract...
Deployed contract to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
✨  Done in 2.81s.
这里不输入 RPC_URL 和私钥也可以部署成功
{% endhighlight %}

### 6.4 Network in Hardhat 
Hardhat 内置了一个 Hardhat Network，即为开发而设计的本地以太坊网络节点类似于 Ganache，允许你进行合约测试，
我们默认部署到 Hardhat Network，只是它不像 Ganache 它没有 UI，只运行在后台

{% highlight ruby %} 
1 实际上，如果我们 hardhat.config.js 没有添加任何网络，则他会添加默认网络 “hardhat”，这是 hardhat 的优势   
module.exports = {
  // default network: hardhat
  solidity: "0.8.17",
};
即类似
module.exports = {
  defaultNetwork: "hardhat",
  solidity: "0.8.17",
};


2 或使用 --network 选项指定部署的 RPC_URL
$ yarn hardhat run scripts/deploy.js --network hardhat

3 我们也可以添加其他网络
3.1 创建 .env 文件，.env -> 写入 GOROLI_RPC_URL 和 GEROLI_RPC_URL
PRIVATE_KEY=你的私钥
GEROLI_RPC_URL=你申请的 Goerli URL

3.2 $ yarn add --dev dotenv 启用 env

3.3 编辑 hardhat.config.ts

// -> ts
import "@nomicfoundation/hardhat-toolbox";
import "dotenv/config"; // 启用 env
import "@nomiclabs/hardhat-etherscan";
import "./tasks/block-number";
import "hardhat-gas-reporter";
import "solidity-coverage";

// 这个不用 import 因为 @nomiclabs/hardhat-etherscan 导入了
// 但如果你想显示声明 可以 import 下
import "@nomiclabs/hardhat-ethers";

import "@typechain/hardhat";

const GOERLI_RPC_URL = process.env.GOERLI_RPC_URL;
const PRIVATE_KEY = process.env.PRIVATE_KEY;
const ETHERSCAN_API_KEY = process.env.ETHERSCAN_API_KEY;
const COINMARKETCAP_API_KEY = process.env.COINMARKETCAP_API_KEY;

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
    defaultNetwork: "hardhat",
    networks: {
        goerli: {
            // 添加 goerli 测试网
            // 教程是直接写 PRIVATE_KEY，但我运行失败，看文档是需要字符串，所以这里我改成了字符串字面量模板，it's work
            url: `${GOERLI_RPC_URL}`,
            accounts: [`${PRIVATE_KEY}`],
            chainId: 5,
        },
        localhost: {
            url: "http://127.0.0.1:8545/",
            // accounts use same with hardhat
            chainId: 31337,
        },
    },
    solidity: "0.8.17",
    etherscan: {
        apiKey: {
            goerli: "你申请的 apiKey",
        },
        customChains: [
            {
                network: "goerli",
                chainId: 5,
                urls: {
                    apiURL: "https://api-goerli.etherscan.io/api",
                    browserURL: "https://goerli.etherscan.io/",
                },
            },
        ],
    },
    gasReporter: {
        enabled: true,
        outputFile: "gas-report.txt",
        noColors: true,
        currency: "CNY",
        coinmarketcap: "xxx", // api call get USD price for gas
        token: "ETH", // 查看部署到 polygon，价格转换为 polygon 的代币
    },
};


Tips：你可以通过 https://chainlist.org/ 查看各网络的 chainId

4 部署到 goreli 测试网络
$ yarn hardhat run scripts/deploy.js --network goreli 
{% endhighlight %}

### 6.5 Programatic Verification
{% highlight ruby %} 
部署完合约是未验证的，可以在这里查看可调用的 API，https://docs.etherscan.io/，这里提供编程方式与 etherscan 交互

document to how to verify contract 
https://docs.etherscan.io/tutorials/verifying-contracts-programmatically

hardhat 还可以添加插件，比较流行的是 hardhat-etherscan
https://hardhat.org/hardhat-runner/plugins/nomiclabs-hardhat-etherscan

1 安装 hardhat-etherscan
$ yarn add --dev @nomiclabs/hardhat-etherscan

2 在 hardhat.config.js 顶部添加 require("@nomiclabs/hardhat-etherscan");

3 验证合约需要注册 etherscan
登录 etherscan.io，选择 APIKey，点击 ADD 按钮添加 APIKey，然后 copy 新创建的 APIKey 
编辑 .env 添加 ETHERSCAN_API_KEY=XXXXX

4 编辑 hardhat.config.js
const ETHERSCAN_API_KEY = process.env.ETHERSCAN_API_KEY;
然后在 module.exports 中新增
  etherscan: {
    apiKey: `${ETHERSCAN_API_KEY}`
  }
  
5 $ yarn hardhat
AVAILABLE TASKS 里会出现 verify，之后你可以通过命令行进行验证了

6 $  yarn hardhat verify --help 查看 verify 如何使用

7 verify 函数
// 用于部署后验证合约
// args 是填充合约 constructor 的参数
async function verify(contractAddress, args) {
    console.log("Verifying contract...");
    // 由于 verify 可能有会异常，这里需要 try
    try {
        await run("verify:verify", {
            address: contractAddress,
            constructorArguments: args,
        });
    } catch (e) {
        if (e.message.toLowerCase().includes("already verified")) {
            console.log("Already Verified!");
        } else {
            console.log(e);
        }
    
}

{% endhighlight %}

### 6.6 Artifacts Troubleshooting
{% highlight ruby %} 
Note：部署合约可能遇到问题，可尝试删除 artifacts 和 cache 文件夹，部署合约会重新编译生成 artifacts 和 cache
重新部署 $ yarn hardhat run scripts/deploy.js --network geroli

删除它们后 hardhat 会重新编译，输出 Compiled 1 Solidity file successfully
{% endhighlight %}

### 6.7 Custom Hardhat Tasks

{% highlight ruby %} 
https://hardhat.org/hardhat-runner/docs/advanced/create-task

1 在 scripts 同级创建 tasks 目录
2 在 tasks 文件夹里创建 block-number.js
3 在 hardhat.config.js 顶部添加 require("./tasks/block-number");

Note Note Note：只有做了这步，yarn hardhat 命令才会出现自己创建的 block-number task

4 $ yarn hardhat block-number 查看 hardhat 默认网
Current block number: 0
✨  Done in 2.59s.

$ yarn hardhat block-number --network goerli
Current block number: 7703297

block-number.js 代码
const {task} = require("hardhat/config");

// task name is block-number
// task description is 
// setAction 即 task 实际做的任务
task("block-number", "Prints the current block number").setAction(
    // async 表示这是一个异步任务
    // 定义函数方式一：const blockTask = async (taskArgs, hre) {}
    // 定义函数方式二：async function blockTask() {} 
    // 这俩方式一样

    // 当 我们运行 task 自动执行下边这个函数，并自动传递参数
    // hre 是 hardhat 运行时环境，
    // 就像 const {task} = require("hardhat/config"); 声明 task 一样
    async (taskArgs, hre) => {
        const blockNumber = await hre.ethers.provider.getBlockNumber();
        console.log(`Current block number: ${blockNumber}`);
    }
)

{% endhighlight %}

### 6.8 Hardhat Localhost Node
每次默认的 Hardhat 网络你部署完合约都会重置，但我们也可以让它不重置，变成和 ganache 一样。
可以本地启动一个独立的节点网络，但这个节点与 hardhat 默认网不一样。但它仍然使用 hardhat 的运行时环境

{% highlight ruby %} 
1 $ yarn hardhat node 在本地启动一个节点，包含很多测试 account
Started HTTP and WebSocket JSON-RPC server at http://127.0.0.1:8545/ 
这是我们启动的一个 hardhat 节点而不是 hardhat 默认的节点

2 hardhat.config.js 中添加 localhost
这里不需要添加 accounts 因为 hardhat 提供了

module.exports = {
  defaultNetwork: "hardhat",
  networks: {
    goerli: {
      url: `${GOERLI_RPC_URL}`,
      accounts: [`${PRIVATE_KEY}`],
      chainId: 5,
    },
    localhost: {
      url: "http://127.0.0.1:8545/",
      // accounts use same with hardhat
      chainId: 31337, 同 hardhat
    },
  },

3 $ yarn hardhat run scripts/deploy.js --network localhost

4 节点 log，我们可以通过查看节点 log 进行调试非常方便
eth_call
  Contract call:       SimpleStorage#retrieve
  From:                0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
  To:                  0x5fbdb2315678afecb367f032d93f642f64180aa3

eth_chainId
eth_estimateGas
eth_gasPrice
eth_sendTransaction
  Contract call:       SimpleStorage#store
  Transaction:         0x05689686ae0d13b78a474dacacb9babcf4bf2cd5cefe84ae984eb03df63c5922
  From:                0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
  To:                  0x5fbdb2315678afecb367f032d93f642f64180aa3
  Value:               0 ETH
  Gas used:            43724 of 43724
  Block #2:            0xf640cf29d33714b2b320441c6571b402d289b028b65be68b33df2b3720b71280

{% endhighlight %}

### 6.9 Hardhat Console
{% highlight ruby %} 
Console 是一个 JavaScript 环境，让我们运行 jav 命令与之任何链交互
$ yarn hardhat console 跳转到 console
$ yarn hardhat console --network localhost

console 不需要 import ethers 你可以直接使用
像这样 
> const SimpleStorageFactory = await ethers.getContractFactory("SimpleStorage");
> const SimpleStorage = await SimpleStorageFactory.deploy();
> await SimpleStorage.retrieve()
> await SimpleStorage.store("7")

> await ethers.provider.getBlockNumber()
Ctrl+C 两次可以推出，Hardhat Console 适用于任何网络
{% endhighlight %}

### 6.10 Running Tests
{% highlight ruby %} 
$ yarn hardhat clean 清理 artifacts 和 cache

测试是非常重要的，https://rekt.news/leaderboard/ 展示了黑客攻击的事件

1 在 test 目录新建，test-deploy.js
2 hardhat 测试使用 mocha 框架，这是一个基于 JavaScript 的框架

3 $  yarn hardhat test
输出
  SimpleStorage
    ✔ Should start with a favorite number of 0
    ✔ Should update when we call store


  2 passing (2s)

✨  Done in 2.42s.

4 执行特定一个测试用例
grep 指定关键字，hardhat 会查找测试用例描述中带关键字的用例
$ yarn hardhat test --grep store 

4.1 或在 it 后加 only 表示 $yarn hardhat test 时只运行带 only 的用例
  it.only("Should update when we call store", async function () {


全部测试代码

const {ethers} = require("hardhat");
const {expect, assert} = require("chai");


// SimpleStorage 测试名称
// 后边是测试的函数，一般用匿名函数
// describe("SimpleStorage", () => {}); 这个下边的写法基本一样, 但下边是最佳实践
describe("SimpleStorage", function () {

  let simpleStorageFactory, simpleStorage
  // do sth before every it 
  // 每次测试前都需要部署合约
  beforeEach(async function () {
    simpleStorageFactory = await ethers.getContractFactory("SimpleStorage")
    simpleStorage = await simpleStorageFactory.deploy()
  })

  // 每一个测试内容块
  it.only("Should start with a favorite number of 0", async function () {
    const currentValue = await simpleStorage.retrieve()
    const expectedValue = "0"
    // assert
    // expect
    // 预期当前值为 0
    assert.equal(currentValue.toString(), expectedValue)
    // expect 这行等价于 上边用 assert，这取决于你喜欢用哪个
    expect(currentValue.toString()).to.equal(expectedValue)

  })

  it.only("Should update when we call store", async function () {
    const expectedValue = "7"
    const transactionResponse = await simpleStorage.store(expectedValue)
    await transactionResponse.wait(1)

    const currentValue = await simpleStorage.retrieve()
    assert.equal(currentValue.toString(), expectedValue)
  })

  // 可以再嵌套，非常有用，用于模块化测试
  // describe("something", () => {
  //   beforeEach
  //   it()
  // })

  
}); 

{% endhighlight %}

### 6.11 Hardhat Gas Report
{% highlight ruby %} 
hardhat-gas-reporter 
https://www.npmjs.com/package/hardhat-gas-reporter

1 安装
$ yarn add hardhat-gas-reporter --dev

2 编辑 hardhat.config.js
require("hardhat-gas-reporter");
const COINMARKETCAP_API_KEY = process.env.COINMARKETCAP_API_KEY

和
  gasReporter: {
    enabled: true,
    outputFile: "gas-report.txt", // gas 使用情况输出到文件，别忘了把 gas-report.txt 加入 .gitignore
    noColors: true,
    currency: "USD",
    coinmarketcap:COINMARKETCAP_API_KEY, // api call get USD price for gas
    token: "MATIC",  // 查看部署到 polygon，价格转换为 polygon 的代币
  },

3 注册 coinmarketcap
https://pro.coinmarketcap.com/account
然后 copy api key 到 .env
COINMARKETCAP_API_KEY=your coinmarketcap api key

4 $yarn hardhat test

{% endhighlight %}

### 6.12  Solidity Coverage
{% highlight ruby %} 

https://github.com/sc-forks/solidity-coverage

Solidity Coverage 告诉你有多少 .sol 代码进行了测试

1 $ yarn add --dev solidity-coverage

2 编辑 hardhat.config.js
添加 require("solidity-coverage");

3 $ yarn hardhat coverage
输出
--------------------|----------|----------|----------|----------|----------------|
File                |  % Stmts | % Branch |  % Funcs |  % Lines |Uncovered Lines |
--------------------|----------|----------|----------|----------|----------------|
 contracts/         |       50 |      100 |    66.67 |       50 |                |
  SimpleStorage.sol |       50 |      100 |    66.67 |       50 |          30,31 |
--------------------|----------|----------|----------|----------|----------------|
All files           |       50 |      100 |    66.67 |       50 |                |
--------------------|----------|----------|----------|----------|----------------|
合约第 30 和 31 行没进行测试

coverage.json 需要加入 .gitignore

{% endhighlight %}

### 6.13 Hardhat Waffle
Hardhat Waffle 是一个测试框架
https://getwaffle.io/

https://hardhat.org/hardhat-runner/plugins/nomiclabs-hardhat-waffle

### 6.14 To Typescript
{% highlight ruby %} 
1 $yarn add @typechain/ethers-v5 @typechain/hardhat @types/chai @types/node @types/mocha ts-node typechain typescript

2 编辑 package.json 
删除下边两行
  },
  "dependencies": {
统一成 devDependencies

3 
deploy.js -> deploy.ts
hardhat.config.js -> hardhat.config.ts
block-number.js -> block-number.ts
test-deploy.js -> test-deploy.ts

4 与 package.json 同级创建 tsconfig.json
内容如下
{
    "compilerOptions": {
        "target": "es2018",
        "module": "commonjs",
        "strict": true,
        "esModuleInterop": true,
        "outDir": "dist",
    },
    "include": ["./scripts", "./test"],
    "files": ["./hardhat.config.ts"],
}

5 block-number 需要加 export defult

6 修改相关 require 为 import

7 $ yarn hardhat run scripts/deploy.ts 
应该可以 work


8 yarn hardhat test 可能会报错
需要 修改 hatdhat.config.ts，添加 import "@typechain/hardhat";
然后 $yarn hardhat 查看可以看到 typechain 可以生成有类型的合约
然后执行 $ yarn hardhat typechain 将创建一个新的名为 typechain-types 文件夹，里边有所有合约的合约类型
生成成功后，修改 test-deploy.ts 添加 import {SimpleStorage, SimpleStorage__factory} from "../typechain-types"

{% endhighlight %}
