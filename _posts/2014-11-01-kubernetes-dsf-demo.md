---
layout: post
title: 【分布式服务框架DSF演进系列-2】服务引擎Docker容器化的实践与探索
published: true
---

本系列前文《基于Docker容器和资源调度的探索及平台规划》已经对分布式服务框架DSF的演进动机及思路做了详细地讨论，本文旨在分享我们在服务引擎容器化方面的实践，并以Kubernetes为基础演示资源调度与自动化部署的方案。

###什么是服务引擎

DSF中的核心系统是服务引擎(Data Service Engine，简称DSE)，是一个服务运行容器，提供统一化的通讯协议与调用接口，支持RPC和Servlet协议。简单来说，DSE降低了分布式服务的开发成本，使得开发人员只专注于业务逻辑，无需考虑通讯协议、容灾、负载均衡等通用逻辑。

###服务引擎的容器化改造

天下没有免费的午餐，要把原本跑在物理机或虚拟机上的应用程序迁移到docker容器里，都需要做一定的改造，"The Twelve Factors"对此做了很系统地总结。下面从网络配置、启动脚本、引擎日志及心跳上报四个方面记录一下我们对DSE所做的改造：

**网络配置**

DSE启动后需要打开一个NIO服务器来接受调用请求，其监听的IP和port是通过一个server.conf文件来配置。假如DSE部署在一台IP为10.136.4.88的机器上，需要做如下配置：

```
#DSE Main TCP Host
taserver.host=10.136.4.88
#DSE Main TCP Port
taserver.port=19800
```

然而，假如DSE运行在docker容器里，这样的配置是无法生效的，因为容器看不到宿主机的网络接口。容器实际上被分配的是一个名为docker0的虚拟网络接口，其IP是从宿主机上随机选择的，通过桥接方式与宿主机互通。因此，方便起见直接将IP配置为0.0.0.0，port配置为19800。这个配置可以写死在文件中保持不变，不必因部署机器的改变而修改：

```
#DSE Main TCP Host
taserver.host=0.0.0.0
#DSE Main TCP Port
taserver.port=19800
```

**启动脚本**

DSE的启动脚本是以后台程序来启动的，这里需要改成前台程序，因为如果没有前台程序，docker容器会立即退出。

**引擎日志**

DSE的系统日志是通过log4j写入特定文件中，这里需要改成写入stdout，因为docker容器默认只会保留stdout中的日志，其他日志文件一旦容器停止将会丢失。P.S：当然，另一种常见的做法是将日志所在的文件夹映射到宿主机上。

**心跳上报**

DSF的另一个核心系统是软负载(详细介绍可以参考wiki链接http://dsfwiki.oa.com/DSF/Wiki.jsp?page=RouterCenter%20Intro)，提供服务发现、路由寻址功能，它要求DSE定期上报心跳，汇报托管在DSE之上的服务的地址信息及其运行状态。原本心跳上报模块是通过从server.conf文件中读取到的IP和port来拼接服务地址的，但如前文所述，server.conf文件中的IP已经修改成了0.0.0.0，显然不能上报这样的IP。

###Docker镜像制作
**Build image for dse**

Dockerfile

```text
### tegdsf/centos在base centos镜像之上安装了jdk, maven及svn
FROM tegdsf/centos
### 通过svn获取服务引擎代码
RUN svn checkout http://tc-svn.tencent.com/doss/doss_openapi_rep/openapi_proj/trunk/commons/DSE /root/dse-trunk
### 通过maven编译打包
RUN cd /root/dse-trunk; mvn package
RUN rm /root/dse-trunk/release/*.tar
RUN mv /root/dse-trunk/release/dse-* /root
RUN chmod +x /root/dse-*/bin/start.sh
RUN chmod +x /root/dse-*/bin/kill.sh
RUN rm -r /root/dse-trunk
RUN rm -r /tmp/mavenRepository
### 服务引擎默认监听19800端口
EXPOSE 19800
### start.sh是服务引擎的启动脚本
ENTRYPOINT ["/root/dse-1.0.3/bin/start.sh"]
```

**Build image for dse service**
```text
FROM tegdsf/dse:1.0.3
RUN svn checkout http://tc-svn.tencent.com/doss/doss_openapi_rep/openapi_proj/trunk/service/dse-service-demo /root/dse-service-demo
RUN cd /root/dse-service-demo; mvn compile
RUN mkdir /root/dse-1.0.3/apps/demo
RUN mv WEB-INF /root/dse-1.0.3/apps/demo
RUN rm -r /root/dse-service-demo
RUN rm -r /tmp/mavenRepository
```

```
docker run -d -p 127.0.0.1:19800:19800 tegdsf/dse-routercenter:1.0
curl -d 'm=queryRoute&p=[{"business":"portaldemo","service":"HelloWorld"}]' localhost:19800/routercenter/RouterCenterService
```

###自动化部署


###


