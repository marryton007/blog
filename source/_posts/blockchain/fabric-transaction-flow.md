---
layout: post
title: "hyperledger fabric交易流程"
date: 2017-08-24 14:17:39
categories: blockchain
tags:
  - blockchain
---

在这篇文章中，我将阐述在 fabric 网络中资产交易时的底层机制。在这个场景中，有 2 个客户，A 和 B，他们想要在区块链网络中买卖萝卜，他们每人都拥有 1 个节点来发送交易或是与区块链进行交互。
![step0](http://hyperledger-fabric.readthedocs.io/en/latest/_images/step0.png)

1. ### 前提

   在开始之前，我们作如下假设：

   - 区块链通道已经建立并已经运行
   - 用户已经在链上注册并通过 CA 认证
   - 智能合约已经安装到节点上并且已经完成初始化，该智能合约包括一些有关萝卜交易市场的初始状态，一系统关于萝卜的交易指令和萝卜成交价格的定义等。
   - 交易的背书策略已经定义好，这里要求所有的交易必须经过 A 和 B 的共同背书

   ![step1](http://hyperledger-fabric.readthedocs.io/en/latest/_images/step1.png)

2. ### 客户 A 发起交易请求

   发生了什么事？客户 A 发起一个购买萝卜请求，这个请求会到达到节点 A 和节点 B，这 2 个节点分别属于客户 A 和客户 B，背书策略要求 2 个节点一起完成每笔交易的背书，所以这个请求先送到节点 A 和节点 B。

   更进一步，一个'交易预案'将被构造出来，如上图，客户 A 通过 SDK(Node 版、Java 版、Python 版)来构造这个'交易预案'，这个预案通过调用智能合约中的函数对账本进行读写。SDK 在这个过程中充当一个'垫片'的角色，它将预案以合适的格式打包并使用用户的加密证书给预案签名。

   ![step2](http://hyperledger-fabric.readthedocs.io/en/latest/_images/step2.png)

3. ### 背书节点验证

   背书节点收到'交易预案'后，需要作如下验证工作：

   - 交易预案是完好的
   - 该预案以前没有提交过(防止[重放攻击][reply-attack])
   - 签名是合法的
   - 交易发起者(客户 A)是否满足区块链写策略

   满足以上要求后，背书节点把'交易预案'作为输入参数，调用智能合约中的函数，智能合约根据当前的账本状态计算出一个'交易结果',该结果包括返回值，读写集。此时，区块链账本并不会被更新。'交易结果'在被签名后与一个是/否的背书结果一同返回，称之为'预案回复'。

   {MSP 是一个节点组件允许验证从客户商过来的交易请求，且对'交易结果'进行签名。区块链的写策略是在通过建立时定义的，它决定谁有资格向通道发起交易请求}

   ![step3](http://hyperledger-fabric.readthedocs.io/en/latest/_images/step3.png)

4. ### 剖析预案回复

   SDK 通过验证背书节点的签名且比较'预案回复'来作决定：

   - '预案回复'是否一致
   - 背书策略是否满足

   ![step4](http://hyperledger-fabric.readthedocs.io/en/latest/_images/step4.png)

5. ### 集合背书成交易

   SDK 将'交易预案'和'预案回复'组合成'交易消息'，并将其广播到排序服务，'交易消息'包含读写集，背书节点签名和智能合约 ID。排序服务并不关心'交易内容'，它只是简单的接收所有网络上所有通道的'交易消息'，分通道对'交易消息'按时间排序，并按通道将交易打包成块。
   ![step5](http://hyperledger-fabric.readthedocs.io/en/latest/_images/step5.png)

6. ###　交易验证
   块会被运送到通道中的所有节点，块中的交易会被验证是否满足背书策略，且要确保交易的中的读集合在交易执行以来账本没做修改，块中的交易都会被上有效/无效的标签。
   ![step6](http://hyperledger-fabric.readthedocs.io/en/latest/_images/step6.png)

7. ### 账本更新

   通道中每个节点将块追加到自身账本后，并且确认交易中的写集合被提交自身的状态数据库中。在这之后，一个事件会被触发，以通知 SDK 交易已被永久地追加到账本上。

8. ### 资料参考

   - [官方参考][flow]

[flow]: http://hyperledger-fabric.readthedocs.io/en/latest/txflow.html
[reply-attack]: https://zh.wikipedia.org/wiki/%E9%87%8D%E6%94%BE%E6%94%BB%E5%87%BB
