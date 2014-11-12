从封装对象和用户工作流两个层面来理解。

##为什么需要Flynn

为了便于理解Flynn的作用，让我们先来看看传统的Java应用程序从开发到部署再到运行需要经过的几个实体：

- **源代码**：包括*.java、log4j.properties、pom.xml等文件。

- **部署包**：源代码被编译打包后生成一个JAR包，这个就是部署包。

- **部署配置**：包括每个进程的启动命令、环境变量、系统属性等。通常，这些配置会写在一个启动脚本里面。
 
- **进程**：运行Java程序的实体。一个Java程序可以起多个进程，每个进程启动不同的主类(实现了main()方法的类，一个JAR包可以包含多个主类）。

引入Docker后，部署包变成封装了JAR包与依赖环境的镜像，进程变成在容器里运行。但是，从源代码到镜像、从镜像到运行容器这两步转换仍然需要用户手工操作。

这时候Flynn这样的PaaS出场了，在Docker之上封装了整个编译、部署、运行工作流，使得用户只需负责开发和提交代码。

##Flynn的基本对象

下面我们来看看部署包、部署配置、进程这三个实体在Flynn中是如何抽象的。如下图所示是其基本对象的关系描述：

![FlynnObjects][1]

  [1]: https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/FlynnObjects.png

- **App**：表示一个应用，所有其他对象都是围绕App而展开。

- **Artifact**：表示应用的部署包，实际上对应一个Docker镜像。

- **Process**：表示应用的进程。通过一个镜像可以启动多个不同的进程，每个进程运行在自己单独的容器里。

- **Release**：在Artifact的基础上增加了一些不可变(immutable)的静态配置，比如进程的启动命令行、环境变量、绑定端口等。要修改这些配置，需要生成一个新Release。Release这种不可变性是为了方便做Rollback，即应用随时可以回退到之前的Release。

- **Formation**: 在Release的基础上增加了一些可变(mutable) 的动态配置，比如进程的实例(replica)个数。

一个App在任意的运行时刻仅对应一个Formation，一个Formation对应一个Release，一个Release对应一个Artifact。

###工作流
####部署应用

####扩缩容
