---
title: "Hyperledge fabric peer 流程分析"
date: 2018-12-08 14:17:39
tags:
  - hyperledge
  - fabric
  - blockchain
categories:
  - code
---

1. ## 背景

   最近花了大概 2 个星期左右阅读了 fabric-1.2.1 中与 peer 相关的源码，着重阅读了与 Chaincode(链码)交互相关的逻辑，因为之前并没有找到一些特别好的参考资料，在这里还是耽误了些时间，不过还好，虽是走了些弯路，但还算是走出来了。

2. ## 关键库

   在[Hyperledger Fabric 源代码分析与深入解读][book]这本书中，第 3 章介绍了相关的库，如日志、配置文件、[grpc][grpc]，Error，不过这本书是针对的 fabric1.0 版的代码，我看的源代码是 1.2.1 版，所以有些地方对不上，只能作为一个参考；下面列出的库还是很重要的：

   - Logging，日志库，可以按级别打印消息，可用来调试
   - [viper][viper]，配置文件读取，源码里相当多的地方要读取配置文件中的内容，所以你要知道[viper][viper]的工作原理，如去哪里找配置文件，及如何查找相关的配置项
   - [grpc][grpc]，fabric 是个分布式系统，各个组件可能并没有在一台主机上，那各组件之间如何通信，这里就用到[grpc][grpc]框架了，强烈建议你先去看[grpc][grpc]的官方文档，学习写个 Demo，再回头来看代码。
   - [cobra][cobra], 用 go 实现的命令行开发库，支持多级子命令，如 peer node start，cobra 的用法也比较好理解，看一下官方文档就好了。

3. ## 相关工具、方法

   工欲善其事，必先利其器，一个好的工具可以让你少走很多弯路，

   - 源码阅读工具  
     推荐使用 JetBrains Goland，它是专来 golang 环境设置的，操作起来比较简单，直观
   - 调试环境  
     推荐使用 docker+remote debug 方式，在 docker 容器中安装 dlv 远程调试工具，这种方式并比较简单，只需要对原来的容器稍做定制，就可以通过命令行或 IDE 方式连接到 docker, 这种方式支持大多数的 IDE 环境，如 Visual code 和 goland,请参考这篇文章[Debugging containerized Go applications][debug]

