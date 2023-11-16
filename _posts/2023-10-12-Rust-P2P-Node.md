---
layout: post
title: 使用 Async Rust 构建 P2P 节点
date: 2023-10-12 16:45:30.000000000 +09:00
categories: [Rust, Rust Async]
tags: [Rust, Rust Async]
---


# `P2P` 简介

## `P2P` 定义
* `P2P`：即 peer-to-peer
* `P2P` 是一种网络技术，可以在不同的计算机之间共享各种计算机资源，如 CPU、网络带宽和存储
* `P2P` 是当今用户在线常用的共享文件（如音乐、图像、和其他数字媒体）的一种方法
  * `Bittorrent 和 Gnutella` 是比较流行的文件共享 P2P 应用程序，
  * 比特币和以太坊等区块链网络也是 `P2P` 的
  * 它们不依赖中央服务器或中介来连接多个客户端
  * 最重要的是，它们利用用户的计算机作为客户端和服务器，从而将计算从中央服务器上卸载下来


* 传统的分布式系统使用 `Client-Server` 范式来部署的
* 而 `P2P` 是另外一种分布式系统
  * 在 `P2P` 中，一组节点（或叫对等点，Peer）彼此直接交互以共同提供公共服务，这样就无需中央协调器或管理员
  * `P2P` 系统中的每个节点（或叫 Peer）都可以充当客户端（从其他节点请求信息）或服务器（存储、检索数据并响应客户端请求执行必要的计算）
  * `P2P` 网络中的所有节点不必完全相同，有一个关键特征将 `Client-Server` 网络与 `P2P` 网络区分开来
    * 即在 `P2P` 网络中，缺乏具有唯一权限的专用服务器。
    * 在开放、无许可的 `P2P` 网络中，任何节点都可以决定提供与 `P2P` 节点相关的全部或部分服务集
  
## `P2P` 特点
* 与 `Client-Server` 网络相比，`P2P` 网络能够在其上构建不同类别的应用程序，这些程序是无许可、容错和抗审查的。
  * 无许可：因为数据和状态是跨多个节点复制的，所以没有服务器可以切断客户机对信息的访问
  * 容错性：因为没有单点故障，例如中央服务器
  * 抗审查：如区块链等网络
  * P2P 计算还可以更好的利用资源
  

![image](/assets/images/rust/peer.png)

## `P2P` 的复杂性
* `P2P` 系统的复杂性比 `Client-Server` 的系统要高
  * 传输：`P2P` 网络中的每个 `Peer` 都可以使用不同的协议，例如 `HTTP(s)、TCP、UDP` 等
  * 身份：每个 `Peer` 都需要知道其想要连接并发送消息的 `Peer` 的身份
  * 安全性：每个 `Peer` 都应该能够以安全的方式与其他 `Peer` 通信，而不存在第三方拦截或修改消息的风险等
  * 路由：每个 `Peer` 可以通过各种路由（例如数据包在 IP 协议中的分别方式）从其他 `Peer` 接收消息，这意味着如果消息不是针对自身的，则每个 `Peer` 都应该能够将消息路由到其他 `Peer`
  * 消息传递：`P2P` 网络应该能够发送点对点消息或组消息（组消息以发布/订阅模式发布的）
  
  
## `P2P` 的要求

### 对传输的要求
* `TCP/IP` 和 `UDP` 协议无处不在，在编写网络应用程序时非常流行。但还有其他更高级别的协议，例如 HTTP（TCP 上分层）和 QUIC（UDP 上分层）
* `P2P` 网络中的每个 `Peer` 都应该能够启动到另一个节点的连接，并且由于网络中 `Peer` 的多样性，能够通过多个协议监听传入的连接。

### `Peer` 身份的要求  
* 与 `web` 开发领域不同，在 `web` 开发领域中，服务器由唯一的域名标识（例如 www.rust-lang.org），然后使用域名服务将其解析为服务器的 IP 地址
* `P2P` 网络中的节点需要唯一身份，以便其他节点可以访问它们
* `P2P` 网络中的节点使用公钥和私钥对（非对称加密），与其他节点建立安全通信
  * `P2P` 网络中节点的身份称为 `PeerId`，是`节点公钥的加密散列`

