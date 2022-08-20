---
layout: post
title: "使用hyperledger fabric构建你的第一个区块链网络"
date: 2017-08-22 14:17:39
tags:
  - blockchain
---

[官方文档参考][reference]

摘要：本文仅描述在单台 mac 机器上使用 docker 创建一个 fabric 区块链网络

1. #### 安装依赖工具

   - [cURL][curl]
   - [docker][docker] 17.03.1-ce 以上, 建议安装 docker for mac
   - docker-compose
   - [go][go] 1.7.x
   - [nodejs][nodejs] 6.9.x

2. #### 下载 fabric sample 项目

   ```sh
   mkdir  $HOME/work
   cd $HOME/work
   git clone https://github.com/hyperledger/fabric-samples.git
   cd fabric-samples
   git checkout -b release
   ```

   > 将源代码切换到 release 分支比较好，master 分支可能会有隐藏的 bug。

3. #### 下载平台相关的二进制文件和 docker 镜像文件

   ```sh
   cd $HOME/work
   curl -sSL https://goo.gl/eYdRbX -o fabric-samples.sh
   ```

   脚本内容如下：

   ```sh
   #!/bin/bash
   #
   # Copyright IBM Corp. All Rights Reserved.
   #
   # SPDX-License-Identifier: Apache-2.0
   #

   export VERSION=1.0.1
   export ARCH=$(echo "$(uname -s|tr '[:upper:]' '[:lower:]'|sed 's/mingw64_nt.*/windows/')-$(uname -m | sed 's/x86_64/amd64/g')" | awk '{print tolower($0)}')
   #Set MARCH variable i.e ppc64le,s390x,x86_64,i386
   MARCH=`uname -m`

   dockerFabricPull() {
     local FABRIC_TAG=$1
     for IMAGES in peer orderer couchdb ccenv javaenv kafka zookeeper tools; do
         echo "==> FABRIC IMAGE: $IMAGES"
         echo
         docker pull hyperledger/fabric-$IMAGES:$FABRIC_TAG
         docker tag hyperledger/fabric-$IMAGES:$FABRIC_TAG hyperledger/fabric-$IMAGES
     done
   }

   dockerCaPull() {
         local CA_TAG=$1
         echo "==> FABRIC CA IMAGE"
         echo
         docker pull hyperledger/fabric-ca:$CA_TAG
         docker tag hyperledger/fabric-ca:$CA_TAG hyperledger/fabric-ca
   }

   : ${CA_TAG:="$MARCH-$VERSION"}
   : ${FABRIC_TAG:="$MARCH-$VERSION"}

   echo "===> Downloading platform binaries"
   curl https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/${ARCH}-${VERSION}/hyperledger-fabric-${ARCH}-${VERSION}.tar.gz | tar xz

   echo "===> Pulling fabric Images"
   dockerFabricPull ${FABRIC_TAG}

   echo "===> Pulling fabric ca Image"
   dockerCaPull ${CA_TAG}
   echo
   echo "===> List out hyperledger docker images"
   docker images | grep hyperledger*
   ```

   该脚本会下载一些平台相关的二进制文件，如 cryptogen、configtxgen 等，下载的二进制文件会放在 bin 目录下，注意将该目录加入系统环境变量 PATH 中，如下设置：

   ```sh
   cd $HOME/work
   sh ./fabric-samples.sh
   echo "export PATH="$HOME/work/bin:$PATH"" >> $HOME/.profile
   ```

   同时，脚本还会下载相关的 docker 镜像，如 hyperledger/fabric-peer, hyperledger/fabric-orderer 等，下载镜像过程可能会比较慢，建议使用[daocloud][daocloud]或是阿里云加速

4. ### 获取区块链建链脚本帮助文档

   ```sh
   cd $HOME/work/fabric-samples/first-network
   ./byfn.sh -h
   Usage:
     byfn.sh -m up|down|restart|generate [-c <channel name>] [-t <timeout>]
     byfn.sh -h|--help (print this message)
       -m <mode> - one of 'up', 'down', 'restart' or 'generate'
         - 'up' - bring up the network with docker-compose up
         - 'down' - clear the network with docker-compose down
         - 'restart' - restart the network
         - 'generate' - generate required certificates and genesis block
       -c <channel name> - config name to use (defaults to "mychannel")
       -t <timeout> - CLI timeout duration in microseconds (defaults to 10000)

   Typically, one would first generate the required certificates and
   genesis block, then bring up the network. e.g.:

     byfn.sh -m generate -c <channelname>
     byfn.sh -m up -c <channelname>
   ```

