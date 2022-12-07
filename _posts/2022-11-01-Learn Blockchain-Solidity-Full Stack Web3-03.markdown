---
layout: post
title: Learn Blockchain-Solidity-Full Stack Web3-03
date: 2022-11-01 16:45:30.000000000 +09:00
tag: 区块链
---

## 0x01 HTML/Javascript Fund Me ( Lesson 8)

本节课制作 FundMe 合约的前端
Full Stack = Smart Contracts（backend）+ HTML / Javascript / Website stuff (front e   nd)

### 1 创建目录
{% highlight ruby %} 
1 $ mkdir html-fund-me-fcc
2 $ cd html-fund-me-fcc
3 $ code .
{% endhighlight %}

### 2 Setup HTML
{% highlight ruby %} 
1 创建 index.html，然后输入 ! 回车，会自动填充默认代码
2 VSCode 插件搜索 Live Server，并安装，安装后 VSCode 底部应该有个 Go Live
3 Command + Shift + P，搜索 Live Server，选择 Open With Live Server（或点击底部 Go Live 按钮），就打开了 index.html，
实际是托管在了 http://127.0.0.1:5500/index.html

4 安装 http-server
$yarn add --dev http-server

5 $ yarn http-server
会显示 http://127.0.0.1:8080/，在浏览器里输入 http://127.0.0.1:8080/ 会看到跟 Go Live 一样的结果

{% endhighlight %}

### 3 Connecting HTML to Metamask
{% highlight ruby %} 
  <body>
    HH!
    <script>
      async function connect() {
        if (typeof window.ethereum !== "undefined") {
          console.log("Metamask has installed! ");
          // 连接 metamask 钱包，浏览器弹出钱包
          await window.ethereum.request({ method: "eth_requestAccounts" });
          console.log("Metamask has connected! ");
          document.getElementById("connectBtn").innerHTML = "Connected!";
        } else {
          console.log("Metamask absent");
          document.getElementById("connectBtn").innerHTML =
            "Please install Metamask!";
        }
      }
    </script>
    <button id="connectBtn" onclick="connect()">Connect</button>
  </body>
{% endhighlight %}

### 4 Javascript in its own files
{% highlight ruby %} 
1 创建 index.js

2 把 connect 函数复制到 index.js

3 重新连接 Metamask 试试
{% endhighlight %}

### 5 ES6（Front End JS）VS NodeJS
{% highlight ruby %} 

1 $yarn add --dev prettier 安装 prettier


2 新建文件 ethers-5.1.esm.min.js
参考 doc: https://docs.ethers.io/v5/getting-started/
操作：copy https://cdn.ethers.io/lib/ethers-5.1.esm.min.js 全部代码到 ethers-5.1.esm.min.js

3 use import 
import { ethers } from "./ethers-5.1.esm.min.js"

const connectBtn = document.getElementById("connectBtn")
const fundBtn = document.getElementById("fundBtn")

connectBtn.onclick = connect
fundBtn.onclick = fund
console.log(`connectBtn is ${connectBtn}`)
console.log("ethers is", ethers)
async function connect() {
  if (typeof window.ethereum !== "undefined") {
    console.log("Metamask has installed! ")
    // 连接 metamask 钱包，浏览器弹出钱包
    await window.ethereum.request({ method: "eth_requestAccounts" })
    console.log("Metamask has connected! ")
    connectBtn.innerHTML = "Connected"
  } else {
    console.log("Metamask absent")
    connectBtn.innerHTML = "Please install Metamask!"
  }
}

{% endhighlight %}

### 6 Sending a Transaction From a Website
{% highlight ruby %} 

1 创建 constants.js 用于放合约 ABI 或地址等

2 找到 FundMe 工程，/artifacts/contracts/FundMe.sol/FundMe.json 里找到 abi，
复制其内容把他放到 constants.js 中

3 cd hardhat-fundme 目录，运行 $ yarn hardhat node 开启本地 node

4 Metamask 添加新网络，RPC-URL=http://127.0.0.1:8545/，chainID=31337