### 对安全的要求
* 加密密钥对和 `PeerId` 使节点能够与它的 `Peers` 建立安全、经过身份验证的通信通道。但这只是安全的一个方面。
* 节点还需要实现授权框架，这个框架为哪个节点可以执行何种操作建立规则
* 还需要解决网络级的安全威胁：
  * 例如 `Sybil Attack`：即其他一个节点运营商利用不同身份启用大量节点，以获得网络中的优势地位
  * 或 `Eclipse Attack`：即其中一组恶意节点共谋以特定节点为目标，使后者无法到达任何合法节点

### 对路由的要求
* `P2P` 网络中的节点首先需要找到其他 `Peer` 才能进行通信。这是通过维护 `Peer` 路由表来实现的，这个表中包含对网络中其他 `Peer` 的引用
* 但是，在具有数千个或更多动态变化的节点（即节点加入和离开网络）的 P2P 网络中，任何单个节点都难以为网络中的所有节点维护完整而准确的路由表。
* 而 `Peer` 路由可以使节点能够将不是给自己准备的消息路由到目标节点

### 对消息的要求
* `P2P` 网络中的节点可以向特定节点发送消息，但也可以参与广播消息协议
  * 例如，发布/订阅模式，其中节点注册对特定主题的兴趣（订阅），而发送该主题消息的任何节点（发布）都由订阅该主题的所有节点接收
  * 这种技术通常用于将消息的内容传输到整个网络

### 对流多路复用的要求
* 流多路复用（Stream multiplexing）：即通过公共通信链路发送多个信息流的一种方法
* 在 `P2P` 情况下，它允许多个独立的 `逻辑流` 共享一个公共 `P2P` 传输层
  * 当考虑到一个节点与不同的 `Peers` 具有多个通信流的可能性，或者连个远程节点之间也可能存在多个并发连接的可能性时，这一点变的很重要
  * 流多路复用有助于优化 `Peer` 之间建立连接的开销


> 流多路复用在后端服务开发中很常见，其中客户端可以与服务器建立底层网络连接，然后通过底层网络连接多路复用不同的流（每个流有唯一的端口号）
{: .prompt-info }


# 构建

* 需要使用 `libp2p` 库，当然也可以不使用库，但是使用这个库会简单一些

## `libp2p` 库
* `libp2p` 是一个由协议、规范和库组成的模块化系统，它支持 `P2P` 应用程序开发
* 它目前支持三种语言：`JS、Go、Rust`
  * 未来将支持 `Hashell、Java、Python` 等
* 它被许多流行的项目使用，例如 `IPFS、Filecoin、Polkadot` 等

## `libp2p` 库的主要模块
* 传输模块（Transport）：负责一个 `Peer` 到另一个 `Peer` 的数据的实际传输和接收
* 身份模块（Identity）：`libp2p` 使用公钥密码（PKI）作为 `Peer` 节点身份的基础。使用加密算法为每个节点生产唯一的 `Peer id`
* 安全模块（Security）：节点使用其私钥对消息进行签名。节点之间的传输连接可以升级为安全的加密通道，以便远程 `Peer` 可以相互信任，并且没有第三方可以拦截它们之间的通信
* Peer 发现（Peer Discovery）：允许 `Peer` 在 `libp2p` 网络中查找并相互通信
* Peer 路由（Peer Routing）：使用其他 `Peer` 的知识信息来实现与 `Peer` 节点的通信
* 内容发现（Content Discovery）：在不知地哪些 `Peer` 节点拥有该内容的情况下，允许 `Peer` 节点从其他 `Peer` 节点获取部分内容
* 消息（Messaging）：允许向指定节点发送消息，也可以使用 `发布/订阅模式` 向一组 `Peer` 发送消息


## `P2P` 节点的身份

### 公钥和私钥
* 加密身份使用公钥基础设置（PKI），广泛用于为用户、设备和应用程序提供唯一身份，并保护端到端通信的安全
* 它的原理是创建两个不同的加密密钥，也称为由私钥和公钥组成的密钥对，它们之间具有数字关系
* 密钥对有着广泛的应用，比如在 `P2P` 网络中：
  * 节点使用密钥对彼此进行身份识别和身份验证
  * 公钥可以在网络中与其他人共享，但决不能泄露节点的私钥


> Note：需要安装 `cmake`，并把 `cmake` 的路径添加到 `PATH`
{: .prompt-info }


## 多地址（Multiaddresses）
* 在 `libp2p` 中，`Peer` 的身份在其整个生命周期内都是稳定且可验证的
* 但 `libp2p` 区分了 `Peer` 的身份和位置
  * `Peer` 的身份就是 `PeerId`
