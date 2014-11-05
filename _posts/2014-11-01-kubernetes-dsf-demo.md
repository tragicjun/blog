---
layout: post
title: 【分布式服务框架DSF演进系列-2】服务引擎Docker容器化的实践与探索
published: true
---

本系列前文《基于Docker容器和资源调度的探索及平台规划》已经对分布式服务框架DSF的演进动机及思路做了详细地讨论，本文旨在分享我们在服务引擎容器化方面的实践，并以Kubernetes为基础演示资源调度与自动化部署的方案。

###什么是服务引擎
DSF中的核心系统是服务引擎(Data Service Engine，简称DSE)，是一个服务运行容器，提供统一化的通讯协议与调用接口，支持RPC和Servlet协议。简单来说，DSE降低了分布式服务的开发成本，使得开发人员只专注于业务逻辑，无需考虑通讯协议、容灾、负载均衡等通用逻辑。

###服务引擎的容器化改造
天下没有免费的午餐，要把原本跑在物理机或虚拟机上的应用程序迁移到docker容器里，都需要做一定的改造，"The Twelve Factors"对此做了很系统地总结。

**网络地址**

**启动脚本**

**引擎日志**

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


