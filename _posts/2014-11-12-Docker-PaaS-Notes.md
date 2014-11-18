---
layout: post
title: Flynn初探：基于Docker的PaaS平台
published: true
---

##### 作者：[TragicJun](http://blog.csdn.net/zhangjun2915)

从封装对象和用户工作流两个层面来理解。

##为什么需要Flynn

为了便于理解Flynn的作用，让我们先来看看传统的Java应用程序从开发到部署再到运行需要经过的几个实体：

![AppPhases][3]

  [3]: https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/appPhases.png

- **源代码**：包括*.java、log4j.properties、pom.xml等文件。

- **部署包**：源代码被编译打包后生成一个JAR包，这个就是部署包。

- **部署配置**：包括每个进程的启动命令、环境变量、系统属性等。通常，这些配置会写在一个启动脚本里面。
 
- **进程**：运行Java程序的实体。一个Java程序可以起多个进程，每个进程启动不同的主类(实现了main()方法的类，一个JAR包可以包含多个主类）。

引入Docker后，部署包变成封装了JAR包与依赖环境的镜像，进程变成在容器里运行。但是，从源代码到镜像、从镜像到运行容器这两步转换仍然需要用户手工操作。尤其是后者的转换，涉及到集群资源调度、自动部署、容器管控等一系列的复杂流程。

这时候Flynn这样的PaaS出场了，在Docker之上封装了整个编译、部署、运行工作流，使得用户只需提交代码即可完成开发到部署运行环节的转换：

- **开发到部署**：用户通过git提交源代码，由Flynn自动构建镜像，并提供版本的管理——用户可以创建新版本(提交新代码或修改部署配置)、回滚老版本等。

- **部署到运行**：Flynn自动选择运行机器，为每个进程副本部署启动单独的容器，并提供进程的管理——用户可以做扩缩容、查看日志等。

##Flynn的基本对象

下面我们来看看部署包、部署配置、进程这三个实体在Flynn中是如何抽象的。如下图所示是其基本对象的关系描述：

![FlynnObjects][1]

  [1]: https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/FlynnObjects.png

- **App**：表示一个应用，所有其他对象都是围绕App而展开。

- **Artifact**：表示应用的部署包，实际上对应一个Docker镜像。

- **Process**：表示应用的进程。通过一个镜像可以启动多个不同的进程，每个进程运行在自己单独的容器里。

- **Release**：是应用*发布态*的抽象表示。它在Artifact的基础上增加了一些不可变(immutable)的静态配置，比如每个进程的启动命令行、环境变量、绑定端口、等。要修改这些配置，需要生成一个新Release。Release这种不可变性是为了方便做Rollback，即应用随时可以回退到之前的Release。

- **Formation**: 是应用*运行态*的抽象表示。它在Release的基础上增加了可变(mutable)的动态配置，即每个进程的副本(replica)个数。

- **Job**: 是进程副本的抽象表示，每个Job对应一个运行容器。因此，在后文中可以看到，Job是资源调度的基本单元。

##Flynn的架构层次

![FlynnComponents][2]

  [2]: https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/FlynnComponents.png

如上图所示，Flynn的架构自下而上分为两个层级——Layer 0和Layer 1。简单地理解，可以认为Layer 1负责接受用户请求，封装成应用的*运行指令*，再由Layer 0解决*在哪里运行*、*以什么方式运行*的问题。具体一点讲，Layer 0面向的对象是Formation，负责将底层的集群资源封装成可执行Formation的一台主机；Layer 1面向的对象是App，负责将App从源代码构建成Artifact，进而封装成Formation提交给Layer 0去执行。

这种分工明确的层次划分，使整个系统非常灵活，相互松耦合，便于任意组件的替换(比如，甚至可以把Layer 0替换成不用容器去执行Formation)。下面总结一下组成两个层级的各个组件及其功能(所有组件自身都可以运行在容器里)：

###Layer 0

- **Scheduler**: 资源调度器，定期从Layer 0获取Formation的更新，再根据每个Formation的部署配置生成一个个的Job，最后从集群中选择合适的机器去运行这些Job。

- **Host Service**: 运行在集群每台机器上的agent，负责管控运行在本机的容器，并收集运行状态信息。

- **Host Leader**：一个特殊的Host Service，做为"cluster leader"，负责维护整个集群的状态信息(比如有哪些机器、每台机器上运行的Job等)，并提供给Scheduler用于资源调度。

- **Discoverd**：基于etcd的服务发现模块，提供容器间的发现机制。实际上，Flynn自身的组件间通讯也是通过Discoverd来相互发现的。

###Layer 1

- **CLI**：提供给用户使用的命令行工具。

- **Controller**：为Flynn系统的入口，封装了核心对象(比如app/artifact/release/job)的增删改查操作，以RESTFul接口方式提供给外部客户和内部组件调用。它维护的REST对象将持久化到postgre数据库。Layer 0的Scheduler就是通过Controller的接口来获取Formation更新的。

- **GitReceiver**：接受用户git push源代码的SSH服务器。接受到git push后将触发Receiver。

- **Receiver**：基于buildpack机制，利用SlugBuilder从源代码包构建slug包。buildpack和slug都是从Heroku借鉴过来的概念。简单地理解，buildpack是一组用于构建源代码的脚本，buildpack可以多种多样，每个buildpack可构建某种类型的源代码，这种类型可以是不同的语言(比如Java、PHP)、不同的构建方式(比如maven、gradle)；而slug则是buildpack构建生成的部署包，包含了编译输出文件、依赖库文件等运行环境。

- **BlobStore**: HTTP文件服务器，用于上传/下载slug包。

- **SlugBuilder**：接受源代码包，基于某种buildpack构建生成slug包。选择哪一种buildpack可以显式地指定，也可以由SlugBuilder根据源文件自动匹配。

- **SlugRunner**：运行slug包，会从BlobStore下载应用的slug包。
