---
layout: post
title: Learn Blockchain-Solidity-Full Stack Web3-02
date: 2022-09-01 16:45:30.000000000 +09:00
tag: 区块链
---

## 0x01 Hardhat Fund Me( Lesson 7)

### 1 Hardhat Setup

{% highlight ruby %} 
1 $ mkdir hardhat-fund-me-fcc

2 $ yarn add --dev hardhat

3 $ yarn hardhat 选择 js 模板

4 solhint
eslint https://eslint.org/ 是 JavaScript 的 lint 可以帮助找到代码中的问题

而 solint 是 solidity lint, https://github.com/protofire/solhint
用于检查我们的 solidity 代码

添加 solhint
$ yarn add solhint
$ yarn solhint --init
$ yarn solhint contracts/*.sol  // lint contracts

5 安装 prettier
vs code 快捷键：command + shift + p 打开命令面板
5.1 输入 setting 选择 worksapce JSON settings，打开 worksapce JSON settings
5.2 编辑添加 
{
    "[solidity]": {
        "editor.defaultFormatter": "NomicFoundation.hardhat-solidity"
    }, 
    "[javascript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    }, 
}

6 copy FundMe 合约

7 $ yarn add dotenv
{% endhighlight %}


### 2 importing from npm
{% highlight ruby %} 
由于 FundMe 合约使用了 chainlink 所以这里编译合约报错
Error HH411: The library @chainlink/contracts, imported from contracts/FundMe.sol, is not installed. Try installing it using npm.

我们必须告诉 hardhat chainlink，这里下载 chainlink，https://www.npmjs.com/package/@chainlink/contracts
$ yarn add --dev @chainlink/contracts
下载 chainlink 后，再编译合约就可以通过了
$ yarn hardhat compile 

注意：如果编译过了，显示 Nothing to compile，可以执行 $ yarn hardhat clean 后再 $ yarn hardhat compile 

{% endhighlight %}

### 3 Hardhat Deploy（use hardhat-deploy）

{% highlight ruby %} 
之前我们使用自己创建的 deploy.js 部署合约，这次我们使用 hardhat-deploy
https://github.com/wighawag/hardhat-deploy
1 安装 hardhat-deploy
$ yarn add --dev hardhat-deploy
2 安装后 编辑 hardhat.config.js 
添加 require("hardhat-deploy")
3 可以删除 scripts 文件夹下的 deploy.js 部署脚本
4 $ yarn hardhat 可以看到一些任务，其中有一个 deploy 任务用于部署合约 

5 $ mkdir deploy

6 使用 hardhat-deploy-ethers 替代 ethers
$ yarn add --dev @nomiclabs/hardhat-ethers@npm:hardhat-deploy-ethers ethers
然后查看 package.json 可以看到，
"@nomiclabs/hardhat-ethers": "npm:hardhat-deploy-ethers"，
表示 hardhat-ethers 目前使用 npm:hardhat-deploy-ethers

7 在 deploy 文件夹下创建 01-deploy-fund-me.js
如果不使用 hardhat-deploy 我们的部署脚本一般包括 import、main function、call main function
但是使用 hardhat-deploy，我们只需要 import，而执行时通过指定函数实现的
Sample begin：
// not use hardhat-deploy, need 3 phase
// 1 import
// 2 main function
// 3 calling of main function

// use hardhat-deploy
// 1 specify a default deploy function
async function deployFunc() {
  console.log("Hi! This is 01-deploy-fund-me")
}

// set the default func
// 这里传函数名称，即函数对象不需要加括号
module.exports.default = deployFunc
Sample end：

$ yarn hardhat deploy 部署合约

编辑 hardhat.config.js 添加 namedAccounts（与 networks 同级）
  namedAccounts: {
    deployer: {
      // 哪个账户进行部署
      default: 0,
      5: 1, // index = 1 用户进行部署在 Goerli 测试网
      31337: 2,
    },
    user: {
      default: 1,
    },
  },

{% endhighlight %}

### 4 Mocking & helper-hardhat-config
{% highlight ruby %}  
我们部署到 Goerli 测试网会比较慢，这时可以部署到 hardhat 进行测试。但需要使用 Mocking chainlink price 到 hardhat 进行测试

Mocking 主要用于单元测试，可以理解为 Mocking 是 fake 的

部署代码如下：
  const priceFeedAddress = "0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e"
  // 部署 FundMe 合约
  const fundMe = await deploy("FundMe", {
    from: deployer,
    args: [priceFeedAddress], // put price feed address
    log: true,
  })

## 使用 aave
上边代码当我们通过 chainlink 获取价格时候，可能会部署多个地址，比如
  // if chainId is X , priceFeedAddress is Y
  // if chainId is Z , priceFeedAddress is A
所以我们需要根据不同链进行不同的部署
so we can use aave，https://github.com/aave/aave-v3-core
aave 是一个协议 可以根据不同链进行不同配置

1 创建 helper.hardhat-config.js
查看连接：https://docs.chain.link/data-feeds/price-feeds/addresses/?network=polygon
编辑内容如下：
const { mod } = require("@nomicfoundation/ethereumjs-evm/dist/opcodes")

const networkConfig = {
  5: {
    name: "goerli",
    ethUsdPriceFeed: "0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e",
  },
  //   137: {
  //     name: "polygon",
  //     ethUsdPriceFeed: "0x0715A7794a1dc8e42615F059dD6e406A6594651A",
  //   },
  // 31337
}

// defines for MockV3Aggregator
const developmentChains = ["hardhat", "localhost"]
const DECIMALS = 8
const INITIAL_ANSWER = 200000000000

module.exports = {
  networkConfig,
  developmentChains,
  DECIMALS,
  INITIAL_ANSWER,
}

2 编辑 01-deploy-fund-me.js 添加如下：
const { networkConfig, developmentChains } = require("../helper-hardhat-config")

3 创建 00-deploy-mocks.js
4 习惯是在 contracts 目录创建 test 目录，然后在 test 目录下创建 MockV3Aggregator.sol，即 mock 相关合约都放到 test 目录

MockV3Aggregator.sol 内容如下：
// 这是为了 mock 本地 price 用
// 用于模拟部署到本地网络获取 usd 价格

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@chainlink/contracts/src/v0.8/tests/MockV3Aggregator.sol";


Note：这里 MockV3Aggregator.sol 使用了和 FundMe 合约一样的 solidity 版本，如果版本不同的话编译会不过，解决方案如下：
编辑：hardhat.config.js 支持多版本 solidity
将 solidity: "0.8.17" 改为
solidity: {
  compilers: [{ version: "0.8.17" }, { version: "0.6.0" }], // 多个 solidity version 支持
},

参考链接：https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/tests/MockV3Aggregator.sol

Note：00-deploy-mocks.js 与 helper.hardhat-config.js 配合使用



$ yarn hardhat deploy --tags mocks


$ yarn hardhat deploy --tags all
输出如下：
Nothing to compile
mocks chainid= 31337
Local network detected! Deploying mocks...
deploying "MockV3Aggregator" (tx: 0x441bbc16160a496c67b04d6d2fd9dffd75b53418389f4bafc1a2f0ac55bd16ae)...: deployed at 0x663F3ad617193148711d28f5334eE4Ed07016602 with 694799 gas
Mocks deplyed!
------------------------------------------------------------------------
Hi! This is 01-deploy-fund-me
fund me chainid= 31337
fund me in hardhat
fund me ethUsdPriceFeedAddress= 0x663F3ad617193148711d28f5334eE4Ed07016602
deploying "FundMe" (tx: 0xe8b7b1b9a7236df42ba660a1bc83ece0911e32821f5397253c337d53b5db8473)...: deployed at 0x2E983A1Ba5e8b38AAAeC4B440B9dDcFBf72E15d1 with 888516 gas
------------------------------------------------------------------------
可以看出 mock.js 部署 MockV3Aggregator 的地址是 0x663F3ad617193148711d28f5334eE4Ed07016602
用于 01-deploy-fund-me.js 的hardhat 环境 ethUsdPriceFeedAddress 部署 


{% endhighlight %}

00-deploy-mocks.js 全部代码

{% highlight ruby %}  

// 用于部署 mock 环境

const { network } = require("hardhat")
const {
  developmentChains,
  DECIMALS,
  INITIAL_ANSWER,
} = require("../helper-hardhat-config")

module.exports = async ({ getNamedAccounts, deployments }) => {
  const { deploy, log } = deployments
  const { deployer } = await getNamedAccounts()
  const chainId = network.config.chainId

  console.log("mocks chainid=", chainId)

  if (developmentChains.includes(network.name)) {
    //if (chainId == 31337) {
    log("Local network detected! Deploying mocks...")
    await deploy("MockV3Aggregator", {
      contract: "MockV3Aggregator",
      from: deployer,
      log: true,
      // MockV3Aggregator constructor(uint8 _decimals, int256 _initialAnswer) {
      // DECIMALS、INITIAL_ANSWER 是 MockV3Aggregator 初始化参数
      args: [DECIMALS, INITIAL_ANSWER],
    })
    log("Mocks deplyed!")
    log(
      "------------------------------------------------------------------------"
    )
  }
}

// $ yarn hardhat deploy --tags 运行特殊标签的部署
// yarn hardhat deploy --tags mock

module.exports.tags = ["all", "mocks"]
{% endhighlight %}

### 5 部署到测试网 Goerli
{% highlight ruby %}  
1 在 contracts 同级创建 utils 目录
2 在 utils 目录创建 verify.js

3 部署到 goerli
$yarn hardhat deploy --network goerli

Note：部署失败可能是账户问题，检查 hardhat.config.js
namedAccounts: {
    deployer: {
      // 哪个账户进行部署
      default: 0,
      5: 0, // index = 0 用户进行部署在 Goerli 测试网, 将账户改成 0 要对应上
      31337: 2,
    },
    user: {
      default: 1,
    },
    
看到 deploying "FundMe" (tx: 0xbc77aeaee509a5bfea43a823848a6c9b8df34622e62e60e6dfc3048a1f0564de)... 
说明部署成功了
https://goerli.etherscan.io/address/0x95c32075B2264418e157eCA1c5A180Cdbcc49001
verify 因为连接超时导致失败了 😂
{% endhighlight %}

### 6 Solidity Style Guide (即代码规范)
参考链接： https://docs.soliditylang.org/en/v0.8.17/style-guide.html

### 7 Testing FundMe

{% highlight ruby %}  
1 cd test 目录
2 mkdir staging // 用于 staging 测试
3 mkdir unit    // 用于 unit 测试，合约最小化功能测试
{% endhighlight %}

####  Unit Test
* local hardhat
* forked hardhat
{% highlight ruby %}  
1 在 unit 目录创建 FundMe.test.js

2 $  yarn hardhat test
会输出  0 passing (2ms)

3 $ yarn hardhat test --grep "amount funded" 搜索 it 中有 ”amount funded“ 进行测试

4 $ yarn hardhat coverage 运行这个，现在哪些没有覆盖到测试

{% endhighlight %}

#### Get Gas
{% highlight ruby %}  
1 有些断言需要获得 gas 费，需要在调用完函数后打断点（比如在 transactionResponse.wait(1) 后）
2 新建 JavaScript 调试终端
3 运行 $ yarn hardhat test 即可触发断点
4 在 JavaScript 调试终端 输入 transactionReceipt，可以看到相关 transactionReceipt 信息
{% endhighlight %}

#### Console log for Solidity

{% highlight ruby %}  
1 在合约中 import：import "hardhat/console.sol";
2 比如在 constructor 中打 log：console.log("Log Owner address is %s", owner);
3 $ npx hardhat test 执行
可以看到 Log Owner address is 0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc
{% endhighlight %}


### 8 优化 Gas
* 全局变量的存储位置是 storage
* 一个全局变量相当于存储在一个 slot 中
* 每个 slot 是 32 bytes 长度，代表对象的 bytes 版本
* 数组或 Mapping 里的元素实际是使用 hashing 函数作为存储
* 数组实际在 slot 中存储的是它的长度，因为 32 bytes 可能不够存储整个数组的
* Mapping 在 slot 中存储是空白，Solidity 知道这是一个 Mapping
* 常量和 immutable 变量不占用存储空间，常量实际是合约代码本身（bytecode）一部分，实际常量是一个指针，常量不占用 slot

* opcodes 操作码，opcodes 中每一个都代表一小块 bytecode 中的内容，这些 opcodes 操作码代码机器做什么，实际它们需要很多计算工作，
运行我们的代码并用我们的代码做事，实际计算 gas 方式是购买这些 opcodes 操作码
* evm-opcodes 连接 https://github.com/crytic/evm-opcodes，SLOAD（加载）和 SSTORE（存储）是耗费 gas 比较大的两个操作
* 可以在 storage 变量前加 s_ 前缀声明，比如 s_funders
* immutable 变量前加 i_ 前缀声明

* storage 使用 private 或 internal 声明会更省 gas，然后提供一个 getter
{% highlight ruby %}  
  address private immutable i_owner;

  function getOwner() public view returns (address) {
    return i_owner;
  }
{% endhighlight %}

### 9 Staging Test
* staging 测试，比如部署到测试网或其他集成测试，staging 不总是需要，但真的很有用
* on a testnet
{% highlight ruby %}  
1 在 test/staging 目录下创建 FundMe.staging.test.js

2 $ yarn hardhat test --network goerli

3 staging test code

const { assert, expect } = require("chai")
const { deployments, ethers, getNamedAccounts, network } = require("hardhat")
const {
  isCallTrace,
} = require("hardhat/internal/hardhat-network/stack-traces/message-trace")
const { developmentChains } = require("../../helper-hardhat-config")

// developmentChains.includes(network.name) 用于判断是否是测试网不是就跳过
developmentChains.includes(network.name)
  ? describe.ship
  : describe("FundMe", async function () {
      let fundMe
      let deployer
      const sendValue = ethers.utils.parseEther("2")

      beforeEach(async function () {
        deployer = (await getNamedAccounts()).deployer
        fundMe = await ethers.getContract("FundMe", deployer)
        console.log("_deployer=", deployer)
        console.log("_deployer.getBalance=", deployer.getBalance)
      })

      it("Allows people to fund and withdraw", async function () {
        await fundMe.fund({ value: sendValue })
        await fundMe.withdraw()
        const endingBalance = await fundMe.provider.getBalance(fundMe.address)
        assert.equal(endingBalance.toString(), "0")
      })
    })
{% endhighlight %}

### 10 Running Scripts on a Local Node
{% highlight ruby %}  
1 在 scripts 目录创建 fund.js
2 $ yarn hardhat node 启动 hardhat 本地节点

3 $ yarn hardhat run scripts/fund.js --network localhost


4 在 scripts 目录创建 withdraw.js

5 $ yarn hardhat run scripts/withdraw.js --network localhost


fund.js 源码
const { getNamedAccounts, ethers } = require("hardhat")

async function main() {
  const { deployer } = await getNamedAccounts()
  const fundMe = await ethers.getContract("FundMe", deployer)
  console.log("Funding Contract...")
  const transactionResponse = await fundMe.fund({
    value: ethers.utils.parseEther("0.1"),
  })
  await transactionResponse.wait(1)
  console.log("Funding funded...")
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error)
    process.exit(1)
  })


withdraw.js 源码
const { getNamedAccounts, ethers } = require("hardhat")

async function main() {
  const { deployer } = await getNamedAccounts()
  const fundMe = await ethers.getContract("FundMe", deployer)
  console.log("Funding Contract...")
  const transactionResponse = await fundMe.withdraw()
  await transactionResponse.wait(1)
  console.log("Withdraw got it back...")
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error)
    process.exit(1)
  })

{% endhighlight %}

### 11 Add scripts to your package.json
编辑 package.json 添加 scripts key

{% highlight ruby %}    
    "scripts": {
    "test": "yarn hardhat test",  // 可以在终端直接 $yarn test
    "test:staging": "yarn hardhat test --network goerli",
    "lint": "yarn solhint 'contracts/*.sol'",
    "lint:fix": "yarn solhint 'contracts/*.sol' --fix",
    "format": "yarn prettier --write .",
    "coverage": "yarn hardhat coverage"
  }
  
  在 scripts 同级添加 
  "name":"hardhat-fund-me",
  "author": "hogetsu",
  "version": "1.0.0",
  
{% endhighlight %}


