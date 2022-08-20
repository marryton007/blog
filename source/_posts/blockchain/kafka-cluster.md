---
layout: post
title: "搭建Kafka集群"
date: 2017-09-28 14:17:39
categories: blockchain
tags:
  - blockchain
  - hyperledger
  - composer
---

1. #### 起因

   因为[Hyperledger Fabric][fabric]使用[Apache Kafka][kafka]来作为期 Orderer Service 的实现，所以在研究[Fabric][fabric]的时候，也来尝试搭建一个[kafka][kafka]集群。

2. #### 简介

   [Kafka][kafka]是由 LinkedIn 开发的一个分布式的消息系统，使用 Scala 编写，它以可水平扩展和高吞吐率而被广泛使用，其依赖于[ZooKeeper][zookeeper]。  
   [ZooKeeper][zookeeper]是一个分布式的，开放源码的分布式应用程序协调服务，它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。  
   [Ansible][ansible]是一款简单的自动化 IT 工具。这个工具的目标有这么几项：自动化部署 APP；自动化管理配置项；自动化的持续交互；自动化的（AWS）云服务管理。即可以批量的在远程服务器上通过 SSH 执行命令，且无须额外在服务器上安装控制工具。

3. #### 硬件环境

   - 3 台 Ubuntu 16.04 机器
   - 1 台 Mac，作为控制机

4. #### 初始配置

   - 3 台 Ubuntu 机器配置 ssh 服务，创建 deploy 用户(自定)，允许 deploy 用户无密码 sudo
   - Mac 上安装[ansible][ansible]，设置从 Mac 用户无密码登录 Ubuntu 机器

5. #### 关键步骤

   - 在 Mac 上配置[Ansible][ansible]主机列表

     - 查看~/.ansible.cfg

       ```
       [defaults]
       hostfile=~/.hosts    //host文件位置
       hash_behaviour = merge
       ```

     - 修改~/.hosts, 添加如下内容

       ```
       [kafka]
       devops0
       devops2
       devops1
       ```

     - 修改/etc/hosts 文件，添加如下内容

       ```
       192.168.1.208   devops0
       192.168.1.207   devops1
       192.168.1.209   devops2
       ```

   - 在 Ubuntu 机器上安装 JAVA
     - 编写 ansible playbook 文件 install-jdk.yml
       ```
       ---
       - hosts: kafka
         user: deploy
         become: yes
         become_user: root
         tasks:
           - name: install some packages
             apt:
               name: "{{item}}"
               state: present
             with_items:
               - openjdk-8-jdk
       ```
     - 执行
       ```
       ansible-playbook install-jdk.yml
       ```
   - 在 Ubuntu 机器上配置[Zookeeper][zookeeper]集群

     - 安装 zookeeper role
       ```
       ansible-galaxy install AnsibleShipyard.ansible-zookeeper
       ```
     - 编写 playbook 文件 zookeeper-cluster.yml

       {%  raw  %}

       ```
       ---
       - name: Installing ZooKeeper
         hosts: kafka
         user: deploy
         become: yes
         become_user: root
         roles:
           - role: AnsibleShipyard.ansible-zookeeper
             zookeeper_hosts: "
               {%- set ips = [] %}
               {%- for host in groups['kafka'] %}
               {{- ips.append(dict(id=loop.index, host=host, ip=hostvars[host]['ansible_default_ipv4'].address)) }}
               {%- endfor %}
               {{- ips -}}"
       ```

       {% endraw %}


     - 执行

       ```
       ansible-playbook zookeeper-cluster.yml
       ```

- 在 ubuntu 机器上配置[Kafka][kafka]集群

  - 安装 kafka role
    ```
    ansible-galaxy install  idealista.kafka-role
    ```
  - 编写 playbook 文件， kafaka.yml

    ```
    ---
    - hosts: kafka
      user: deploy
      become: yes
      roles:
        - role: idealista.kafka-role
          kafka_hosts:
            - host: devops0
              id: 1
            - host: devops1
              id: 2
            - host: devops2
              id: 3
          kafka_zookeeper_hosts:
            - 192.168.1.207:2181
            - 192.168.1.208:2181
            - 192.168.1.209:2181

      environment:
        http_proxy: http://192.168.2.59:1087
        https_proxy: http://192.168.2.59:1087
    ```

  - 执行

    ```
    ansible-playbook kafka.yml
    ```

