---
layout: post
title: "基于hyperledger fabric网络的第一个应用"
date: 2017-08-23 14:17:39
categories: blockchain
tags:
  - blockchain
  - hyperledger
  - fabric
---

1. ### 摘要

   本文基本 hyperledger fabric 区块链网络，编写一个简单的应用，旨在讲述以下几点：

   - 如何启动一个 hyperledger fabric 区块链网络
   - 如何编写一个简单的智能合约
   - 如何将区块链应用与现有系统整合

2. ### 准备工作

   ```sh
   git clone https://github.com/hyperledger/fabric-samples.git
   cd fabric-samples/fabcar
   ```

3. ### 启动区块链网络

   ```sh
   ./startFabric.sh
   ```

   来看一下上面的 startFabric.sh 这个脚本

   ```sh
   #!/bin/bash
   #
   # Copyright IBM Corp All Rights Reserved
   #
   # SPDX-License-Identifier: Apache-2.0
   #
   # Exit on first error
   set -e

   # don't rewrite paths for Windows Git Bash users
   export MSYS_NO_PATHCONV=1

   starttime=$(date +%s)

   if [ ! -d ~/.hfc-key-store/ ]; then
   	mkdir ~/.hfc-key-store/
   fi
   cp $PWD/creds/* ~/.hfc-key-store/
   # launch network; create channel and join peer to channel
   # 进入同级目录basic-network
   cd ../basic-network
   # 运行start.sh脚本
   ./start.sh

   # Now launch the CLI container in order to install, instantiate chaincode
   # and prime the ledger with our 10 cars
   # 使用docker-compose 启动cli容器
   docker-compose -f ./docker-compose.yml up -d cli

   # 在cli容器中执行peer chaincode install来安装合约
   docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp" cli peer chaincode install -n fabcar -v 1.0 -p github.com/fabcar

   # 在cli容器中执行peer chaincode instantiate来初始化合约，这个操作将会调用合约里的Init函数
   # -C 指定通道名称
   # -c 指定初始化参数
   # -P 指定背书策略，这里只要是Org1MSP或Org2MSP任一成员即可
   docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp" cli peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n fabcar -v 1.0 -c '{"Args":[""]}' -P "OR ('Org1MSP.member','Org2MSP.member')"
   sleep 10

   # 在cli容器里执行peer chaincode invoke操作来调用合约中的initLedger函数
   docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp" cli peer chaincode invoke -o orderer.example.com:7050 -C mychannel -n fabcar -c '{"function":"initLedger","Args":[""]}'

   printf "\nTotal execution time : $(($(date +%s) - starttime)) secs ...\n\n"
   ```

   看一下*startFabric.sh*脚本中用到的*../basic-network/start.sh*脚本

   ```sh
   #!/bin/bash
   #
   # Copyright IBM Corp All Rights Reserved
   #
   # SPDX-License-Identifier: Apache-2.0
   #
   # Exit on first error, print all commands.
   set -ev

   # don't rewrite paths for Windows Git Bash users
   export MSYS_NO_PATHCONV=1

   # 根据docker-compose.yml文件停止相关的容器
   docker-compose -f docker-compose.yml down

   # 根据docker-compose.yml文件启动如下容器：
   # ca.example.com容器提供CA认证服务
   # orderer.example.com提供排序(共识)服务
   # peer0.org1.example.com提供背书和账本服务，
   # couchdb作为Key-Value服务来存储区块链状态(world state)
   docker-compose -f docker-compose.yml up -d ca.example.com orderer.example.com peer0.org1.example.com couchdb

   # wait for Hyperledger Fabric to start
   # incase of errors when running later commands, issue export FABRIC_START_TIMEOUT=<larger number>
   export FABRIC_START_TIMEOUT=10
   #echo ${FABRIC_START_TIMEOUT}
   sleep ${FABRIC_START_TIMEOUT}

   # Create the channel
   # 创建通道
   docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel create -o orderer.example.com:7050 -c mychannel -f /etc/hyperledger/configtx/channel.tx

   # Join peer0.org1.example.com to the channel.
   # 将peer1.org1.example.com加入通道
   docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel join -b mychannel.block
   ```

