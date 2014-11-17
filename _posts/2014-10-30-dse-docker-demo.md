---
layout: post
title: DSE镜像制作
published: true
---

**Build image for DSE**

Dockerfile

    ### tegdsf/centos在base centos镜像之上安装了jdk, maven及svn
    FROM tegdsf/centos
    ### 通过svn获取服务引擎代码
    RUN svn checkout http://tc-svn.tencent.com/doss/doss_openapi_rep/openapi_proj/branches/commons/DSE/docker_1.0 /root/dse-docker
    ### 通过maven编译打包
    RUN cd /root/dse-docker; mvn package
    ### 将部署包拷贝到特定文件夹
    RUN rm /root/dse-docker/release/*.tar
    RUN mkdir /root/dse-latest
    RUN mv /root/dse-docker/release/dse-*/* /root/dse-latest/
    ### 对启动/停止脚本设置可执行权限
    RUN chmod +x /root/dse-*/bin/start.sh
    RUN chmod +x /root/dse-*/bin/kill.sh
    RUN rm -r /root/dse-docker
    RUN rm -r /tmp/mavenRepository
    ### 服务引擎默认监听19800端口
    EXPOSE 19800
    ### start.sh是服务引擎的启动脚本
    ENTRYPOINT /root/dse-latest/bin/start.sh

Build command

    docker build -t tegdsf/dse --no-cache-true .

**Build image for dse service**

    FROM tegdsf/dse
    RUN svn checkout http://tc-svn.tencent.com/doss/doss_openapi_rep/openapi_proj/trunk/service/dse-service-demo /root/dse-service-demo
    RUN cd /root/dse-service-demo; mvn compile
    RUN mkdir /root/dse-latest/apps/demo
    RUN mv /root/dse-service-demo/WEB-INF /root/dse-latest/apps/demo
    RUN rm -r /root/dse-service-demo
    RUN rm -r /tmp/mavenRepository

Build command

    docker build -t tegdsf/dse-demo --no-cache-true . 

Start command

    docker run -d -p 10.6.207.228:19800:19800 -e DSE_DOCKER_HOST=10.6.207.228 -e DSE_DOCKER_PORT=19800 tegdsf/dse-demo

**Build image for routercenter**

    ### tegdsf/dse在base centos镜像之上安装了jdk和DSE
    FROM tegdsf/dse
    ### 通过svn获取软负载代码
    RUN svn checkout http://tc-svn.tencent.com/doss/doss_openapi_rep/openapi_proj/branches/commons/routercenter/1.1.1 /tmp/routercenter
    ### 通过maven编译打包
    RUN cd /tmp/routercenter/routercenter-service; mvn compile
    ### 将部署包拷贝到DSE的apps文件夹下
    RUN mkdir /root/dse-latest/apps/routercenter
    RUN mv /tmp/routercenter/routercenter-service/WEB-INF /root/dse-latest/apps/routercenter
    RUN rm -r /tmp/routercenter
    RUN rm -r /tmp/mavenRepository
    ### 关闭Internal服务，包括心跳上报、指标收集
    ENV internal_enable false

Start command

    docker run -d -P -e CONFIG_MODE=LOCAL  tegdsf/routercenter