* `Peer` 的位置指可以到达对方的网络地址
  * 例如：可以通过 `TCP、websockets、QUIC` 或任何其他协议访问 `Peer`
  * `libp2p` 将这些网络地址编码成一个自描述格式，它叫做 `Multiaddresses`
  * 因此，在 `libp2p` 中，`Multiaddresses` 就表示 `Peer` 的位置
* 当 `P2P` 网络上的节点共享其联系信息时，它会发送一个包含网络地址和 `PeerId` 的多地址
* 节点的多地址的 `PeerId` 表示如下：
  * `/p2p/I2D3Kooxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
* 多地址的网络地址表示如下：
  * `ip4/192.157.1.23/tcp/1234`
* 节点的完整的多地址就是 `PeerId` 和网络地址的组合
  * `ip4/192.157.1.23/tcp/1234/p2p/I2D3Kooxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
  * 即网络地址+多地址


## `Swarm` 和网络行为
* `Swarm` 是 `libp2p` 中给定 `P2P` 节点内的网络管理器模块
* 它维护从给定节点到远程节点的所有活动和挂起连接，并管理已打开的所有子流状态

### `Swarm` 的结构和上下文环境
* `Swarm` 代表了一个低级接口，并提供了对 `libp2p` 网络的细粒度控制。 
* `Swarm` 是使用传输、网络行为和节点 `PeerId` 的组合构建的
* 传输（Transport）：会指明如何在网络上发送字节，而`Swarm` 中的网络行为会指明发送什么字节，发送给谁
  * 多个网络行为可以与单个运行节点相关联
  
  
![image](/assets/images/rust/swarm.png)
  

> Note：`libp2p` 网络中所有节点上运行的是同一套代码，这与 `Client-Server` 模式下的客户端与服务器具有不同的代码库是不同的
{: .prompt-info }


## Demo1：生成 `PeerId`


```rust
// main.rs

use libp2p::{identity, PeerId};
use libp2p::futures::StreamExt;
use libp2p::swarm::{DummyBehaviour, Swarm, SwarmEvent};
use std::error::Error;

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // 密钥对的类型是 25519
    let new_key = identity::Keypair::generate_ed25519();
    /*
        使用密钥对的公钥(new_key.public())生产 PeerId
        在 libp2p 中，公钥不会用来直接验证 Peer 的身份，
        我们使用的是它的一个 hash 的版本，也就是这个 peer_id
     */
    let new_peer_id = PeerId::from(new_key.public());
    println!("New Peer ID is {:?}", new_peer_id);

    // 创建一个空的网络行为，这个行为会关联到 swarm
    let behaviour = DummyBehaviour::default();
    // 创建一个传输
    let transport = libp2p::development_transport(new_key).await?;
    let mut swarm = Swarm::new(transport, behaviour, new_peer_id);
    // 让 swarm 监听 0.0.0.0 地址，端口是 0，
    // 0.0.0.0 表示本地机器上所有的 ipv4 的地址
    // 端口 0 指随机选一个端口
    swarm.listen_on("/ip4/0.0.0.0/tcp/0".parse()?)?;


    /*
        持续的轮询来检查事件
     */
    loop {
        match swarm.select_next_some().await {
            /*
                如果是 SwarmEvent::NewListenAddr 事件，
                即创建一个新的监听地址
             */
            SwarmEvent::NewListenAddr { address, .. } => {
                println!("Listening on Local Address {:?}", address);
            }
            _ => {}
        }
    }

}
```


```rust
// Cargo.toml

[package]
name = "p2p"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
libp2p = "0.46.1"
tokio = { version = "1.19.2", features = ["full"]}
```

* 测试：然后开两个终端运行 `carog run`，下边表示监听本地 63126 端口

```
New Peer ID is PeerId("12D3KooWHKUjLaSd4tSnvtKqhBP9S429nsjN8XoC5UWCkWFvdrHv")
Listening on Local Address "/ip4/127.0.0.1/tcp/63126"
Listening on Local Address "/ip4/192.168.50.227/tcp/63126"
```

## Demo2：在 `Peer` 节点之间交换 `ping` 命令