4) ### 智能合约关键内容解释

   合约代码在*../chaincode/fabcar/fabcar.go*文件中定义，

   ```go
   package main

   /* Imports
    * 4 utility libraries for formatting, handling bytes, reading and writing JSON, and string manipulation
    * 2 specific Hyperledger Fabric specific libraries for Smart Contracts
    */
   import (
   	"bytes"
   	"encoding/json"
   	"fmt"
   	"strconv"

   	"github.com/hyperledger/fabric/core/chaincode/shim"
   	sc "github.com/hyperledger/fabric/protos/peer"
   )
   // Define the Smart Contract structure
   type SmartContract struct {
   }

   // Define the car structure, with 4 properties.  Structure tags are used by encoding/json library
   type Car struct {
   	Make   string `json:"make"`
   	Model  string `json:"model"`
   	Colour string `json:"colour"`
   	Owner  string `json:"owner"`
   }

   /*
    * The Init method is called when the Smart Contract "fabcar" is instantiated by the blockchain network
    * Best practice is to have any Ledger initialization in separate function -- see initLedger()
    * 合约初始化函数
    */
   func (s *SmartContract) Init(APIstub shim.ChaincodeStubInterface) sc.Response {
   	return shim.Success(nil)
   }

   /*
    * The Invoke method is called as a result of an application request to run the Smart Contract "fabcar"
    * The calling application program has also specified the particular smart contract function to be called, with arguments
    * 查询和更新账本的入口
    */
   func (s *SmartContract) Invoke(APIstub shim.ChaincodeStubInterface) sc.Response {

   	// Retrieve the requested Smart Contract function and arguments
   	function, args := APIstub.GetFunctionAndParameters()
   	// Route to the appropriate handler function to interact with the ledger appropriately
   	// 根据传入的函数名称调用对应的代码
   	if function == "queryCar" {
   		return s.queryCar(APIstub, args)
   	} else if function == "initLedger" {
   		return s.initLedger(APIstub)
   	} else if function == "createCar" {
   		return s.createCar(APIstub, args)
   	} else if function == "queryAllCars" {
   		return s.queryAllCars(APIstub)
   	} else if function == "changeCarOwner" {
   		return s.changeCarOwner(APIstub, args)
   	}

   	return shim.Error("Invalid Smart Contract function name.")
   }

   /*
    * 根据轿车ID查询某一辆轿车的信息，轿车ID存放在传入args[0]中
    */
   func (s *SmartContract) queryCar(APIstub shim.ChaincodeStubInterface, args []string) sc.Response {

   	if len(args) != 1 {
   		return shim.Error("Incorrect number of arguments. Expecting 1")
   	}

   	carAsBytes, _ := APIstub.GetState(args[0])
   	return shim.Success(carAsBytes)
   }

   /*
    * 初始化账本，存入10台轿车数据
    */
   func (s *SmartContract) initLedger(APIstub shim.ChaincodeStubInterface) sc.Response {
   	cars := []Car{
   		Car{Make: "Toyota", Model: "Prius", Colour: "blue", Owner: "Tomoko"},
   		Car{Make: "Ford", Model: "Mustang", Colour: "red", Owner: "Brad"},
   		Car{Make: "Hyundai", Model: "Tucson", Colour: "green", Owner: "Jin Soo"},
   		Car{Make: "Volkswagen", Model: "Passat", Colour: "yellow", Owner: "Max"},
   		Car{Make: "Tesla", Model: "S", Colour: "black", Owner: "Adriana"},
   		Car{Make: "Peugeot", Model: "205", Colour: "purple", Owner: "Michel"},
   		Car{Make: "Chery", Model: "S22L", Colour: "white", Owner: "Aarav"},
   		Car{Make: "Fiat", Model: "Punto", Colour: "violet", Owner: "Pari"},
   		Car{Make: "Tata", Model: "Nano", Colour: "indigo", Owner: "Valeria"},
   		Car{Make: "Holden", Model: "Barina", Colour: "brown", Owner: "Shotaro"},
   	}

   	i := 0
   	for i < len(cars) {
   		fmt.Println("i is ", i)
   		carAsBytes, _ := json.Marshal(cars[i])
   		APIstub.PutState("CAR"+strconv.Itoa(i), carAsBytes)
   		fmt.Println("Added", cars[i])
   		i = i + 1
   	}

   	return shim.Success(nil)
   }

   /*
    * 在账本中添加一辆新轿车，轿车信息由args参数传入
    */
   func (s *SmartContract) createCar(APIstub shim.ChaincodeStubInterface, args []string) sc.Response {

   	if len(args) != 5 {
   		return shim.Error("Incorrect number of arguments. Expecting 5")
   	}

   	var car = Car{Make: args[1], Model: args[2], Colour: args[3], Owner: args[4]}

   	carAsBytes, _ := json.Marshal(car)
   	APIstub.PutState(args[0], carAsBytes)

   	return shim.Success(nil)
   }

   /*
    * 查询所有轿车信息
    */
   func (s *SmartContract) queryAllCars(APIstub shim.ChaincodeStubInterface) sc.Response {

   	startKey := "CAR0"
   	endKey := "CAR999"

   	resultsIterator, err := APIstub.GetStateByRange(startKey, endKey)
   	if err != nil {
   		return shim.Error(err.Error())
   	}
   	defer resultsIterator.Close()

   	// buffer is a JSON array containing QueryResults
   	var buffer bytes.Buffer
   	buffer.WriteString("[")

   	bArrayMemberAlreadyWritten := false
   	for resultsIterator.HasNext() {
   		queryResponse, err := resultsIterator.Next()
   		if err != nil {
   			return shim.Error(err.Error())
   		}
   		// Add a comma before array members, suppress it for the first array member
   		if bArrayMemberAlreadyWritten == true {
   			buffer.WriteString(",")
   		}
   		buffer.WriteString("{\"Key\":")
   		buffer.WriteString("\"")
   		buffer.WriteString(queryResponse.Key)
   		buffer.WriteString("\"")

   		buffer.WriteString(", \"Record\":")
   		// Record is a JSON object, so we write as-is
   		buffer.WriteString(string(queryResponse.Value))
   		buffer.WriteString("}")
   		bArrayMemberAlreadyWritten = true
   	}
   	buffer.WriteString("]")

   	fmt.Printf("- queryAllCars:\n%s\n", buffer.String())

   	return shim.Success(buffer.Bytes())
   }

   /*
    * 修改某一轿车的车主，轿车ID和车主信息args参数传入
    */
   func (s *SmartContract) changeCarOwner(APIstub shim.ChaincodeStubInterface, args []string) sc.Response {

   	if len(args) != 2 {
   		return shim.Error("Incorrect number of arguments. Expecting 2")
   	}

   	carAsBytes, _ := APIstub.GetState(args[0])
   	car := Car{}

   	json.Unmarshal(carAsBytes, &car)
   	car.Owner = args[1]

   	carAsBytes, _ = json.Marshal(car)
   	APIstub.PutState(args[0], carAsBytes)

   	return shim.Success(nil)
   }

   // The main function is only relevant in unit test mode. Only included here for completeness.
   func main() {

   	// Create a new Smart Contract
   	err := shim.Start(new(SmartContract))
   	if err != nil {
   		fmt.Printf("Error creating new Smart Contract: %s", err)
   	}
   }
   ```

   fabric 使用 go 语言来编写智能合约，使用 docker 容器来运行合约代码。每份合约只需要实现如下接口

   ```go
   type Chaincode interface {
   	// 合约初始时被调用，即在执行peer chaincode instantiate命令时调用且仅调用一次
   	Init(stub ChaincodeStubInterface) pb.Response

   	// Invoke 用来查询或更新账本状态
   	Invoke(stub ChaincodeStubInterface) pb.Response
   }
   ```

   可以将 fabric 的账本(world state)理解为一个 Key-Value 数据库或是 map(对象，字典等)，这也是为什么可以使用[couchdb][couchdb]的原因，合约的主要功能就是与这个数据库打交道，如：

   - APIstub.GetState(key) 读取 Key 对应的 Value
   - APIstub.PutState(key, value)，将 key-value 值对写入数据库中

