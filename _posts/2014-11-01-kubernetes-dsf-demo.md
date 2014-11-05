---
layout: post
title: 服务容器化的实践分享
published: true
---

**Build image for dse**

Dockerfile

```text
#tegdsf/centos在base centos镜像之上安装了jdk, maven及svn
FROM tegdsf/centos
#通过svn获取服务引擎代码
RUN svn checkout http://tc-svn.tencent.com/doss/doss_openapi_rep/openapi_proj/trunk/commons/DSE /root/dse-trunk
#通过maven编译打包
RUN cd /root/dse-trunk; mvn package
RUN rm /root/dse-trunk/release/*.tar
RUN mv /root/dse-trunk/release/dse-* /root
RUN chmod +x /root/dse-*/bin/start.sh
RUN chmod +x /root/dse-*/bin/kill.sh
RUN rm -r /root/dse-trunk
RUN rm -r /tmp/mavenRepository
EXPOSE 19800
#start.sh是服务引擎的启动脚本
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