5 将 本地启动的 node 的账户私钥导入 Metamask
或者也可以使用私钥生成 JSON，将 JSON 导入 Metamask

代码
import { abi, contractAddress } from "./constants.js"

async function fund() {
  const ethAmout = "100"
  console.log(`Funding with ${ethAmout}...`)
  if (typeof window.ethereum !== "undefined") {
    // 总是需要 provide / connect to the blockchain
    // signer / wallet / someone with some gas
    // contract that we are interacting with ^ ABI & Address

    // web3Provider 与之前的 JsonRpcProvider 很像
    // web3Provider 使用 http 端点，并自动连接以太坊
    // provider 会连接到 metamask
    const provider = new ethers.providers.Web3Provider(window.ethereum)
    const signer = provider.getSigner()
    console.log(signer)
    // 获取 ABI 来签名合约
    const contract = new ethers.Contract(contractAddress, abi, signer)
    try {
      const transactionResponse = await contract.fund({
        value: ethers.utils.parseEther(ethAmout),
      })
    } catch (error) {
      console.log(error)
    }
  }
}
{% endhighlight %}

### 7 Resetting an Account in Metamask
对于测试账户出现问题，可以在 Metamask 设置->高级->重置账户。对于本地节点这样是可以的
{% highlight ruby %} 
async function fund() {
  const ethAmout = "100"
  console.log(`Funding with ${ethAmout}...`)
  if (typeof window.ethereum !== "undefined") {
    // 总是需要 provide / connect to the blockchain
    // signer / wallet / someone with some gas
    // contract that we are interacting with ^ ABI & Address

    // web3Provider 与之前的 JsonRpcProvider 很像
    // web3Provider 使用 http 端点，并自动连接以太坊
    // provider 会连接到 metamask
    const provider = new ethers.providers.Web3Provider(window.ethereum)
    const signer = provider.getSigner()
    console.log(signer)
    // 获取 ABI 来签名合约
    const contract = new ethers.Contract(contractAddress, abi, signer)
    try {
      const transactionResponse = await contract.fund({
        value: ethers.utils.parseEther(ethAmout),
      })
      // listen for the tx to be mined
      // listen for an event we haven't learned about yet!

      // wait for this TX to finish
      // https://docs.ethers.io/v5/search/?search=once
      await listenForTransactionMine(transactionResponse, provider)
      console.log("Fund done!")
    } catch (error) {
      console.log(error)
    }
  }
}

function listenForTransactionMine(transactionResponse, provider) {
  console.log(`Mining ${transactionResponse.hash}`)
  // listen TX to finish
  // wait listen finished need return Promise
  return new Promise((resolve, reject) => {
    provider.once(transactionResponse.hash, (transactionReceipt) => {
      console.log(
        `Completed with ${transactionReceipt.confirmations} confirmations`
      )
      resolve()
    })
  })
}
{% endhighlight %}


### 8 Reading from the Blockchain
{% highlight ruby %} 
async function getBalance() {
  console.log("getBalance")
  if (typeof window.ethereum !== "undefined") {
    const provider = new ethers.providers.Web3Provider(window.ethereum)
    const balance = await provider.getBalance(contractAddress)
    console.log("balance is ", ethers.utils.formatEther(balance))
  }
}
{% endhighlight %}

### 9 本节全部代码
HTML
{% highlight ruby %} 
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Fund Me DApp</title>
  </head>
  <body>
    Fund Me DApp
    <!-- use require -->
    <!-- <script src="./index.js" type="text/javascript"></script> -->
    <!-- ES6 or TypeScript 需要改成 module，用 import -->
    <script src="./index.js" type="module"></script>

    <!-- not use module 这样写 -->
    <!-- <button id="connectBtn" onclick="connect()">Connect</button>
    <button id="fundBtn" onclick="fund()">Fund</button> -->

    <button id="connectBtn">Connect</button>
    <button id="balanceBtn">getBalance</button>
    <button id="withdrawBtn">withdraw</button>

    <!-- form -->
    <label for="fund">ETH Amount</label>
    <input id="ethAmount" placeholder="0.1" />
    <button id="fundBtn">Fund</button>
  </body>