5. ### 生成相关证书,启动和停止区块链网络

   ```sh
   ./byfn.sh -m generate
   ./byfn.sh -m up
   ./byfn.sh -m down
   ```

   - generate 的作用:
     - 调用 cryptogen 生成证书树，在区块链的运作过程中，每个节点都要使用证书对自己所生成的内容进行签名，其他节点都也需要证书对数据进行验证。
     - 调用 configtxgen 生成‘上帝区块’和‘通道初始信息’
   - up，使用 docker-compose 启动区块链网络
   - down,停止区块链网络

6) ### 相关名词解释

   - peer，节点，区块链组网基本单元，主要作用：
     - 负责对客户端发起的交易进行背书
     - 记账
   - anchor peer，锚(主)节点，用来跨组织进行通信的节点
   - orderer, 排序节点，通过对客户端发起的交易进行排序，其结果将发布到 peer 进行记账，orderer 节点是区块链共识算法的主要执行者，orderer 节点依据共识算法的不同，可以是 1 台机器，也可以是多台机器组成共识网络，fabric 支持集成 kafka、zookeeper 等
   - channel, 通道，更好的说法是‘子账本’，在一个区块链网络中，允许存在多个子账本，子账本之间互相隔离，一个通道里可以包含多个 peer 和 order。
   - ca，fabric 的认证授权管理，支持基于数据库和[LDAP][ldap]认证的方法
   - genesis block, 上帝区块，链中第 1 个区块
   - [msp][msp]， 成员服务提供商

7) ### crypto-config.yaml 文件说明

   ```yaml
   - Name: Orderer
     Domain: example.com
     Specs:
       - Hostname: orderer
   - Name: Org1
     Domain: org1.example.com
     Template:
       Count: 2
     Users:
       Count: 1
   - Name: Org2
     Domain: org2.example.com
     Template:
       Count: 2
     Users:
       Count: 1
   ```

   crypto-config.yaml 文件是 cryptogen 的配置文件，crypto-config.yaml 文件定义了整个 fabric 网络的组织结构。有如下关键因素。

   ```yaml
   * OrdererOrgs  定义orderer组织结构
       *  Name 组织名称
       *  Domain 组织域名
       *  Specs  显示定义组织
           *  Hostname 主机名称，与组织域名一起构成orderer全名
           *  CommonName 自定义主机名称，覆盖默认名称

   * PeerOrgs  定义peer组织结构
       *  Name 组织名称
       *  Domain 组织域名
       *  Specs  同OrdererOrgs
       *  Template  以模板形式定义组织
           *  Count 定义在该组织下peer节点数量，每个peer节点名称形如：peer[0-count-1].Domain
       * Users  定义用户
           *  Count 定义除了Admin之外用户数，每个用户名称形如User[1-count]@Domain
   ```