6. #### 测试

   - zookeeper 测试

     ```
     // 查看节点状态
     deploy@devops2:/opt/zookeeper-3.4.9/bin$ ./zkServer.sh status
     ZooKeeper JMX enabled by default
     Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
     Mode: leader  //节点角色

     // 发送stat指令
     deploy@devops2:/opt/zookeeper-3.4.9/bin$ echo stat |nc localhost 2181
     Zookeeper version: 3.4.9-1757313, built on 08/23/2016 06:50 GMT
     Clients:
      /0:0:0:0:0:0:0:1:54100[0](queued=0,recved=1,sent=0)

     Latency min/avg/max: 0/0/33
     Received: 6353
     Sent: 6369
     Connections: 1
     Outstanding: 0
     Zxid: 0x1000001b2
     Mode: leader
     Node count: 32

     // 使用./zkCli.sh 连接到zookeeper
     deploy@devops2:/opt/zookeeper-3.4.9/bin$ ./zkCli.sh -server localhost
     Connecting to localhost
     2017-09-28 10:02:24,947 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.9-1757313, built on 08/23/2016 06:50 GMT
     2017-09-28 10:02:24,949 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=devops2
     2017-09-28 10:02:24,949 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.8.0_131
     2017-09-28 10:02:24,950 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
     2017-09-28 10:02:24,950 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/usr/local/jre1.8.0_131
     2017-09-28 10:02:24,950 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/opt/zookeeper-3.4.9/bin/../build/classes:/opt/zookeeper-3.4.9/bin/../build/lib/*.jar:/opt/zookeeper-3.4.9/bin/../lib/slf4j-log4j12-1.6.1.jar:/opt/zookeeper-3.4.9/bin/../lib/slf4j-api-1.6.1.jar:/opt/zookeeper-3.4.9/bin/../lib/netty-3.10.5.Final.jar:/opt/zookeeper-3.4.9/bin/../lib/log4j-1.2.16.jar:/opt/zookeeper-3.4.9/bin/../lib/jline-0.9.94.jar:/opt/zookeeper-3.4.9/bin/../zookeeper-3.4.9.jar:/opt/zookeeper-3.4.9/bin/../src/java/lib/*.jar:/opt/zookeeper-3.4.9/bin/../conf:
     2017-09-28 10:02:24,950 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
     2017-09-28 10:02:24,951 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp
     2017-09-28 10:02:24,951 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
     2017-09-28 10:02:24,951 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux
     2017-09-28 10:02:24,951 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd64
     2017-09-28 10:02:24,951 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=4.4.0-72-generic
     2017-09-28 10:02:24,951 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=deploy
     2017-09-28 10:02:24,951 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/home/deploy
     2017-09-28 10:02:24,951 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/opt/zookeeper-3.4.9/bin
     2017-09-28 10:02:24,952 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=localhost sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@531d72ca
     Welcome to ZooKeeper!
     2017-09-28 10:02:24,967 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server localhost/0:0:0:0:0:0:0:1:2181. Will not attempt to authenticate using SASL (unknown error)
     JLine support is enabled
     2017-09-28 10:02:25,013 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@876] - Socket connection established to localhost/0:0:0:0:0:0:0:1:2181, initiating session
     [zk: localhost(CONNECTING) 0] 2017-09-28 10:02:25,043 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server localhost/0:0:0:0:0:0:0:1:2181, sessionid = 0x25ebcbe589d000d, negotiated timeout = 30000

     WATCHER::

     WatchedEvent state:SyncConnected type:None path:null

     [zk: localhost(CONNECTED) 0] ls /
     [cluster, controller_epoch, controller, brokers, zookeeper, admin, isr_change_notification, consumers, config]
     [zk: localhost(CONNECTED) 2] ls /brokers/ids
     [1, 2, 3]
     [zk: localhost(CONNECTED) 3] ls /brokers/topics
     [mytopic]
     [zk: localhost(CONNECTED) 4] ls /brokers/topics
     [mytopic, second]
     [zk: localhost(CONNECTED) 5]
     ```

   - kafka 测试

     - 创建主题

       ```
       deploy@devops0:/opt/kafka/bin$ ./kafka-topics.sh --create --zookeeper 192.168.1.207:2181 --replication-factor 2 --partitions 1 --topic second

       Created topic "second".

       // 查看zookeeper中kafka的内容
       deploy@devops0:/opt/zookeeper-3.4.9/bin$ ./zkCli.sh  -server localhost
       Connecting to localhost
       2017-09-28 10:11:32,907 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.9-1757313, built on 08/23/2016 06:50 GMT
       2017-09-28 10:11:32,909 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=devops0
       2017-09-28 10:11:32,909 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.8.0_131
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/usr/local/jre1.8.0_131
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/opt/zookeeper-3.4.9/bin/../build/classes:/opt/zookeeper-3.4.9/bin/../build/lib/*.jar:/opt/zookeeper-3.4.9/bin/../lib/slf4j-log4j12-1.6.1.jar:/opt/zookeeper-3.4.9/bin/../lib/slf4j-api-1.6.1.jar:/opt/zookeeper-3.4.9/bin/../lib/netty-3.10.5.Final.jar:/opt/zookeeper-3.4.9/bin/../lib/log4j-1.2.16.jar:/opt/zookeeper-3.4.9/bin/../lib/jline-0.9.94.jar:/opt/zookeeper-3.4.9/bin/../zookeeper-3.4.9.jar:/opt/zookeeper-3.4.9/bin/../src/java/lib/*.jar:/opt/zookeeper-3.4.9/bin/../conf:
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd64
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=4.4.0-72-generic
       2017-09-28 10:11:32,911 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=deploy
       2017-09-28 10:11:32,912 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/home/deploy
       2017-09-28 10:11:32,912 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/opt/zookeeper-3.4.9/bin
       2017-09-28 10:11:32,912 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=localhost sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@446cdf90
       Welcome to ZooKeeper!
       2017-09-28 10:11:32,927 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server localhost/0:0:0:0:0:0:0:1:2181. Will not attempt to authenticate using SASL (unknown error)
       JLine support is enabled
       2017-09-28 10:11:32,967 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@876] - Socket connection established to localhost/0:0:0:0:0:0:0:1:2181, initiating session
       [zk: localhost(CONNECTING) 0] 2017-09-28 10:11:33,007 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server localhost/0:0:0:0:0:0:0:1:2181, sessionid = 0x15ebcbe5acc0029, negotiated timeout = 30000

       WATCHER::

       WatchedEvent state:SyncConnected type:None path:null

       [zk: localhost(CONNECTED) 0] ls /
       [cluster, controller_epoch, controller, brokers, zookeeper, admin, isr_change_notification, consumers, config]
       [zk: localhost(CONNECTED) 1] ls /brokers
       [ids, topics, seqid]
       [zk: localhost(CONNECTED) 2] ls /brokers/ids
       [1, 2, 3]
       [zk: localhost(CONNECTED) 3] ls /brokers/topics
       [mytopic, second]
       ```

     - 创建生产者

       ```
       deploy@manashare:/opt/kafka/bin$ ./kafka-console-producer.sh --broker-list 192.168.1.208:9092 --topic mytopic

       // 以下为命令行输入内容，每次换行将消息发到主题中
       hello
       world
       this is a test
       ```

     - 创建消费者

       ```
       deploy@devops0:/opt/kafka/bin$ ./kafka-console-consumer.sh --zookeeper localhost:2181 --topic mytopic
       Using the ConsoleConsumer with old consumer is deprecated and will be removed in a future major release. Consider using the new consumer by passing [bootstrap-server] instead of [zookeeper].

       // 以下内容是从主题中接收到消息
       hello
       world
       this is a test
       ```

     - 查看主题列表

       ```
       deploy@devops0:/opt/kafka/bin$ ./kafka-topics.sh --list --zookeeper localhost:2181
       // 以下为主题列表
       mytopic
       second
       ```

