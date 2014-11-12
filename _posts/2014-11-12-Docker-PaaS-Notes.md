从封装对象和用户工作流两个层面来理解。

###Flynn
####抽象对象

Flynn一切以Application为中心，所有的抽象对象都是从Application引申而来。如下图所示是基本对象的关系描述：

![FlynnObjects][1]

  [1]: https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/FlynnObjects.png

* **Artifact**：表示应用的镜像。通过一个URI来引用Docker镜像。

* **Process**：表示应用的进程。通过一个镜像可以启动多个不同的进程，每个进程运行在自己单独的容器里。做个类比，镜像就像Java的JAR包，一个JAR包可以包含多个有main()方法的主类，每个主类都可以用于启动一个进程。

* **Release**：在Artifact的基础上增加了一些不可变(immutable)的进程配置，比如启动命令行、绑定的端口、环境变量等。要修改这些配置，需要生成一个新Release。

* **Formation**: 在Release的基础上增加了一些可变(mutable) 的进程配置，比如复制(replica)个数。
  
####工作流
