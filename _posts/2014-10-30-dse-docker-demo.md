---
layout: post
title: DSE镜像制作
published: true
---

**Build image for dse**

Dockerfile

    FROM tegdsf/centos
    RUN svn checkout http://tc-svn.tencent.com/doss/doss_openapi_rep/openapi_proj/branches/commons/DSE/docker_1.0 /root/dse-docker
    RUN cd /root/dse-docker; mvn package
    RUN rm /root/dse-docker/release/*.tar
    RUN mkdir /root/dse-latest
    RUN mv /root/dse-docker/release/dse-*/* /root/dse-latest/
    RUN chmod +x /root/dse-*/bin/start.sh
    RUN chmod +x /root/dse-*/bin/kill.sh
    RUN rm -r /root/dse-docker
    RUN rm -r /tmp/mavenRepository
    EXPOSE 19800
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