7. #### TIPS

   - zoo.cfg 注意事项

     ```
     tickTime=2000
     dataDir=/var/lib/zookeeper
     dataLogDir=/var/log/zookeeper
     // 客户端访问端口
     clientPort=2181
     initLimit=10
     syncLimit=5

     // Server配置中主要部分要使用IP，而不是主机名形式
     server.1=192.168.1.208:2888:3888
     server.2=192.168.1.209:2888:3888
     server.3=192.168.1.207:2888:3888
     ```

   - kafka 的 server.properties

     ```
     // 在这里定义对外的主机名
     advertised.host.name=devops2
     # The address the socket server listens on. It will get the value returned from
     # java.net.InetAddress.getCanonicalHostName() if not configured.
     #   FORMAT:
     #     listeners = security_protocol://host_name:port
     #   EXAMPLE:
     #     listeners = PLAINTEXT://your.host.name:9092
     // 这里使用IP形式，不要使用主机名
     listeners=PLAINTEXT://192.168.1.209:9092
     ```

   - ansible-galaxy role 的安装路径

     ```
     // 我是使用Homebrew安装的ansible，ansbile-galaxy安装的Role放在‘/usr/local/etc/ansible/roles’目录下
     /usr/local/etc/ansible/roles ls -l
     total 0
     drwxr-xr-x  14 jiaxi  admin  476 Sep 22 14:55 AnsibleShipyard.ansible-zookeeper
     drwxr-xr-x  15 jiaxi  admin  510 Sep 25 16:40 idealista.kafka-role
     ```

8. #### 资源

   - [Hyperledger fabric][fabric]官方主页
   - [Apache Kafka][kafka]网站
   - [Ansible][ansible-doc]官方文档
   - [Apache Zookeeper][zookeeper]主页
   - [Ansible galaxy][galaxy]网站
   - [AnsibleShipyard ansible-zookeeper role][zookeeper-role]
   - [Idealista kafka-role][kafka-role]
   - [Mr.心弦 Kafka 集群搭建](http://www.cnblogs.com/luotianshuai/p/5206662.html)

9. #### 心得

   整个过程还是算顺利，以前没有接触过 Zookeeper 和 Kafka，花了 2 天时间把这个集群搭起来，主要的时候还是花在寻找合适的 ansible Role，还好 Ubuntu 用得比较熟，Ansile 的代码不算难懂，下载的 Role 有时候需要做一些小的改动。

[fabric]: https://www.hyperledger.org/projects/fabric
[kafka]: https://kafka.apache.org/
[ansible]: https://www.ansible.com/
[ansible-doc]: http://docs.ansible.com/
[zookeeper]: https://zookeeper.apache.org/
[galaxy]: https://galaxy.ansible.com/
[zookeeper-role]: https://galaxy.ansible.com/AnsibleShipyard/ansible-zookeeper/
[kafka-role]: https://galaxy.ansible.com/idealista/kafka-role/
