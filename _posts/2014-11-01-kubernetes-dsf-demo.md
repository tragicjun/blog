---
layout: post
title: 【分布式服务框架DSF演进系列-2】服务引擎Docker容器化的实践与探索
published: true
---

本系列前文《基于Docker容器和资源调度的探索及平台规划》已经对分布式服务框架DSF的演进动机及思路做了详细地讨论，本文旨在分享我们在服务引擎容器化方面的实践，并以Google的Kubernetes系统为基础演示资源调度与自动部署的方案。

###什么是服务引擎

DSF中的核心系统是服务引擎(Data Service Engine，简称DSE)，是一个服务运行容器，提供统一化的通讯协议与调用接口，支持RPC和Servlet协议。简单来说，DSE降低了分布式服务的开发成本，使得开发人员只专注于业务逻辑，无需考虑通讯协议、容灾、负载均衡等通用逻辑。(详细介绍参考wiki链接http://dsfwiki.oa.com/DSF/Wiki.jsp?page=DSE%20Intro)

###服务引擎的容器化改造

前文提到服务引擎可以大大降低开发成本，但是也面临一些限制，比如服务开发语言只支持Java、服务有第三方软件依赖需要手工部署等等。为了解决这些问题，我们决定将服务引擎容器化，使得服务可以与其运行环境(语言环境、软件环境等)一起打包部署。

天下没有免费的午餐，要把原本跑在物理机或虚拟机上的应用程序迁移到docker容器里，都需要做一定的改造，["The Twelve Factors"](http://12factor.net/)对此做了很系统地总结。下面从网络配置、启动脚本、引擎日志及心跳上报四个方面记录一下我们对DSE所做的改造：

**网络配置**

DSE启动后需要打开一个NIO服务器来接受调用请求，其监听的IP和port是通过一个server.conf文件来配置。假如DSE部署在一台IP为10.136.4.88的机器上，需要做如下配置：

    #DSE Main TCP Host
    taserver.host=10.136.4.88
    #DSE Main TCP Port
    taserver.port=19800

然而，假如DSE运行在docker容器里，这样的配置是无法生效的，因为容器看不到宿主机的网络接口。容器实际上被分配的是一个名为docker0的虚拟网络接口，其IP是从宿主机上随机选择的，通过桥接方式与宿主机互通。因此，方便起见直接将IP配置为0.0.0.0，port配置为19800。这个配置可以写死在文件中保持不变，不必因部署机器的改变而修改：

    #DSE Main TCP Host
    taserver.host=0.0.0.0
    #DSE Main TCP Port
    taserver.port=19800

**启动脚本**

DSE的启动脚本是以后台程序来启动的，这里需要改成前台程序，因为如果没有前台程序，docker容器会立即退出。

**引擎日志**

DSE的系统日志是通过log4j写入特定文件中，这里需要改成写入stdout，因为docker容器默认只会保留stdout中的日志，其他日志文件一旦容器停止将会丢失。P.S：当然，另一种常见的做法是将日志所在的文件夹映射到宿主机上。

**心跳上报**

DSF的另一个核心系统是软负载(详细介绍可以参考wiki链接http://dsfwiki.oa.com/DSF/Wiki.jsp?page=RouterCenter%20Intro )，提供服务发现、路由寻址功能，它要求DSE定期上报心跳，汇报托管在DSE之上的服务的地址信息及其运行状态。原本心跳上报模块是通过从server.conf文件中读取到的IP和port来拼接服务地址的，但如前文所述，server.conf文件中的IP已经修改成了0.0.0.0，显然不能上报这样的IP。

由于目前还没有很好的方式在docker容器里获取其宿主机的IP，我们采取的临时方案是：在docker启动容器时，通过环境变量将宿主机的IP及该容器映射的宿主机port注入容器。

###服务引擎的Docker镜像制作

在对DSE进行简单的改造后，就可以来制作docker镜像了，首先是编写Dockerfile，内容如下：

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

然后通过docker build命令来构建：

    docker build -t tegdsf/dse .

最后，可以对构建好的镜像做简单的测试：

    # docker run -d -p 127.0.0.1:19800:19800 tegdsf/dse
    4c55f2081eac
    # curl localhost:19800/internal/heartbeat.jsp
    <html>
    Heartbeat Page.
    </html>

测试通过后，我们可以把镜像push到远程仓库，这里推荐数平gaia团队搭建的私有仓库docker.oa.com:8080(详细介绍参考链接http://km.oa.com/group/docker/articles/show/205373 )。

###自动部署

本系列前文《基于Docker容器和资源调度的探索及平台规划》详细描述了我们结合Gaia系统来做资源调度和自动部署的方案，这里分享一下另一种基于Kubernetes的方案。Kubernetes是Google开源的容器集群管理系统。它构建于docker技术之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等整一套功能，本质上可看作是基于容器技术的mini-PaaS平台。具体的原理介绍和实践操作可以参考文章http://km.oa.com/group/597/articles/show/205884。

首先，我们搭建了由两台机器组成的Kubernetes集群，一台机器既做master也做slave，通过命令行工具查看slave(称为minion)如下：

    $ kubecfg -h 10.6.207.17 list /minions
    Minion identifier
    ----------
    10.6.207.17
    10.6.207.228

接下来，我们通过Kubernetes的replicationController对象来部署多个服务实例，编写json描述文件如下：

    {
      "id": "dse-demo",
      "apiVersion": "v1beta1",
      "kind": "ReplicationController",
      "desiredState": {
        "replicas": 2,
        "replicaSelector": {"name": "dse-demo"},
        "podTemplate": {
          "desiredState": {
             "manifest": {
               "version": "v1beta1",
               "id": "dse-demo",
               "containers": [{
                 "name": "dse-demo",
                 "image": "tegdsf/dse",
                  "imagePullPolicy": "PullIfNotPresent",
                 "ports": [{"containerPort": 19800, "hostPort": 19800}]
               }]
             }
           },
           "labels": {"name": "dse-demo"}
          }},
      "labels": {"name": "dse-demo"}
    }

可以看到，通过"replicas"字段可以指定服务实例的个数。最后，用命令行工具提交描述文件：

    $ kubecfg -h 10.6.207.17 -c dseController.json create /replicationControllers
    ID                  Image(s)            Selector            Replicas
    ----------          ----------          ----------          ----------
    dse-demo            tegdsf/dse          name=dse-demo       2

可以看到2个实例已经部署成功，并处于"Running"状态：

    $ kubecfg -h 10.6.207.17 list /pods
    ID                                     Image(s)            Host                Labels                                         Status
    ----------                             ----------          ----------          ----------                                     ----------
    9a64445a-6557-11e4-b410-001f290cf88c   tegdsf/dse          10.6.207.228/       name=dse-demo,replicationController=dse-demo   Running
    9a646fb6-6557-11e4-b410-001f290cf88c   tegdsf/dse          10.6.207.17/        name=dse-demo,replicationController=dse-demo   Running

通过docker ps也可以看到被Kubernetes启动的容器：

<pre><code class="bash">
$ docker ps
CONTAINER ID        IMAGE                     COMMAND                CREATED             STATUS              PORTS                      NAMES
37fab78246e0        tegdsf/dse:latest         "/bin/sh -c /root/ds   3 minutes ago       Up 3 minutes                                   k8s_dse-demo.4314872b_9a646fb6-6557-11e4-b410-001f290cf88c.default.etcd_1415238765_7e8d8da7   
8f91f09d6d4e        kubernetes-pause:latest   "/pause"               3 minutes ago       Up 3 minutes        0.0.0.0:19800->19800/tcp   k8s_net.40d9846a_9a646fb6-6557-11e4-b410-001f290cf88c.default.etcd_1415238765_751ba7fd       
</code></pre>

###后续工作

经过前文所述的工作，我们已经把服务框架DSF的服务引擎DSE容器化，并试验了自动部署的方案。但是，真正要应用于生产环境，还有不少工作要做，这里提两点方向，会在后面的系列文章中分享我们的探索与实践。

####服务发现

当我们把服务通过Gaia或Kubernetes这样的系统来做调度和部署时，服务真正的运行机器是动态可变的，面临地问题是服务消费者如何及时地发现服务。Kubernetes基于代理的思路包装了service的概念，但主要是用于系统内部各服务之间相互发现，没有解决外部消费者的发现问题。接下来，我们考虑将服务框架的软负载与Kubernetes结合，来为外部客户提供服务路由功能。

####日志收集

Docker容器的日志收集这块，业界已经有很多人在探索，目标都是将传统成熟的日志收集方案(比如，Logstash + ElasticSearch +Kibana)平滑地迁移到基于容器的平台系统。接下来，我们也将探索各种不同的方案，筛选出最合适自己的。
