---
layout: post
title: NextJS-Lottery-Contract
date: 2022-12-02 16:45:30.000000000 +09:00
categories: [区块链, Chainlink 公开课]
tags: [区块链, Chainlink 公开课]
---

# NextJS-Lottery-Contract ( Lesson 10)

## 简介
Next.js 是一个构建在 React 之上的轻型框架

[6 Ways to connect your dapp to a wallet](https://www.youtube.com/watch?v=pdsYCkUWrgQ)

## NextJS Setup

### 创建工程

``` console
1 $ mkdir nextjs-contract-lottery
2 $ cd nextjs-contract-lottery
3 $ code .
4 $ yarn create next-app .
```

### 结构介绍
创建完 nextjs 后会多出一些目录结构
1. pages 目录是页面相关
* _app.js 就是程序入口
* api 目录就是一些 http 请求等，这里不会使用，先忽略
2. public 目录是图片
3. styles 目录存放 css 样式文件
4. next.config.js 配置相关

```console
$ yarn run dev 运行网站

5 $ yarn add --dev prettier
```

如果你已经运行 `yarn run dev` 开启一个网站如 `http://localhost:3000/`，可以在 `yarn dev` 开启下一个 `http://localhost:3000/`，
这是因为 packages.json 中 script 中预先配置了一些脚本，如 `"dev": "next dev"`

## Manual Header
创建 components 目录，然后在里边创建 ManualHeader.js 或 ManualHeader.jsx。
[参考链接](https://www.w3schools.com/react/react_components.asp)

```console
$ yarn add moralis react-moralis moralis-v1
```
安装 moralis 后显示的依赖

```
  "dependencies": {
    "eslint": "8.29.0",
    "eslint-config-next": "13.0.6",
    "moralis": "^2.9.0",
    "moralis-v1": "^1.12.0",
    "next": "13.0.6",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "react-moralis": "^1.4.2"
  }
```
## React Hooks
hook 允许函数组件访问状态和 react 功能。[hooks-overview]https://reactjs.org/docs/hooks-overview.html

```react
import { useMoralis } from "react-moralis"

export default function ManualHeader() {
    //
    /**
     * useMoralis 就是 react hook
     * 一种跟踪状态的方法
     */
    const { enableWeb3, account } = useMoralis()
    // some button that connects us and changes connected to be true
    // await enableWeb3() 连接钱包
    return (
        <div>
            {/* 没有账户显示连接按钮，否则显示已连接 */}
            {/* address 显示为 0xec49...ad6d */}
            {/* 切换账户会自动显示 */}
            {account ? (
                <div>
                    Connected to {account.slice(0, 6)}...{account.slice(account.length - 4)}
                </div>
            ) : (
                <button
                    onClick={async () => {
                        await enableWeb3()
                    }}
                >
                    Connect
                </button>
            )}
        </div>
    )
}
```


## useEffect Hook
useEffect 类似观察者模式

```react
    const { enableWeb3, account, isWeb3Enabled } = useMoralis()
    // some button that connects us and changes connected to be true

    /**
     * useEffect
     * 第一个参数是一个函数
     * 
     * 第二个参数是可选的，一个依赖数组
     * case1: useEffect 会检查依赖数组中的值，一旦数组中的值由变化就调用参数一的函数
     * case2: 这里不需要给数组也行
     * 如果不传依赖数组，它会随时重新渲染
     * 你要小心，因为你会获得 circular render
     * case3: 如果依赖数组是一个空数组，它只会在加载时运行一次，只运行一次
     */
    // automatically run on load
    // then, it'll run checking the value
    useEffect(() => {
        console.log("HI!")
        console.log("isWeb3Enabled is", isWeb3Enabled)
    }, [isWeb3Enabled])
```
## Local Storage
设置注入设置项，记录最近的交互。连接钱包，查看 application，local storage 能看到 connected inject
```
if (typeof window !== "undefined") {
  window.localStorage.setItem("connected", "injected")
}                      
```

## web3uikit
1. 在 components 目录创建 Header.js
2. 安装 [web3uikit](https://github.com/web3ui/web3uikit)


```console
$  yarn add web3uikit
```

Header.js 代码，此时可替换 ManuelHeader 为 Header
```react
import { ConnectButton } from "web3uikit"

export default function Header() {
    return (
        <div>
            {/* // moralisAuth false 不连接服务器 */}
            <ConnectButton moralisAuth={false} />
        </div>
    )
}
```

## Calling Functions in NextJS
[MoralisWeb3](https://github.com/MoralisWeb3/react-moralis#useweb3contract)

### 设置 abi.json 和 Addresses.json
创建 constants 目录，创建 abi.json 和 contractAddresses.json，先设置好 contractAddresses.json 内容
```json
{ "5": [], "31337": [] }

```

在 hh-contract-lottery-fcc 里 deploy 目录创建 99-update-front-end.js
```javascript
const { ethers, network } = require("hardhat")
const fs = require("fs")

const FRONT_END_ADDRESSES_FILE = "../nextjs-contract-lottery/constants/contractAddresses.json"
const FRONT_END_ABI_FILE = "../nextjs-contract-lottery/constants/abi.json"

module.exports = async function () {
    if (process.env.UPDATE_FRONT_END) {
        console.log("Updating front end...")
        await updateContractAddresses()
        await updateAbi()
    }
}

async function updateAbi() {
    const raffle = await ethers.getContract("Raffle")
    fs.writeFileSync(FRONT_END_ABI_FILE, raffle.interface.format(ethers.utils.FormatTypes.json))
}

async function updateContractAddresses() {
    const raffle = await ethers.getContract("Raffle")
    const chainId = network.config.chainId.toString()

    let addresses = fs.readFileSync(FRONT_END_ADDRESSES_FILE, "utf8")
    console.log(`addresses ${addresses}`)
    let contractAddresses = JSON.parse(addresses)

    if (chainId in contractAddresses) {
        if (!contractAddresses[chainId].includes(raffle.address)) {
            contractAddresses[chainId].push(raffle.address)
        }
    } else {
        contractAddresses[chainId] = [raffle.address]
    }
    fs.writeFileSync(FRONT_END_ADDRESSES_FILE, JSON.stringify(contractAddresses))
}

module.exports.tags = ["all", "frontend"]
```

执行 $ hh deploy 或 hh node 写入 abi.json 和 Addresses.json

```
$ cd hh-contract-lottery-fcc 
$ hh node // 开启 hardhat node
```

LotteryEntrance 代码

```
// have a function to call the lottery
import { useWeb3Contract } from "react-moralis"
import { contractAddresses, abi } from "../constants"
import { useMoralis } from "react-moralis"
import { useEffect } from "react"

export default function LotteryEntrance() {
    const { chainId: chainIdHex, isWeb3Enabled } = useMoralis()
    const chainId = parseInt(chainIdHex)
    console.log(chainId)

    const raffleAddress = chainId in contractAddresses ? contractAddresses[chainId][0] : null
    // 这是我们发送交易的方式
    // 运行函数可以同时交易和读取状态
    // 可以发送交易也可以发送函数
    const { runContractFunction: getEntranceFee } = useWeb3Contract({
        abi: abi,
        contractAddress: raffleAddress, // specify the networkId
        functionName: "getEntranceFee",
        params: {},
    })

    useEffect(() => {
        if (isWeb3Enabled) {
            // try to read the raffle entrance fee
            async function updateUI() {
                const entranceFee = await getEntranceFee()
                console.log(`fee is ${entranceFee}`)
            }
            updateUI()
        }
    }, [isWeb3Enabled])

    return <div>Hi from LotteryEntrance!</div>
}
```

> fee is 有可能是 undefined 注意切换本地网络还是测试网
{: .prompt-info }

## useState
entranceFee 可以作为一个 state 常量，有点类似提供一个属性

[Using the State Hook](https://reactjs.org/docs/hooks-state.html)


```react
import { useEffect, useState } from "react"

    // use State for entranceFee
    // useState("0") 设置初始值为 0
    const [entranceFee, setEntranceFee] = useState("0")
```

## Calling Functions in NextJS

```react
const { runContractFunction: enterRaffle } = useWeb3Contract({
    abi: abi,
    contractAddress: raffleAddress, // specify the networkId
    functionName: "enterRaffle",
    params: {},
    msgValue: entranceFee,
})

return (
    <div>
        Hi from LotteryEntrance!
        {raffleAddress ? (
            <div>
                <button
                    onClick={async () => {
                        await enterRaffle()
                    }}
                >
                    Enter Raffle
                </button>
                Entrance Fee is: {ethers.utils.formatUnits(entranceFee, "ether")} ETH
            </div>
        ) : (
            <div>No Raffle Address Deteched</div>
        )}
    </div>
)
```

## useNotification
[notification-hook](https://web3ui.github.io/web3uikit/?path=/docs/5-popup-notification--hook-demo)

编辑 _app.js

```react
import "../styles/globals.css"
import { MoralisProvider } from "react-moralis"
import { NotificationProvider } from "web3uikit"

// _app.js 就是程序入口
function MyApp({ Component, pageProps }) {
    return (
        /**
         * 程序需要被 MoralisProvider 包裹起来，即所谓的 react 提供者
         * 这将成为我们的 context provider
         *
         * initializeOnMount 是可选 hook，进入服务器以向我们网站添加更多功能
         * 这里我们不想链接服务器，不需要额外功能
         *
         * MoralisProvider 就是 hook，
         * hook 允许函数组件访问状态和 react 功能
         */
        <MoralisProvider initializeOnMount={false}>
            <NotificationProvider>
                <Component {...pageProps} />
            </NotificationProvider>
        </MoralisProvider>
    )
}

export default MyApp
```

LotteryEntrance.js

```react
const dispatch = useNotification()

const handleNewNotification = async (tx) => {
    dispatch({
        type: "info",
        message: "Transaction Complete!",
        title: "Tx Notification",
        position: "topR",
        icon: "bell",
    })
}
    
<button
    onClick={async () => {
        await enterRaffle({
            onSuccess: handleSuccess,
            onError: (error) => {
                console.log(error)
            },
        })
    }}
>
    Enter Raffle
</button>
```

## Reading & Displaying Contract Data

### 测试本地环境网站是否正常
1 启动 localhost 本地节点

```console
$ hh node
```
2 MetaMask 设置-高级-重置账户
3 刷新网站 UI
4 脚本 mock recentWinner

```console
$ yarn hardhat run script/mockOffchain.js --network localhost
```
5 刷新网站 UI 查看 recentWinner


> 如果运行网站没有对应合约连接的话，调用合约代码如`(await getEntranceFee()).toString()`可能出现 TypeError，通常这种情况出现在 MetaMask 切网后或合约对应不上
{: .prompt-danger }


## Tailwind Style CSS

[Tailwind](https://tailwindcss.com/docs/guides/nextjs)

安装 Tailwind

```console
$ yarn add --dev tailwindcss postcss autoprefixer
$ yarn tailwindcss init -p
```

初始化后会创建 tailwind 的配置文件 `tailwind.config.js` 和 `postcss.config.js` 文件

copy 代码到 `globals.css` 覆盖其当前内容，全局 css 使用了 tailwind

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```
然后 VSCode 安装插件 `postcss` 和 `Tailwind CSS IntelliSense`

tailwind 可以允许我们在 `div` 中将所有内容设置为类名

```react
<div className="p-5 border-b-2 flex flex-row">
```

## Hosting your Site
[Vercel](https://vercel.com/?utm_source=create-next-app&utm_medium=default-template&utm_campaign=create-next-app)

### 1 IPFS
它是分布式去中心化数据结构，类似区块链，但没有 mining，你可以向其添加数据。用于存储和访问数据
实际 IPFS 做的第一件事是，它对我们的数据进行哈希处理以获得唯一哈希，仅指向该数据，你可以将所有这些编码为一个哈希函数，ipfs 节点为你做哈希，并且全球 ipfs 节点有完全相同的哈希函数

IPFS 只能做去中心化存储，不能执行

#### Hosting on IPFS

[IPFS](https://ipfs.tech/#install)

安装 IPFS.dmg 然后导入文件，选择 nextjs-contract-lottery 下的 next.config.js。copy next.config.js 的 CID。

安装 IPFS Companion（Chrome 插件），然后访问 ipfs://next.config.js 的 CID。如果没有安装 IPFS Companion 可以访问 https://ipfs.tech/ipfs/ next.config.js 的 CID

![image](/assets/web3-full/ipfs-01.png)


将网站部署到 IPFS 上

```console
$ yarn build
```


○  (Static)  automatically rendered as static HTML (uses no initial props)

检查 build 出来的网站是否有问题

```console
$ yarn next export
```

如果报错的话编辑 next.config.js，添加 images.unoptimized = true，然后重新 yarn build && yarn next export
```javascript
const nextConfig = {
    reactStrictMode: true,
    images: {
        unoptimized: true,
    },
}
```

然后在 IPFS 导入文件夹，选择 out 文件夹，设置 out pin 到本地节点进行测试

![image](/assets/web3-full/ipfs-02.png)

然后访问 ipfs://out 的 CID，即可看到网站

## Hosting on IPFS & Filecoin using Fleek
[Fleek](https://app.fleek.co/#/auth/sign-up?method=Git+Provider&provider=GitHub) 很容易部署网站和 app 到新的开放网络，通过 github 注册 Fleek

点击 add new site
![image](/assets/web3-full/fleek-01.png)

![image](/assets/web3-full/fleek-02.png)

然后在 github 创建一个 nextjs-contract-lottery-fcc 仓库，将网站上传到 github，然后点击 Fleek 的 connect github，
选择 Only select repositories 选择 nextjs-contract-lottery-fcc，选择安装和授权

![image](/assets/web3-full/fleek-03.png)

> 部署命令是 `yarn install && yarn run build && yarn next export`
{: .prompt-danger }

Filecoin 就是固定你的数据，并使用去中心化存储来做到这一点

部署成功后
![image](/assets/web3-full/fleek-04.png)

也可以使用 ipfs://Current IPFS Hash，进行访问

## Filecoin Overview
ipfs 必须让人 pin 你的数据才行，filecoin 是一个专门的区块链保存数据分布式与持久性。ipfs 提供内容的寻址，filecoin 提供持久性存储。
ipfs 很像 http 的一种协议，filecoin 是一个区块链。所以 ipfs 与存储层无关，它需要和你选择的存储层配合使用。你也可以把 ipfs 数据存储在云端

![image](/assets/web3-full/filcoin-01.png)
![image](/assets/web3-full/filcoin-02.png)
![image](/assets/web3-full/filcoin-03.png)
![image](/assets/web3-full/filcoin-04.png)
![image](/assets/web3-full/filcoin-05.png)

### ipfs & filecoin 开发工具

Fleek 
![image](/assets/web3-full/filcoin-06.png)


NFT.Storage 用于存储 NFT 数据，是免费的
![image](/assets/web3-full/filcoin-07.png)

web3.Storage 提供类似于 NFT 存储，使用起来非常方便，它还有 JavaScript 和 Go 客户端库
![image](/assets/web3-full/filcoin-08.png)


Textile 更灵活的，提供一些连接多个 Layer 1 的桥
![image](/assets/web3-full/filcoin-09.png)

Estuary.tech，目前处于 alpha 阶段，需要邀请才行
![image](/assets/web3-full/filcoin-10.png)

OrbitDB
![image](/assets/web3-full/filcoin-11.png)


![image](/assets/web3-full/filcoin-12.png)

一些教程
![image](/assets/web3-full/filcoin-13.png)
