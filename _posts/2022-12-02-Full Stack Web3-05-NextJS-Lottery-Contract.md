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
$ yarn add moralis react-moralis
```
安装 moralis 后显示的依赖

```
  "dependencies": {
    "eslint": "8.29.0",
    "eslint-config-next": "13.0.6",
    "moralis": "^2.9.0",
    "next": "13.0.6",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "react-moralis": "^1.4.2"
  }
```
## React Hooks