8) ### configtx.yaml

   ```yaml
   # Copyright IBM Corp. All Rights Reserved.
   #
   # SPDX-License-Identifier: Apache-2.0
   #

   ---
   ################################################################################
   #
   #   Profile
   #
   #   - Different configuration profiles may be encoded here to be specified
   #   as parameters to the configtxgen tool
   #
   ################################################################################
   Profiles:

       TwoOrgsOrdererGenesis:
           Orderer:
               <<: *OrdererDefaults
               Organizations:
                   - *OrdererOrg
           Consortiums:
               SampleConsortium:
                   Organizations:
                       - *Org1
                       - *Org2
       TwoOrgsChannel:
           Consortium: SampleConsortium
           Application:
               <<: *ApplicationDefaults
               Organizations:
                   - *Org1
                   - *Org2

   ################################################################################
   #
   #   Section: Organizations
   #
   #   - This section defines the different organizational identities which will
   #   be referenced later in the configuration.
   #
   ################################################################################
   Organizations:

       # SampleOrg defines an MSP using the sampleconfig.  It should never be used
       # in production but may be used as a template for other definitions
       - &OrdererOrg
           # DefaultOrg defines the organization which is used in the sampleconfig
           # of the fabric.git development environment
           Name: OrdererOrg

           # ID to load the MSP definition as
           ID: OrdererMSP

           # MSPDir is the filesystem path which contains the MSP configuration
           MSPDir: crypto-config/ordererOrganizations/example.com/msp

       - &Org1
           # DefaultOrg defines the organization which is used in the sampleconfig
           # of the fabric.git development environment
           Name: Org1MSP

           # ID to load the MSP definition as
           ID: Org1MSP

           MSPDir: crypto-config/peerOrganizations/org1.example.com/msp

           AnchorPeers:
               # AnchorPeers defines the location of peers which can be used
               # for cross org gossip communication.  Note, this value is only
               # encoded in the genesis block in the Application section context
               - Host: peer0.org1.example.com
                 Port: 7051

       - &Org2
           # DefaultOrg defines the organization which is used in the sampleconfig
           # of the fabric.git development environment
           Name: Org2MSP

           # ID to load the MSP definition as
           ID: Org2MSP

           MSPDir: crypto-config/peerOrganizations/org2.example.com/msp

           AnchorPeers:
               # AnchorPeers defines the location of peers which can be used
               # for cross org gossip communication.  Note, this value is only
               # encoded in the genesis block in the Application section context
               - Host: peer0.org2.example.com
                 Port: 7051

   ################################################################################
   #
   #   SECTION: Orderer
   #
   #   - This section defines the values to encode into a config transaction or
   #   genesis block for orderer related parameters
   #
   ################################################################################
   Orderer: &OrdererDefaults

       # Orderer Type: The orderer implementation to start
       # Available types are "solo" and "kafka"
       OrdererType: solo

       Addresses:
           - orderer.example.com:7050

       # Batch Timeout: The amount of time to wait before creating a batch
       BatchTimeout: 2s

       # Batch Size: Controls the number of messages batched into a block
       BatchSize:

           # Max Message Count: The maximum number of messages to permit in a batch
           MaxMessageCount: 10

           # Absolute Max Bytes: The absolute maximum number of bytes allowed for
           # the serialized messages in a batch.
           AbsoluteMaxBytes: 99 MB

           # Preferred Max Bytes: The preferred maximum number of bytes allowed for
           # the serialized messages in a batch. A message larger than the preferred
           # max bytes will result in a batch larger than preferred max bytes.
           PreferredMaxBytes: 512 KB

       Kafka:
           # Brokers: A list of Kafka brokers to which the orderer connects
           # NOTE: Use IP:port notation
           Brokers:
               - 127.0.0.1:9092

       # Organizations is the list of orgs which are defined as participants on
       # the orderer side of the network
       Organizations:

   ################################################################################
   #
   #   SECTION: Application
   #
   #   - This section defines the values to encode into a config transaction or
   #   genesis block for application related parameters
   #
   ################################################################################
   Application: &ApplicationDefaults

       # Organizations is the list of orgs which are defined as participants on
       # the application side of the network
       Organizations:
   ```

   configtxgen 根据 configtx.yaml 的配置来生成‘上帝区块’和‘channel’信息，上例中定义了 2 个配置。

   ```yaml
   * TwoOrgsOrdererGenesis 第1个配置名称，一般取比较有意义的名字，这里意思是包含2个组织的上帝区块配置说明
       * Orderer 排序(共识)选项
           * <<: *OrdererDefaults <<是yaml语法，表示引入OrdererDefaults的定义
           * Organizations 组织选项
               * OrdererOrg  引入OrdererOrg定义
       * Consortiums 多组织联合信息
           * SampleConsortium 联合名称
           * Organizations 联合中的组织列表

   * TwoOrgsChannel 第2个配置名称，包含2个组织的通道说明
       * Consortium 组织联合信息
       * Application
           * <<: *ApplicationDefaults  引入应用的定义
           * Organizations 应用的组织列表

   ```

9) ### 总结

   hyperledger fabric 的工具比较多，概念也不少，理解起来有一定的门槛。个人认为先要理解一些基本概念，再参考[官方文档][doc]尝试搭建区块链网络，尝试写 1 个合约，尝试与现有系统集成。再深入理解 crypto-config.yaml 和 configtx.yaml 文件，当然，这需要一定的时间和坚持。

10) ### 免责声明
    hyperledger fabric 的配置信息相对还是比较复杂的，且中文文档较少，以上内容只是个人理解，如有错误之处，万望指正。

[reference]: http://hyperledger-fabric.readthedocs.io/en/latest/build_network.html
[doc]: http://hyperledger-fabric.readthedocs.io/en/latest
[curl]: https://curl.haxx.se/download.html
[docker]: https://www.docker.com/products/overview
[go]: https://golang.org
[nodejs]: https://nodejs.org/en/
[daocloud]: https://www.daocloud.io/mirror#accelerator-doc
[ldap]: https://segmentfault.com/a/1190000002607140
[msp]: http://hyperledger-fabric.readthedocs.io/en/latest/msp.html
