---
title: "使用composer+k8s部署hyperledger fabric 1.1.0"
date: 2018-04-22 14:17:39
categories: blockchain
tags:
  - blockchain
  - hyperledger
  - composer
---

最近在[K8s 上升级了 fabric 网络][fabric_k8s], 然后想结合[Hyperledger Composer][composer]来部署一些例子。中间遇到了一些问题，走了一些弯路，在[这里记录][composer-work]下来。其实问题答案很简单，一些没注意的地方。

首先说一下背景，在这篇[部署 Fabric 多组织][deploy_multi_orgs]的教程中，最重要的是其中的连接描述文件，这个文件中描述了整个 Fabric 网络的结构，如有几个组织？每个组织下有几个 Peer？有多少个 Orderer？有哪些通道？又有哪些 Peer 加入了通道？连接时是否使用 TLS？等等。在依照教程走完后，又尝试在原先的 K8s 环境中也进行一次部署，这里就产生问题了。

1.  在 k8s 中无法正常导入 network card

    - 教程中使用 TLS 连接各 Peer 节点

      ```
      "peer1.org1.example.com": {
        "url": "grpcs://localhost:8051",
        "eventUrl": "grpcs://localhost:8053",
        "grpcOptions": {
          "ssl-target-name-override": "peer1.org1.example.com"
        },
        "tlsCACerts": {
          "pem": "-----BEGIN CERTIFICATE-----\nMIICSTCCAe+gAwIBAgIQUF0P/..."
        }
      },
      ```

    - K8S 中没有使用 TLS 方式**_(错误)_**

      ```
      "peer1.org1.example.com": {
        "url": "grpcs://localhost:8051",
        "eventUrl": "grpcs://localhost:8053"
       }
      ```

    - 正确的方式(**_注意区分 grpcs 和 grpc_**)

      ```
      "peer1.org1.example.com": {
        "url": "grpc://localhost:8051",
        "eventUrl": "grpc://localhost:8053"
       }
      ```

    - 怎么发现的

      在[这个项目][composer-work]中，运行脚本时有条命令执行失败

      `composer card import -f PeerAdmin@byfn-network-org1.card ... PEM encoded certificate is required.`

      这里使用了 Nodejs 的 Inspect 功能

            ```
            node --inspect-brk ~/.nvm/versions/node/v8.11.1/bin/composer card import -f PeerAdmin@byfn-network-org1.card
            ```

      在 Google 浏览器打开 chrome://inspect

      ![](http://ou36ephoe.bkt.clouddn.com/15244705012669.jpg)

      打开右上角的 pause on caught exception 开关

      ![](http://ou36ephoe.bkt.clouddn.com/15244705472639.jpg)

      在跳过前几次异常后，会跳到~/.nvm/versions/node/v8.11.1/lib/node_modules/composer-cli/node_modules/fabric-client/lib/Remote.js 这个文件中这一段代码

      ```
      if (protocol === 'grpc') {
        this.addr = purl.host;
        this.creds = grpc.credentials.createInsecure();
      } else if (protocol === 'grpcs') {
        if(!(typeof pem === 'string')) {
          throw new Error('PEM encoded certificate is required.');
        }
      ```

      当时看到这里时，顿时就是明白了，grpcs 代表 TLS 连接，而 grpc 则代表非加密连接，修改连接配置文件，代码成功执行。

2.  在 k8s，无法正常执行 composer network start,也即无法实例化 Chainacode

        这个问题折腾了一个下午，通过查看容器日志，发现在执行Chaincode实例化的化时候，需要生成类似如下镜像文件

        ```
        deploy@devops2:~$ docker images |ag dev

    dev-peer0.org2-trade-network-0.2.4-deploy.0-xxx
    `其中peer0.org2是节点名字，表示组织2上的peer0这个节点，trade-network是composer生成的Business Network名称，0.2.4-deploy.0是Business Network的版本号，这个镜像一开始是没有的，需要根据hyperledger/fabric-ccenv构建出来，在我们的这个K8S环境中，就是这个镜像构建失败，导致Chaincode无法实例化而出现问题。这也是1.1.0这个版本与前几个版本的一个区别，在构建镜像的过程中，需要通过NPM下载一些工具，却发现NPM下载失败。 一开始我还怀疑是不是翻墙的问题，结果发现是Docker容器启动后根本就无法连接网络，这就郁闷了。这里用到了一些测试工具。`
    docker run -it webwurst/curl-utils > ping www.baidu.com > curl -v www.baidu.com
    `问题的根本原因：错误的docker启动项`
    deploy@devops2:~\$ cat /etc/systemd/system/docker.service.d/docker-options.conf
    [Service]
    Environment="DOCKER_OPTS=--insecure-registry=10.233.0.0/18 --graph=/var/lib/docker --log-opt max-size=50m --log-opt max-file=5 --iptables=false"

    ```

     在 dockerd 的 man 手册页里面，--iptables 默认为 true，不知道为什么这里设置成了 false，删除掉这个选项后，系统工作正常了。
    ```

[deploy_multi_orgs]: https://hyperledger.github.io/composer/latest/tutorials/deploy-to-fabric-multi-org
[composer-work]: https://github.com/marryton007/composer-work
[fabric_k8s]: https://github.com/marryton007/hyperledger-fabric-k8s
[composer]: https://hyperledger.github.io/composer/latest/index.html