4. ## 流程

   - 总流程  
     peer/main.go --> node/start.go 对应 peer node start 命令， peer 模块从 start.go 中的 serv 函数开始
     ![peer_start](/img/fabric/peer-start.png)

     可以看出 peer node start 就是启动一系列的 grpc 服务，供其他模块调用，下面具体看一下 Chaincode support 服务

   - Chaincode-support
     该服务主要用来响应 Chaincode 的相关调用。
     ![Chaincode_handle](/img/fabric/chaincode.png)

   - Chaincode docker  
     大家已经知道 fabric 的 Chaincode 是通过 Docker 运行的，那其中到底是个什么逻辑呢？ 我们来跟踪一下。前面已经知道 Chaincode-support 服务启动后，从 Created 状态一直到 Ready 状态，这时就可以开始处理链码相关调用了，如下面这样

     ```go
     func (h *Handler) handleMessageReadyState(msg *pb.ChaincodeMessage) error {
        switch msg.Type {
        case pb.ChaincodeMessage_COMPLETED, pb.ChaincodeMessage_ERROR:
            h.Notify(msg)

        case pb.ChaincodeMessage_PUT_STATE:
            go h.HandleTransaction(msg, h.HandlePutState)
        case pb.ChaincodeMessage_DEL_STATE:
            go h.HandleTransaction(msg, h.HandleDelState)
        case pb.ChaincodeMessage_INVOKE_CHAINCODE:
            go h.HandleTransaction(msg, h.HandleInvokeChaincode)

        case pb.ChaincodeMessage_GET_STATE:
            go h.HandleTransaction(msg, h.HandleGetState)
        case pb.ChaincodeMessage_GET_STATE_BY_RANGE:
            go h.HandleTransaction(msg, h.HandleGetStateByRange)
        case pb.ChaincodeMessage_GET_QUERY_RESULT:
            go h.HandleTransaction(msg, h.HandleGetQueryResult)
        case pb.ChaincodeMessage_GET_HISTORY_FOR_KEY:
            go h.HandleTransaction(msg, h.HandleGetHistoryForKey)
        case pb.ChaincodeMessage_QUERY_STATE_NEXT:
            go h.HandleTransaction(msg, h.HandleQueryStateNext)
        case pb.ChaincodeMessage_QUERY_STATE_CLOSE:
            go h.HandleTransaction(msg, h.HandleQueryStateClose)

        default:
            return fmt.Errorf("[%s] Fabric side handler cannot handle message (%s) while in ready state", msg.Txid, msg.Type)
        }

        return nil
     }
     ```

     其中比较主要的服务 HandleInvokeChaincode，我们来看看

     ```go
     func (h *Handler) HandleInvokeChaincode(){
        ...
        ctxt = context.WithValue(ctxt, TXSimulatorKey, txsim)
        ctxt = context.WithValue(ctxt, HistoryQueryExecutorKey, historyQueryExecutor)
        ...
        responseMessage, err := h.Invoker.Invoke(ctxt, cccid, cciSpec)
     }
     ```

     ```go
     func (cs *ChaincodeSupport) Invoke(){
        var cctyp pb.ChaincodeMessage_Type
        switch spec.(type) {
        case *pb.ChaincodeDeploymentSpec:
            cctyp = pb.ChaincodeMessage_INIT
        case *pb.ChaincodeInvocationSpec:
            cctyp = pb.ChaincodeMessage_TRANSACTION
        default:
            return nil, errors.New("a deployment or invocation spec is required")
        }
        // 这里非常重要，这里是要启动链码，如果是普通合约，则使用Docker容器来启动
        err := cs.Launch(ctxt, cccid, spec)
        ...
        // 开始执行链码中对应的函数
        return cs.execute(ctxt, cccid, ccMsg)
     }
     ```

     继续跟踪 Launch

     ```go
     func (cs *ChaincodeSupport) Launch(){
        ctx = context.WithValue(ctx, ccintf.GetCCHandlerKey(), cs)
        return cs.Launcher.Launch(ctx, cccid, spec)
     }
     ```

     进入到 RuntimeLauncher.Launch

     ```go
     func (r *RuntimeLauncher) Launch(){
         err := r.start(ctx, cccid, cds)
     }
     ```

     进入到 RuntimeLauncher.start

     ```go
     func (r *RuntimeLauncher) start(){
         err := r.Runtime.Start(ctx, cccid, cds)
     }
     ```

     进入到 ContainerRuntime.start

     ```go
     func (c *ContainerRuntime) Start(){
        if err := c.Processor.Process(ctxt, vmtype, scr); err != nil {
        return errors.WithMessage(err, "error starting container")
     }
     ```

     继续到 VMController.Process

     ```go
     func (vmc *VMController) Process(ctxt context.Context, vmtype string, req VMCReq) error {
        v := vmc.newVM(vmtype)
        ccid := req.GetCCID()
        id := ccid.GetName()

        vmc.lockContainer(id)
        defer vmc.unlockContainer(id)
        // 根据vmtype执行相应的do， 如果是一般使用，使用dockerVM执行，如果是系统合约，则使用InporcVM执行
        return req.Do(ctxt, v)
     }
     ```

     启动 docker 容器，如果没有相关镜像，还要 build 一个镜像出来，再使用镜像启动容器

     ```go
     func (vm *DockerVM) Start(){
         // 创建Docker容器，该容器中已经包含了要调用的链码程序
         err = vm.createContainer(ctxt, client, imageName, containerName, args, env, attachStdout)
         if err != nil {
         // 这里如果没有相关的镜像，这里还要负责编译一个镜像出来
            reader, err1 := builder.Build()
            if err1 != nil {
                dockerLogger.Errorf("Error creating image builder for image <%s> (container id <%s>), "+
                    "because of <%s>", imageName, containerName, err1)
            }

            if err1 = vm.deployImage(client, ccid, args, env, reader); err1 != nil {
                return err1
            }

            dockerLogger.Debug("start-recreated image successfully")
            if err1 = vm.createContainer(ctxt, client, imageName, containerName, args, env, attachStdout); err1 != nil {
                dockerLogger.Errorf("start-could not recreate container post recreate image: %s", err1)
                return err1
            }
         // 启动Docker容器
         err = client.StartContainer(containerName, nil)
         }
     ```

     回到 ChaincodeSupport.Invoke()

     ```go
     func (cs *ChaincodeSupport) Invoke(){
         ...
         return cs.execute(ctxt, cccid, ccMsg)
     }
     ```

     在 cs.execute 函数中，ChaincodeSupport 向 docker 容器发送消息，请求执行链码对应的功能，容器中的链码收到消息后，处理请求，并返回结果。主流程并不是太复杂，下图可以总结：

     ![invoke_call](/img/fabric/chaincode-ps.png)

   - docker build
     现在把目光放到如何把链码编译成镜像，并启动容器上来，还记得前面的那段吧？

     ```go
     type Builder interface {
        Build() (io.Reader, error)
     }
     ```

     JetBrains Goland 这个工具确实很人性化，接口在哪里实现的，都会直接列出来

     ```go
     // Build a tar stream based on the CDS
     func (b *PlatformBuilder) Build() (io.Reader, error) {
        return platforms.GenerateDockerBuild(b.DeploymentSpec)
     }
     ```

     ```go
     func GenerateDockerBuild(){

        inputFiles := make(InputFiles)

        // ----------------------------------------------------------------------------------------------------
        // Determine our platform driver from the spec
        // ----------------------------------------------------------------------------------------------------
        platform, err := _Find(cds.ChaincodeSpec.Type)
        if err != nil {
            return nil, fmt.Errorf("Failed to determine platform type: %s", err)
        }

        // ----------------------------------------------------------------------------------------------------
        // Generate the Dockerfile specific to our context
        // ----------------------------------------------------------------------------------------------------
        dockerFile, err := _generateDockerfile(platform, cds)
        if err != nil {
            return nil, fmt.Errorf("Failed to generate a Dockerfile: %s", err)
        }

        inputFiles["Dockerfile"] = dockerFile

        // ----------------------------------------------------------------------------------------------------
        // Finally, launch an asynchronous process to stream all of the above into a docker build context
        // ----------------------------------------------------------------------------------------------------
        input, output := io.Pipe()

        go func() {
            gw := gzip.NewWriter(output)
            tw := tar.NewWriter(gw)
            err := _generateDockerBuild(platform, cds, inputFiles, tw)
            if err != nil {
                logger.Error(err)
            }

            tw.Close()
            gw.Close()
            output.CloseWithError(err)
        }()

        return input, nil
     }
     ```

     这个函数干了 3 件事

     - 判断是哪种语言编写的链码
       fabric-1.2.1 已经开始支持 4 种语言的链码了，golang/java/car/javascrit，不同的语言环境，需要不同的编译工具和依赖环境，最终将链码打包成一个可执行的程序，这里以 golang 语言为例分析。
     - 生成一个 Dockerfile 文件，熟悉 Docker 的同学看到这里，可以已经有点眉目了。
     - 启动一个 ccenv 容器编译链码，并生成一个新的镜像文件，专用于执行链码

     我们来看一下第 2 步，生成的 Dockerfile 文件

     ```go
     func (goPlatform *Platform) GenerateDockerfile(cds *pb.ChaincodeDeploymentSpec) (string, error) {

        var buf []string

        buf = append(buf, "FROM "+cutil.GetDockerfileFromConfig("chaincode.golang.runtime"))
        buf = append(buf, "ADD binpackage.tar /usr/local/bin")

        dockerFileContents := strings.Join(buf, "\n")

        return dockerFileContents, nil
     }
     ```

     非常典型的 Dockfile 文件，进入在一个正在运行 peer 的容器，查看/etc/hyperledger/img/fabric/core.yml 文件，最终出来的大概是如下这个样子

     ```Dockfile
     // FROM $(BASE_DOCKER_NS)/fabric-baseos:$(ARCH)-$(BASE_VERSION)
     FROM hyperledger/fabric-baseos:amd64-0.4.13
     ADD binpackage.tar /usr/local/bin
     ```

     这个 Dockerfile 要求一个 binpackage.tar 文件，这个文件在哪呢？ 接着往下看

     ```go
     func (goPlatform *Platform) GenerateDockerBuild(cds *pb.ChaincodeDeploymentSpec, tw *tar.Writer) error {
        spec := cds.ChaincodeSpec
        ...
        codepackage := bytes.NewReader(cds.CodePackage)
        binpackage := bytes.NewBuffer(nil)
        err = util.DockerBuild(util.DockerBuildOptions{
            Cmd:          fmt.Sprintf("GOPATH=/chaincode/input:$GOPATH go build -tags \"%s\" %s -o /chaincode/output/chaincode %s", gotags, ldflagsOpt, pkgname),
            InputStream:  codepackage,
            OutputStream: binpackage,
        })
        ...
        return cutil.WriteBytesToPackage("binpackage.tar", binpackage.Bytes(), tw)
     }
     ```

     看到 Cmd 那行，看起来是不是用 go build 在编译 go 代码？
     其中 util.DockerBuild()如下

     ```go
     func DockerBuild(opts DockerBuildOptions) error {
        client, err := cutil.NewDockerClient()
        ...
        // 这里镜像文件是 hyperledger/fabric-ccenv:latest
        if opts.Image == "" {
            opts.Image = cutil.GetDockerfileFromConfig("chaincode.builder")
        }
        //-----------------------------------------------------------------------------------
        // 确认镜像是否存在，如果没有，则先使用docker pull下载镜像文件
        //-----------------------------------------------------------------------------------
        _, err = client.InspectImage(opts.Image)
        if err != nil {
            logger.Debugf("Image %s does not exist locally, attempt pull", opts.Image)

            err = client.PullImage(docker.PullImageOptions{Repository: opts.Image}, docker.AuthConfiguration{})
            if err != nil {
                return fmt.Errorf("Failed to pull %s: %s", opts.Image, err)
            }
        }
        //-----------------------------------------------------------------------------------
        // Upload our input stream
        //-----------------------------------------------------------------------------------
        err = client.UploadToContainer(container.ID, docker.UploadToContainerOptions{
            Path:        "/chaincode/input",
            InputStream: opts.InputStream,
        })
        ...
        err = client.StartContainer(container.ID, nil)
        ...
        retval, err := client.WaitContainer(container.ID)
        ...
        err = client.DownloadFromContainer(container.ID, docker.DownloadFromContainerOptions{
            Path:         "/chaincode/output/.",
            OutputStream: opts.OutputStream,
        })
     }
     ```

     这里大概意思是使用 hyperledger/ccenv 这个镜像来编译链码，被编译出的可执行文件为/chaincode/output/chaincode，并将 docker 里的/chaincode/output/这个目录的内容打包，最终生成前面所需要的 binpackage.tar，最终再根据 Dockerfile 生成了我们需要的镜像文件。到这里为止，链码镜像的生成就算是结束了，不过还有个小的问题:

     - 链码编译的时候的依赖是怎么解决的
       这里我们看一看 ccenv 这个镜像是怎么生成的(images/ccenv/Dockerfile.in)

       ```Dockerfile
        FROM _BASE_NS_/fabric-baseimage:_BASE_TAG_
        COPY payload/chaintool payload/protoc-gen-go /usr/local/bin/
        ADD payload/goshim.tar.bz2 $GOPATH/src/
        RUN mkdir -p /chaincode/input /chaincode/output
       ```

       其中有一行将 goshim.tar.bz2 拷贝到$GOPATH/src目录，$GOPATH/src 目录下一般放的都 go 的源码，go 程序在编译的时候会在这个目录下搜索相关的库，那这里 goshim.tar.bz2 是又是怎么来的？

       ```Makefile
        GOSHIM_DEPS = $(shell ./scripts/goListFiles.sh $(PKGNAME)/core/chaincode/shim)

        $(BUILD_DIR)/goshim.tar.bz2: $(GOSHIM_DEPS)
            @echo "Creating $@"
            @tar -jhc -C $(GOPATH)/src $(patsubst $(GOPATH)/src/%,%,$(GOSHIM_DEPS)) > $@

        $(BUILD_DIR)/image/ccenv/payload:      $(BUILD_DIR)/docker/gotools/bin/protoc-gen-go \
                $(BUILD_DIR)/bin/chaintool \
                $(BUILD_DIR)/goshim.tar.bz2
       ```

       我们看 goshim.tar.bz2 这个压缩文件，是通过一个脚本得到 GOSHIM_DEPS，再把 GOSHIM_DEPS 打成 tar.bz2 包

       ![docker-build](/img/fabric/chaincode-build.png)

   - shim  
     在没看代码之间，一直不理解 shim(垫片)是个啥？大家在编写 Chaincode 的时候是不是总有这么一段：

     ```go
     func main() {
        // Create a new Smart Contract
        err := shim.Start(new(SmartContract))
        if err != nil {
            fmt.Printf("Error creating new Smart Contract: %s", err)
        }
     }
     ```

     这一段其实是在 Docker 容器里执行的，

     ```go
     // chaincodes.
     func Start(cc Chaincode) error {
        //mock stream not set up ... get real stream
        if streamGetter == nil {
            streamGetter = userChaincodeStreamGetter
        }

        stream, err := streamGetter(chaincodename)
        if err != nil {
            return err
        }

        err = chatWithPeer(chaincodename, stream, cc)

        return err
     }
     ```

     合约里调用 shim.Start()，就对应到了下面的这面代码，这里主要的用途是建立一个对话的通道，将来 peer 和 docker 的消息交换都是这个通道进行。

5. ## 总结

   内容有点多，其实有 2 大块内容，

   - SDK/peer/docker 是如何互相配合的
   - 链码是如何被编译成镜像，运行起来的
   - shim(垫片)

6. ## 参考资料
   - [Hyperledger Fabric 源代码分析与深入解读][book]
   - [Hyperledger Chaincode 啟動過程-掃文資訊][start]
   - [grpc][grpc]
   - [viper][viper]
   - [Debugging containerized Go applications][debug]
   - [cobra][cobra]

[book]: https://read.douban.com/ebook/57504608/
[start]: https://hk.saowen.com/a/2dbad2b467f6331a5ad8e73d46500aa5efa9b848f30b5dfdcca0b475bbddb860
[grpc]: https://grpc.io/
[viper]: https://github.com/spf13/viper
[debug]: https://blog.jetbrains.com/go/2018/04/30/debugging-containerized-go-applications/
[cobra]: https://github.com/spf13/cobra