```rust
// main.rs

use libp2p::{identity, PeerId, Multiaddr};
use libp2p::futures::StreamExt;
use libp2p::swarm::{DummyBehaviour, Swarm, SwarmEvent};
use std::error::Error;
use libp2p::ping::{Ping, PingConfig};

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // 密钥对的类型是 25519
    let new_key = identity::Keypair::generate_ed25519();
    /*
        使用密钥对的公钥(new_key.public())生产 PeerId
        在 libp2p 中，公钥不会用来直接验证 Peer 的身份，
        我们使用的是它的一个 hash 的版本，也就是这个 peer_id
     */
    let new_peer_id = PeerId::from(new_key.public());
    println!("New Peer ID is {:?}", new_peer_id);

    // 创建一个空的网络行为，这个行为会关联到 swarm
    // let behaviour = DummyBehaviour::default();

    // 创建 ping 网络行为
    let behaviour = Ping::new(PingConfig::new().with_keep_alive(true));


    // 创建一个传输
    let transport = libp2p::development_transport(new_key).await?;
    let mut swarm = Swarm::new(transport, behaviour, new_peer_id);
    // 让 swarm 监听 0.0.0.0 地址，端口是 0，
    // 0.0.0.0 表示本地机器上所有的 ipv4 的地址
    // 端口 0 指随机选一个端口
    swarm.listen_on("/ip4/0.0.0.0/tcp/0".parse()?)?;

    // 
    /*
        本地向远程地址发出连接的代码
        远程地址是命令行输入的参数中取出的
     */
    if let Some(remote_peer) = std::env::args().nth(1) {
        let remote_peer_multiaddr: Multiaddr = remote_peer.parse()?;
        swarm.dial(remote_peer_multiaddr)?;
        println!("Dialed remote peer: {:?}", remote_peer);
    }

    /*
        持续的轮询来检查事件
     */
    loop {
        match swarm.select_next_some().await {
            /*
                如果是 SwarmEvent::NewListenAddr 事件，
                即创建一个新的监听地址
             */
            SwarmEvent::NewListenAddr { address, .. } => {
                println!("Listening on Local Address {:?}", address);
            }
            /*
                当本地节点发送 Ping 消息时，远程节点会返回 Pong，
                接收到 Pong 消息，会打印下边代码
             */
            SwarmEvent::Behaviour(event) => {
                println!("Event received from peer is {:?}", event);
            }
            _ => {}
        }
    }

}
```

* 测试：开一个终端运行 `carog run`，先把地址 `copy` 一下，例如 `/ip4/127.0.0.1/tcp/49469`
* 然后再开一个终端运行 `carog run /ip4/127.0.0.1/tcp/49469`


```
New Peer ID is PeerId("12D3KooWRnAHf1L7aww8KPocuf64BU4FoG18nFTQEpiR1pLPmhjY")
Dialed remote peer: "/ip4/127.0.0.1/tcp/49469"        // 对 这个地址进行拨号
Listening on Local Address "/ip4/127.0.0.1/tcp/50077"
Listening on Local Address "/ip4/192.168.50.227/tcp/50077"
Event received from peer is Event { peer: PeerId("12D3KooWMP4yRQs4iPyNSkRtct2EWJFzAzYBHcNnrR5SDaG5ThfP"), result: Ok(Pong) } // 接收到 Pong 消息
Event received from peer is Event { peer: PeerId("12D3KooWMP4yRQs4iPyNSkRtct2EWJFzAzYBHcNnrR5SDaG5ThfP"), result: Ok(Ping { rtt: 517.041µs }) }
Event received from peer is Event { peer: PeerId("12D3KooWMP4yRQs4iPyNSkRtct2EWJFzAzYBHcNnrR5SDaG5ThfP"), result: Ok(Pong) }
Event received from peer is Event { peer: PeerId("12D3KooWMP4yRQs4iPyNSkRtct2EWJFzAzYBHcNnrR5SDaG5ThfP"), result: Ok(Ping { rtt: 591.638µs }) }
```


## Demo3：发现 `Peer`
* `mDNS` 是由 RFC6762 定义的协议，它将主机名解析为 IP 地址
  * 在 `libp2p` 中，它用于发现网络上的其他节点
* 在 `libp2p` 中实现的 `mDNS` 网络行为，它会自动的发现本地网络上的其他 `libp2p` 节点


