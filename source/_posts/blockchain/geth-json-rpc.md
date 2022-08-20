---
layout: post
title: "以太坊节点geth json rpc使用指南"
date: 2017-16-13 14:17:39
categories: blockchain
tags:
  - blockchain
  - hyperledger
  - geth
---

1.  #### 参考

    - https://github.com/ethereum/go-ethereum/wiki/Management-APIs
    - https://github.com/ethereum/wiki/wiki/JSON-RPC
    - https://solidity.readthedocs.io/en/develop/abi-spec.html

2.  #### 新建账号

        ```
        curl -X POST --data '{
          "jsonrpc": "2.0",
          "method": "personal_newAccount",
          "params": ["hello"],     // 新帐户密码
          "id": 3

    }' localhost:8545
    `返回结果`
    {
    "id": 4,
    "jsonrpc": "2.0",
    "result": "0xb14dc5a4bba7fc2796b41543c76aa7da732a7271" // 新帐户地址
    }

    ```

    ```

3.  #### 查询余额

    ```
    {
      "jsonrpc": "2.0",
      "method": "eth_getBalance",
      "params": ["0x627306090abaB3A6e1400e9345bC60c78a8BEf57"],  //帐户地址
      "id": 3
    }
    ```

    返回结果

    ```
    {
      "id": 3,
      "jsonrpc": "2.0",
      "result": "0x00000000000000056bc75e2d63100000"  //以wei为单位，十六进制形式，转换成十进制为 100000000000000000000，即100ETH
    }
    ```

4.  #### 以太币转账

    ```
    curl -X POST --data '{
      "jsonrpc": "2.0",
      "method": "eth_sendTransaction",
      "params": [{
        "from": "0x627306090abaB3A6e1400e9345bC60c78a8BEf57",   // 转出帐户
        "to": "0xf17f52151EbEF6C7334FAD080c5704D77216b732",     // 转入帐户
        "gas": "0x76c0",                                        // gas
        "gasPrice": "0x9184e72a000",                            // gas单价
        "value": "0x8AC7230489E80000",                          // 转账金额
        "data": "0x0"
      }],
      "id": 3}' localhost:8545
    ```

    返回结果

    ```
      {
      "id": 3,
      "jsonrpc": "2.0",
      "result": "0x126bd21f39f23b7c6df8fe3e205fd592fe9dcb47abb3e356b96bff98ea58e137"   // 交易Hash
    }
    ```

5.  #### Token 转账

        * Token信息

        ```
        Token contract address:  0x8cdaf0cd259887258bc13a92c0a6da92698644c0
        Token 代号: JXC
        Token decimals(小数点位数): 2
        ```

        * 信息结构

        ```
        接收地址：　0xf17f52151EbEF6C7334FAD080c5704D77216b732    (40字母长度)
        value:  128
        data内容拼接：
        0xa9059cbb +  左补位token接收地址 + 左补位value
        ```

        * 信息具体构造方法

        ```
        其中token接收地址和value要在前面补0，直到长度满足64个字母，如下：

        Token地址前要加24个0，

        value作如下转换： 128*10^合约的小数位，这里是2，转成16进制，再在前面补0，长度满足64位字母
        128*100 = 12800
        hex(12800) = 3200

        data内容最终为：

    0xa9059cbb000000000000000000000000f17f52151EbEF6C7334FAD080c5704D77216b7320000000000000000000000000000000000000000000000000000000000003200

    ````

        * curl请求示例

        ```
        curl -X POST --data '{
          "jsonrpc": "2.0",
          "method": "eth_sendTransaction",
          "params": [{
            "from": "0x627306090abaB3A6e1400e9345bC60c78a8BEf57",   // 转出帐户
            "to": "0x8cdaf0cd259887258bc13a92c0a6da92698644c0",     // Token合约地址
            "gas": "0x186A0",                                       // gas
            "gasPrice": "0x9184e72a000",                            // gas单价
            "value": "0x0",                          //
            "data": "0xa9059cbb000000000000000000000000f17f52151EbEF6C7334FAD080c5704D77216b7320000000000000000000000000000000000000000000000000000000000003200"
          }],
          "id": 3}' localhost:8545
        ```

        返回结果

        ```
        {
          "id": 5,
          "jsonrpc": "2.0",
          "result": "0x7f68504a79cc20aff446e8a8799845f8c1a9433c34644dcf5b7876fd1bcba30a"   // 交易号
        }
        ```
    ````

6) #### 获取 Token 余额

   - Token 信息

   ```
   Token contract address:  0x8cdaf0cd259887258bc13a92c0a6da92698644c0
   Token 代号: JXC
   Token decimals(小数点位数): 2
   ```

   - 信息结构

   ```
   data 数据拼接：
   0x70a08231 + 左补位的地址
   ```

   - curl 示例

   ```
   curl -X POST --data '{
     "jsonrpc": "2.0",
     "method": "eth_call",
     "params": [{
       "from": "0x627306090abaB3A6e1400e9345bC60c78a8BEf57",   // 帐户
       "to": "0x8cdaf0cd259887258bc13a92c0a6da92698644c0",     // Token合约地址
       "value": "0x0",                          //
       "data": "0x70a08231000000000000000000000000f17f52151EbEF6C7334FAD080c5704D77216b732"
     }],
     "id": 3}' localhost:8545
   ```

   - 返回结果

   ```
   {
     "id": 3,
     "jsonrpc": "2.0",
     "result": "0x0000000000000000000000000000000000000000000000000000000000003200"
   }
   ```

   - 提取结果

   ```
   3200转十进制/10^2(Token小数点位数)

   0x3200 = 12800
   12800 / 100 = 128 个JXC
   ```
