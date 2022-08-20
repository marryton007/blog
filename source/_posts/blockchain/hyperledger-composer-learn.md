---
layout: post
title: "hyperledger composer 实战"
date: 2017-09-04 14:17:39
categories: blockchain
tags:
  - blockchain
  - hyperledger
  - composer
---

1. ### 背景介绍

   [hyperledger composer][composer]是 hyperledger 组织下的新星项目，封装了[Node SDK][sdk]与区块链交互的复杂细节，对使用者提供了一套简洁直观的语法来定义业务模型，可以让区块链开发人员快速上手，缩短开发时间，是一套非常好的工具。

2. ### 业务模型概念讲解  

   ![Business Networking](https://hyperledger.github.io/composer/assets/img/BusinessNetworkFiles.svg)    定义业务模型的过程在[composer][composer]中称之为定义'Business Network', 业务模型有以下几个关键要素：

   - Asset, 资产，可用于交易的价值物，比如房产买卖中的房子
   - Participant, 参与者，参与交易的人，比如房产买卖中房主、购房者、中介
   - Transaction, 交易，由某一参与者发起，对资产状态带来改变的操作，如房主挂牌销售、购房者出价，成交等
   - ACL, 权限控制, 限制参与者能够发起的交易(操作)

   同样，在上图可以看到，在[composer][composer]定义一个业务模型需要下面 3 类文件：

   - Model files， 模型文件，其中包含资产、参与者的定义，以及交易需要的参数
   - Javascript Files， 定义交易的具体实现逻辑
   - Access Control File， 定义 ACL 规则

   最终，将业务模型的所有文件打包成以.bna 结尾'Business Network'文件

3. ### 安装  

   ```sh
   npm install -g composer-cli generator-hyperledger-composer composer-rest-server yo
   ```

   - composer-cli 包含了开发'Business Network'的命令行工具，如打包，部署等
   - generator-hyperledger-composer Yo 脚手架插件，专用于快速生成'Business Newtork'
   - yo 脚手架开发框架
   - composer-rest-server 基于[loopback][loopback]的 API 生成工具，可以快速将'Business Network'转换成可以访问的 API，大大减少了开发时间

4. ### Composer 实战

   - 清理 Docker 容器，获得干净的测试环境

     ```sh
     docker kill $(docker ps -q)
     docker rm $(docker ps -aq)
     docker rmi $(docker images dev-* -q)
     ```

   - 启动 Fabric 网络

     ```sh
     mkdir ~/work/fabric-tools && cd ~/work/fabric-tools
     curl -O https://raw.githubusercontent.com/hyperledger/composer-tools/master/packages/fabric-dev-servers/fabric-dev-servers.tar.gz
     tar xvzf fabric-dev-servers.tar.gz

     ./downloadFabric.sh        //下载fabric相关docker镜像
     ./startFabric.sh           //使用Docker-compose启动Fabric网络
     ./createComposerProfile.sh //创建连接文件
     ```

     在 createComposerProfile.sh 脚本中， 将会生成~/.composer-connection-profiles/hlfv1/connection.json 连接文件，这个文件明确了 composer 如何连接 Fabric 区块链网络，如果你对 Fabric 网络比较熟悉的话，也可以不用执行前 2 个脚本，直接生成连接文件，然后按实际情况修正即可。注意：composer 推荐使用 Fabric1.0 网络。

   - 停止 Fabric 网络

     ```sh
     cd ~/work/fabric-tools
     ./stopFabric.sh
     ./teardownFabric.sh
     ```

   - 克隆 composer 例子代码

     ```sh
     cd ~/work
     git clone https://github.com/hyperledger/composer-sample-networks.git
     cp -r ./composer-sample-networks/packages/basic-sample-network/  ./my-network
     ```

     使用编辑器(官方推荐[VSCode][vscode])打开 my-network 目录, 从目录结构中可以看到几个重要的目录和文件

     - permissions.acl ACL 定义文件
     - models 目录，用于存储 Model Files(.cto)，每个.cto 使用独立的命名空间
     - lib 目录，用于存储 Javascript Files

   - 修改 my-network 项目

     - 修改 package.json 文件
       - 修改 name 为'my-network'
       - 修改 description 为'My Commodity Trading network'
       - 修改 prepublish 脚本中修改生成的 Business network 打包名为'my-network.bna'
     - 使用如下内容替换 models/sample.cto

       ```js
       /**
        * My commodity trading network
        */
       namespace org.acme.mynetwork
       asset Commodity identified by tradingSymbol {
           o String tradingSymbol
           o String description
           o String mainExchange
           o Double quantity
           --> Trader owner
       }
       participant Trader identified by tradeId {
           o String tradeId
           o String firstName
           o String lastName
       }
       transaction Trade {
           --> Commodity commodity
           --> Trader newOwner
       }
       ```

       在 sample.cto 文件中，定义了 Asset(货物)、Participant(商人)和 Transaction(贸易)，在贸易当中，货物会被易主

     - 使用如下内容替换 lib/sample.js

       ```js
       /*
        * Licensed under the Apache License, Version 2.0 (the "License");
        * you may not use this file except in compliance with the License.
        * You may obtain a copy of the License at
        *
        * http://www.apache.org/licenses/LICENSE-2.0
        *
        * Unless required by applicable law or agreed to in writing, software
        * distributed under the License is distributed on an "AS IS" BASIS,
        * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
        * See the License for the specific language governing permissions and
        * limitations under the License.
        */

       /**
        * Track the trade of a commodity from one trader to another
        * @param {org.acme.mynetwork.Trade} trade - the trade to be processed
        * @transaction
        */
       function tradeCommodity(trade) {
         trade.commodity.owner = trade.newOwner;
         return getAssetRegistry("org.acme.mynetwork.Commodity").then(function (
           assetRegistry
         ) {
           return assetRegistry.update(trade.commodity);
         });
       }
       ```

       如果一个 js 文件中的一个函数被@trasnaction 注释，则注释中@param 指定的交易类型(本例中为**'org.acme.mynetwork.Trade'**)被提交时，该函数会被自动执行。sample.js 文件完成了'货物易主'的逻辑:在一个 Trade(贸易)操作中，货物的属主被更新为 trade.newOwner，且被永久地更新到区块链上。

     - 修改权限控制文件 permissions.acl 为如下内容

       ```js
       /**
        * Access control rules for mynetwork
        */
       rule Default {
           description: "Allow all participants access to all resources"
           participant: "ANY"
           operation: ALL
           resource: "org.acme.mynetwork.*"
           action: ALLOW
       }

       rule SystemACL {
         description:  "System ACL to permit all access"
         participant: "org.hyperledger.composer.system.Participant"
         operation: ALL
         resource: "org.hyperledger.composer.system.**"
         action: ALLOW
       }
       ```

- 生成 Business Network Archive

  ```sh
  cd ~/work/my-network
  npm install
  ```

  你将看到如下输出：

  ```sh
  $ mkdirp ./dist && composer archive create --sourceType dir --sourceName . -a ./dist/my-network.bna

    Creating Business Network Archive
  ```


    Looking for package.json of Business Network Definition
    	Input directory: /Users/jiaxi/work/fabric-composer/my-network

    Found:
    	Description: My Commodity Trading network
    	Name: my-network
    	Identifier: my-network@0.1.8

    Written Business Network Definition Archive file to
    	Output file: ./dist/my-network.bna

    Command succeeded

    Done in 158.57s.

````

在./dist/目录下会生成一个.bna 文件，这个文件就 Business Network Archive 文件，这个文件将会被部署到 fabric 区块链上。

- 部署 Business Network Archive 文件

  ```sh
  cd dist
  composer network deploy -a my-network.bna -p hlfv1 -i PeerAdmin -s randomString
  ```

  部署过程差不多会花费 30 秒左右，如果一切正常，你将会看到如下结果：

  ```sh
   Deploying business network from archive: my-network.bna
   Business network definition:
   	Identifier: my-network@0.1.8
   	Description: My Commodity Trading network

   ✔ Deploying business network definition. This may take a minute...

   Command succeeded
  ```

  你可以通过以下指令到验证部署是否成功

  ```sh
  composer network ping -n my-network -p hlfv1 -i PeerAdmin -s adminpw
  ```

  如果结果如下，表明部署成功

  ```sh
  The connection to the network was successfully tested: my-network
       version: 0.12.0
       participant: <no participant found>

   Command succeeded
  ```

- 生成 REST API
到现在为止，你已经完成了 Business Network 的开发，你已经编写了业务模型文件.cto，还完成了业务逻辑的.js 的工作，权限控制文件.acl，并打包成了.bna 文件，部署到了 Fabric 区块链网络中。那么，接下来做什么？是时候将你的 Business Network 与其他系统集成在一起了。幸好，Hyperledger composer 提供了 composer-rest-server 组件，该组件将根据你定义的 Business Network，自动生成一系列的 REST API，以供外部系统访问。具体操作如下：

    ```sh
    cd ..
    composer-rest-server
    ```

    你需要回答一系列问题，composer-rest-server会自动生成一系统的REST API，下面的内容中，以*？* 开头的行是问题。

    ```sh
    ? Enter your Fabric Connection Profile Name: hlfv1
    ? Enter your Business Network Identifier : my-network
    ? Enter your Fabric username : PeerAdmin
    ? Enter your secret: adminpw
    ? Specify if you want namespaces in the generated REST API: never use namespaces
    ? Specify if you want to enable authentication for the REST API using Passport: No
    ? Specify if you want to enable event publication over WebSockets: Yes
    ? Specify if you want to enable TLS security for the REST API: No

    To restart the REST server using the same options, issue the following command:
       composer-rest-server -p hlfv1 -n my-network -i PeerAdmin -s adminpw -N never -w true

    Discovering types from business network definition ...
    Discovered types from business network definition
    Generating schemas for all types in business network definition ...
    Generated schemas for all types in business network definition
    Adding schemas for all types to Loopback ...
    Added schemas for all types to Loopback
    Web server listening at: http://localhost:3000
    Browse your REST API at http://localhost:3000/explorer
    ```

    请特别注意上面内容中的最后2行。

    ```sh
    // REST API 服务网址
    http://localhost:3000

    // REST API 服务列表，详述了每个API的功能，参数列表等，并且可以在线测试，可做API文档使用
    http://localhost:3000/explorer
    ```

    生成的REST API中，你可以创建、获取、更新、删除Asset和Participant，还可以发起一笔Transaction，还有一些查询系统所有交易，或是根据ID查询交易之类的系统函数。

* 生成 WEB 应用程序骨架
除了可用 composer-rest-server 来对 Business Network 来进行访问以外，还可以使用 generator-hyperledger-composer 来生成 WEB 应用程序骨架，骨架程序之中，已经包含对 Business Network 的访问代码，可以让 WEB 开发人员不必关心 Business Network 的实现细节，只要调用暴露出来的 REST API 即可。使用如下命令来生成骨架
```sh
yo hyperledger-composer
````

同样要回答一系列问题

```sh
Welcome to the Hyperledger Composer project generator
? Please select the type of project: Angular
You can run this generator using: 'yo hyperledger-composer:angular'
Welcome to the Hyperledger Composer Angular project generator
? Do you want to connect to a running Business Network? Yes
? Project name: my-app
? Description: Commodities Trading App
? Author name: liujiaxi
? Author email: liujiaxi@mana.com
? License: Apache-2.0
? Business network identifier: my-network
? Connection profile: hlfv1
? Enrollment ID: PeerAdmin
? Enrollment secret: adminpw
? Do you want to generate a new REST API or connect to an existing REST API?  Generate a new REST API
? REST server port: 3000
? Should namespaces be used in the generated REST API? Never use namespaces
Created application!
Completed generation process
   create e2e/app.e2e-spec.ts
   create e2e/app.po.ts
   create e2e/tsconfig.e2e.json
   create e2e/tsconfig.json
   create karma.conf.js
   create package.json
   create protractor.conf.js
   create README.md
   create src/app/app-routing.module.ts
   create src/app/app.component.css
   create src/app/app.component.html
   create src/app/app.component.spec.ts
   create src/app/app.component.ts
   create src/app/app.module.ts
   create src/app/configuration.ts
   create src/app/data.service.ts
   create src/app/home/home.component.html
   create src/app/home/home.component.ts
   create src/environments/environment.prod.ts
   create src/environments/environment.ts
   create src/favicon.ico
   create src/index.html
   create src/main.ts
   create src/polyfills.ts
   create src/styles.css
   create src/test.ts
   create src/tsconfig.app.json
   create src/tsconfig.json
   create src/tsconfig.spec.json
   create tsconfig.json
   create tslint.json
   create .angular-cli.json
   create .editorconfig
   create .gitignore
   create src/app/Commodity/Commodity.component.ts
   create src/app/Commodity/Commodity.service.ts
   create src/app/Commodity/Commodity.component.spec.ts
   create src/app/Commodity/Commodity.component.html
   create src/app/Commodity/Commodity.component.css
```

进入项目目录，运行应用程序

```sh
cd my-app  //根据骨架自动生成
npm start  //启动应用程序
```

根据 README.md 和 pacakge.json 文件可以看出，该项目使用[angular-cli][cli]和 composer-rest-server，其中 composer-rest-server 提供访问 Business Network 的 API，angular-cli 提供 WEB 访问界面。以下内容出自 package.json 文件

```json
  "name": "my-app",
  "description": "Commodities Trading App",
  "version": "0.0.1",
  "author": "liujiaxi",
  "email": "liujiaxi@mana.com",
  "license": "Apache-2.0",
  "scripts": {
    "app": "composer-rest-server -n my-network -p hlfv1 -i PeerAdmin -s adminpw -N never -P 3000",
    "start": "concurrently \"ng serve --host 0.0.0.0\" \"npm run app\""
  }
```

如果启动过程中没有出现错误信息，则访问http://0.0.0.0:4200/即可进入WEB应用界面。

4. ### FAQ

   1. 建议使用 npm 来安装相关的模块。笔者一开始是使用 yarn 来全局安装模块，结果后面某些步骤就无法进行下去了，这里还没有弄明白原因，但考虑为了节约读者的时间，还是先用 npm 吧。
   2. docker 镜像最好使用 1.0.0 版。笔者在尝试部署 Business Network 的时候总是失败，最后试下来发现用的 docker 镜像是 1.0.1 版(最新版)，无奈之下，只好使用[fabric-samples][samples]中的 fabcar 来启动 fabric 区块链网络。
   3. 读者已经发现笔者总是使用 PeerAdmin 这个用户名来访问区块链，这是怎么做到的呢，请读者参考[Composer Identity Import][import]

5. ### 总结

   本是只是笔者个人对于 hyperledger composer 工具集一些个人理解，并整合官方的[hyperledger composer 开发者指南][guide],这里笔者仅描述了使用 hyperledger composer 的开发基于 Fabric 区块链应用的一个大概流程，还有很多具体的细节没有涉及，希望读者能通过阅读官方文档的方式来进一步学习。

6. ### 参考资料
   [hyperledger composer][composer]
   [Node 模块包依赖管理工具——Yarn][yarn]
   [loopback: rest API server][loopback]
   [yeoman][yo]
   [angular-cli][cli]
   [hyperledger composer 开发者指南][guide]

[composer]: https://hyperledger.github.io/composer/index.html
[sdk]: https://github.com/hyperledger/fabric-sdk-node
[yarn]: https://yarnpkg.com/zh-Hans/
[yo]: http://yeoman.io/
[loopback]: https://loopback.io/
[vscode]: https://code.visualstudio.com/
[cli]: https://github.com/angular/angular-cli
[samples]: https://github.com/hyperledger/fabric-samples
[import]: https://hyperledger.github.io/composer/reference/composer.identity.import.html
[guide]: https://hyperledger.github.io/composer/tutorials/developer-guide.html
