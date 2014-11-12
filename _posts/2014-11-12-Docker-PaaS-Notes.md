从封装对象和用户工作流两个层面来理解。

###Flynn
####抽象对象

Flynn一切以Application为中心，所有的抽象对象都是从Application引申而来。如下图所示是基本对象的关系描述：

![FlynnObjects][1]

  [1]: https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/FlynnObjects.png

* **Artifact**：表示应用的Docker镜像，通过一个URI来引用镜像
* **Release**：在Artifact的基础上增加了相关的配置，比如启动命令行、绑定的端口、环境变量等。
  
####工作流