```rust
// main.rs
use libp2p::{
    futures::StreamExt,
    identity,
    mdns::{Mdns, MdnsConfig, MdnsEvent},
    swarm::{Swarm, SwarmEvent},
    PeerId,
};
use std::error::Error;


#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // 密钥对的类型是 25519
    let new_key = identity::Keypair::generate_ed25519();
    /*
        使用密钥对的公钥(new_key.public())生产 PeerId
        在 libp2p 中，公钥不会用来直接验证 Peer 的身份，
        我们使用的是它的一个 hash 的版本，也就是这个 peer_id
     */
    let new_peer_id = PeerId::from(new_key.public());
    println!("New Peer ID is {:?}", new_peer_id);

    // 创建一个传输
    let transport = libp2p::development_transport(new_key).await?;

    // for create peer id: 创建一个空的网络行为，这个行为会关联到 swarm
    // let behaviour = DummyBehaviour::default();

    // for ping: 创建 ping 网络行为
    // let behaviour = Ping::new(PingConfig::new().with_keep_alive(true));

    // for mdns
    let behaviour = Mdns::new(MdnsConfig::default()).await?;

    let mut swarm = Swarm::new(transport, behaviour, new_peer_id);
    // 让 swarm 监听 0.0.0.0 地址，端口是 0，
    // 0.0.0.0 表示本地机器上所有的 ipv4 的地址
    // 端口 0 指随机选一个端口
    swarm.listen_on("/ip4/0.0.0.0/tcp/0".parse()?)?;

    // 
    /*
        本地向远程地址发出连接的代码
        远程地址是命令行输入的参数中取出的
        only for ping

     */
    // if let Some(remote_peer) = std::env::args().nth(1) {
    //     let remote_peer_multiaddr: Multiaddr = remote_peer.parse()?;
    //     swarm.dial(remote_peer_multiaddr)?;
    //     println!("Dialed remote peer: {:?}", remote_peer);
    // }

    /*
        持续的轮询来检查事件
     */
    loop {
        match swarm.select_next_some().await {
            /*
                如果是 SwarmEvent::NewListenAddr 事件，
                即创建一个新的监听地址
             */
            SwarmEvent::NewListenAddr { address, .. } => {
                println!("Listening on Local Address {:?}", address);
            }
            /*
                当本地节点发送 Ping 消息时，远程节点会返回 Pong，
                接收到 Pong 消息，会打印下边代码
                only for ping

             */
            // SwarmEvent::Behaviour(event) => {
            //     println!("Event received from peer is {:?}", event);
            // }
            SwarmEvent::Behaviour(
                // 这个事件，表示发现 peer 了
                MdnsEvent::Discovered(peers)) => {
                    for (peer, addr) in peers {
                        println!("discovered peer={}, addr={}", peer, addr);
                    }
            }
            SwarmEvent::Behaviour(
                // 这个事件，表示过期了
                MdnsEvent::Expired(expired)) => {
                    for (peer, addr) in expired {
                        println!("expired peer={}, addr={}", peer, addr);
                    }
            }
            _ => {}
        }
    }

}
```


> 本次多开几个终端运行 `cargo run` 即可，不需要在命令后传入地址，因为它会自动发现节点
{: .prompt-info }


```
New Peer ID is PeerId("12D3KooWMTtqj2gye9d5HNLNMLMT7vw4yKBTerAsQieKQgyabjP2")
Listening on Local Address "/ip4/127.0.0.1/tcp/53946"
Listening on Local Address "/ip4/192.168.50.227/tcp/53946"
discovered peer=12D3KooWNehdbAVLuPkudvpzmmcJG9ydtQPSdqWRtA7oyh7uEDmM, addr=/ip4/192.168.50.227/tcp/53781
discovered peer=12D3KooWNehdbAVLuPkudvpzmmcJG9ydtQPSdqWRtA7oyh7uEDmM, addr=/ip4/127.0.0.1/tcp/53781
discovered peer=12D3KooWRQpxJqFcaMr5FgEDYX9LAUPsS8beMfaZmRLeSf3byXXm, addr=/ip4/192.168.50.227/tcp/53747
discovered peer=12D3KooWRQpxJqFcaMr5FgEDYX9LAUPsS8beMfaZmRLeSf3byXXm, addr=/ip4/127.0.0.1/tcp/53747
discovered peer=12D3KooWJwLBj2Bw6ev6PBJP2FGpdj1ehCBm4K2n7mjUMC96Rqqa, addr=/ip4/192.168.50.227/tcp/53854
discovered peer=12D3KooWJwLBj2Bw6ev6PBJP2FGpdj1ehCBm4K2n7mjUMC96Rqqa, addr=/ip4/127.0.0.1/tcp/53854
```

* 更多教程可以查阅 `libp2p` 官方文档，或购买相关书籍