</html>

{% endhighlight %}

JavaScript
{% highlight ruby %} 
// in nodejs
// use require()

// in front-end javascript you can't use require
// instead of import

import { ethers } from "./ethers-5.1.esm.min.js"
import { abi, contractAddress } from "./constants.js"

const connectBtn = document.getElementById("connectBtn")
const fundBtn = document.getElementById("fundBtn")
const balanceBtn = document.getElementById("balanceBtn")
const withdrawBtn = document.getElementById("withdrawBtn")

connectBtn.onclick = connect
fundBtn.onclick = fund
balanceBtn.onclick = getBalance
withdrawBtn.onclick = withdraw

console.log(`connectBtn is ${connectBtn}`)
console.log("ethers is", ethers)

async function connect() {
  if (typeof window.ethereum !== "undefined") {
    console.log("Metamask has installed! ")
    // 连接 metamask 钱包，浏览器弹出钱包
    await window.ethereum.request({ method: "eth_requestAccounts" })

    console.log("Metamask has connected! ")
    //document.getElementById("connectBtn").innerHTML = "Connected!"
    connectBtn.innerHTML = "Connected"
  } else {
    console.log("Metamask absent")
    //document.getElementById("connectBtn").innerHTML = "Please install Metamask!"
    connectBtn.innerHTML = "Please install Metamask!"
  }
}

async function getBalance() {
  console.log("getBalance")
  if (typeof window.ethereum !== "undefined") {
    const provider = new ethers.providers.Web3Provider(window.ethereum)
    const balance = await provider.getBalance(contractAddress)
    console.log("balance is ", ethers.utils.formatEther(balance))
  }
}

// fund function
async function fund() {
  const ethAmout = document.getElementById("ethAmount").value
  console.log(`Funding with ${ethAmout}...`)
  if (typeof window.ethereum !== "undefined") {
    // 总是需要 provide / connect to the blockchain
    // signer / wallet / someone with some gas
    // contract that we are interacting with ^ ABI & Address

    // web3Provider 与之前的 JsonRpcProvider 很像
    // web3Provider 使用 http 端点，并自动连接以太坊
    // provider 会连接到 metamask
    const provider = new ethers.providers.Web3Provider(window.ethereum)
    const signer = provider.getSigner()
    console.log(signer)
    // 获取 ABI 来签名合约
    const contract = new ethers.Contract(contractAddress, abi, signer)
    try {
      const transactionResponse = await contract.fund({
        value: ethers.utils.parseEther(ethAmout),
      })
      // listen for the tx to be mined
      // listen for an event we haven't learned about yet!

      // wait for this TX to finish
      // https://docs.ethers.io/v5/search/?search=once
      await listenForTransactionMine(transactionResponse, provider)
      console.log("Fund done!")
    } catch (error) {
      console.log(error)
    }
  }
}

function listenForTransactionMine(transactionResponse, provider) {
  console.log(`Mining ${transactionResponse.hash}`)
  // listen TX to finish
  // return new Promise
  // wait listen finished need return Promise
  return new Promise((resolve, reject) => {
    provider.once(transactionResponse.hash, (transactionReceipt) => {
      console.log(
        `Completed with ${transactionReceipt.confirmations} confirmations`
      )
      resolve()
    })
  })
}

// withdraw
async function withdraw() {
  if (typeof window.ethereum !== "undefined") {
    console.log("Withdraw....")
    const provider = new ethers.providers.Web3Provider(window.ethereum)
    const signer = provider.getSigner()
    const contract = new ethers.Contract(contractAddress, abi, signer)
    try {
      const transactionResponse = await contract.withdraw()
      await listenForTransactionMine(transactionResponse, provider)
      console.log("Withdraw done!")
    } catch (error) {
      console.log(error)
    }
  }
}

{% endhighlight %}