5) ### 调用合约功能

   区块链网络已经启动，合约也已经部署好，现在来看看如何访问合约中的功能，先看看 query.js，其他的 js 文件如 invoke.js 程序结构相似。

   ```js
   "use strict";
   /*
    * Copyright IBM Corp All Rights Reserved
    *
    * SPDX-License-Identifier: Apache-2.0
    */
   /*
    * Hyperledger Fabric Sample Query Program
    */

   // nodejs的fabric客户端
   var hfc = require("fabric-client");
   var path = require("path");

   // 相关配置参数
   var options = {
     wallet_path: path.join(__dirname, "./creds"),
     user_id: "PeerAdmin",
     channel_id: "mychannel",
     chaincode_id: "fabcar",
     network_url: "grpc://localhost:7051",
   };

   var channel = {};
   var client = null;

   Promise.resolve()
     .then(() => {
       console.log("Create a client and set the wallet location");
       // 初始化新的fabric客户端
       client = new hfc();
       // 设置客户端证书缓存位置，连接时首先要经过CA认证，客户端会缓存用户的认证信息
       return hfc.newDefaultKeyValueStore({ path: options.wallet_path });
     })
     .then((wallet) => {
       console.log(
         "Set wallet path, and associate user ",
         options.user_id,
         " with application"
       );
       client.setStateStore(wallet);
       // 获取用户对应的信息
       return client.getUserContext(options.user_id, true);
     })
     .then((user) => {
       console.log(
         "Check user is enrolled, and set a query URL in the network"
       );
       if (user === undefined || user.isEnrolled() === false) {
         // 用户认证失败
         console.error("User not defined, or not enrolled - error");
       }
       // 新建通道
       channel = client.newChannel(options.channel_id);
       // 将节点加入通道中
       channel.addPeer(client.newPeer(options.network_url));
       return;
     })
     .then(() => {
       console.log("Make query");
       // 构造交易ID
       var transaction_id = client.newTransactionID();
       console.log(
         "Assigning transaction_id: ",
         transaction_id._transaction_id
       );

       // queryCar - requires 1 argument, ex: args: ['CAR4'],
       // queryAllCars - requires no arguments , ex: args: [''],
       // 构造区块链请求结构体
       const request = {
         // 合约ID
         chaincodeId: options.chaincode_id,
         // 交易ID
         txId: transaction_id,
         // 要调用的合约函数名
         fcn: "queryAllCars",
         // 传递给合约函数的参数
         args: [""],
       };
       // 发起合约调用
       return channel.queryByChaincode(request);
     })
     .then((query_responses) => {
       // 合约调用结果返回
       console.log("returned from query");
       if (!query_responses.length) {
         console.log("No payloads were returned from query");
       } else {
         console.log("Query result count = ", query_responses.length);
       }
       if (query_responses[0] instanceof Error) {
         console.error("error from query = ", query_responses[0]);
       }
       console.log("Response is ", query_responses[0].toString());
     })
     .catch((err) => {
       console.error("Caught Error", err);
     });
   ```

6) ### 总结

   看到这里，是不是已经懵了:(

   想用区块链写个合约是不是好麻烦啊，要用到 docker，还要会 go，再来点 Nodejs，好难啊。。。想必 hyperledger 社区也意识到了这个问题，所以他们推出了[Hyperledger Composer][composer]项目，不用专门学 go——其实学 go 也没坏处:)，composer 主页上介绍说可以将开发时间变成以周为单位，而不是月为单位。后续会推出[Hyperledger Composer][composer]相关文章，敬请关注。

7. ### 参考资料
   [hyperledger fabric write first application][reference]
   [fabric-client 代码库][client]
   [fabric-client-api 文档][api]

[reference]: http://hyperledger-fabric.readthedocs.io/en/latest/write_first_app.html
[client]: https://github.com/hyperledger/fabric-sdk-node
[api]: https://fabric-sdk-node.github.io/
[composer]: https://hyperledger.github.io/composer/
[couchdb]: http://couchdb.apache.org/
