---
layout: post
title: 全民拥抱Docker云--Lhotse系统经验分享
published: true
---

###前言
“只要站在风口，猪也能飞起来”，这碗心灵鸡汤不知道激励了多少英雄豪杰踏上寻风口之路。而现如今，Docker这阵龙卷风呼啸来袭，更让众人生起迎风而上、直冲云霄的欲望。为了找到这风口，数据平台部开始全面拥抱Docker，基于多年的大数据集群管理经验，倾力打造DockerOnGaia云平台（简称[Gaia云](http://km.oa.com/group/docker/articles/show/207094)），并动员将数平自身的核心系统[Lhotse](http://km.oa.com/group/2430/articles/show/165306)、[Hermes](http://km.oa.com/group/597/articles/show/191455)、[Hive](http://km.oa.com/group/2430/articles/show/163736)、[TRC](http://km.oa.com/group/3292/articles/show/175686)、[TDBank](http://km.oa.com/group/3292/articles/show/176743)等全面接入Gaia云。

Lhotse系统作为先锋部队，经过一段时间的改造-验证-灰度，目前现网已经完全接入、稳定运营。本文旨在分享Lhotse接入Gaia云的一些经验与想法，抛砖引玉，期待更多的系统加入队伍，一起在Docker云中探索前行。

###背景介绍

Lhotse是一个大数据任务调度系统，从架构上看是典型的Master-Agent分布式架构，如下图所示，作为调度核心的Base统筹分配任务，交由对应类型的Runner执行：

![Lhotse架构图](https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/LhotseArch.png)

到目前为止，Lhotse线上支持68种Runner，分别对应68种不同的任务类型，集群总机器数将近200台。面对这种复杂多样的系统环境，Lhotse接入Gaia云的核心诉求是**自动化**，通过自动化来提高运维效率。这体现在如下两个方面：

1. 机器操作取代人工操作，将复杂枯燥的工作（比如资源分配、程序部署）交给云平台处理。

2. 通用实现取代重复实现，云平台将一些通用性的基础工作（比如进程监控、自动拉起）抽象成标准化服务，不必重复造轮子。

下面我们将分别从部署、调度、容错、扩缩容及服务发现五个纬度来讨论Lhotse如何在Gaia云中获得自动化。

###自动部署

Lhotse部署遇到的最大难点在于，每个Runner类型所依赖的运行环境是有差异的。例如，Pig Runner需要机器上预装PigClient、Hadoop，而Compute Runner需要预装PLC。极端的情况是，68种Runner对应68种不同的运行环境依赖，且要在200台机器里交叉部署，还要保证各个运行环境完整一致，相互不受影响，这对运维来说是极大的挑战。

所幸的是，Docker正是为了解决此类问题而生的。我们将每个Runner及其运行环境build成一个Docker镜像，push到Gaia云的Registry(docker.oa.com)，通过Registry自动分发到集群的任意机器上。标准化的镜像分发将保证Runner的运行环境跑到哪里都是一致的，且不受其他环境的影响。

![Lhotse镜像分发](https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/LhotseRunnerImages.png)

###自动调度

有了自动化部署，我们可以方便地将Base/Runner发布到集群的任意机器。但接下来的问题是，选择哪台机器在什么时候运行哪个程序——这是调度要解决的问题。

以前，Lhotse集群的调度都是人工静态处理，资源分配粒度只能到机器级别。这导致调度工作量大，资源利用率却不高。如下两图分别展示了各个Runner机器的CPU与内存利用率：

![Lhotse CPU利用率](https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/LhotseCPUUsage.png)

![Lhotse内存利用率](https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/LhotseMemUsage.png)

现在，得益于Gaia云强大的调度能力，Lhotse已经告别了人工资源分配的时代，完全达到调度的自动化。Gaia自动调度主要带来两个方面的收益：

1. 资源分配粒度细分到CPU与内存级别。Lhotse可以根据每个Runner的资源需求来设定具体的CPU个数、内存值。目前，Gaia正在规划支持磁盘、网络带宽等其他资源，实现更精细化的调度。

2. 集群公平地与其他租户共享（租户可以是不同的系统，也可以是同一个系统内的不同模块）。目前，Lhotse与Hermes系统共享集群，在Gaia中通过树形队列来抽象租户层次，如下图所示，Lhotse租户与Hermes租户各自获得集群50%的资源，在Lhotse内部Base租户与Runner租户再按比例瓜分Lhotse的资源。这种多租户的资源调配，即实现了资源的合理利用，也保证了资源共享的公平性。

![Lhotse资源池划分](https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/LhotseQueueHierarchy.png)
    
###自动容错

如前文所言，Lhotse的Runner多达68种，任何一个Runner出错，都有可能影响大批的调度任务。因此，对于每个Runner的监控及出错处理至关重要。

以前，Lhotse通过自定义脚本来监控Runner进程，在发现错误的情况下自动重启或拉起。这种方式存在两点弊端：一是监控脚本本身也需要维护；二是只能做本机重启，无法跨机重试，对于机器挂掉的情况无能为力。

现在，借助Gaia云的自动容错机制，Lhotse从单机容错上升为集群容错，主要体现在两个层面：

1. 自动重试，包括本地重试和跨机重试。Gaia云对于机器宕机、进程异常退出、OutOfMemory等异常都可以做到自动重试告警，且重试次数可以自行设定，本地重试优先于跨机重试。对于有本地上下文信息的Runner，可以设定较高的本地重试次数，以尽量保证效率。

2. 自动屏蔽，即黑名单机制。当机器失败达到预设的次数，将被推上黑名单，在一定的时间内对调度屏蔽，以尽量降低出错的影响，且对业务来说是透明的。

###自动扩缩容
    
弹性资源分配是云计算的核心理念，它的存在是基于这样一个事实——大部分的系统都在某种程度上遵循着二八定律，即有20%的时间处于高峰期，承受成倍于其他时间的负载。

TODO

###自动服务发现

从前文的架构图上可以看到，Lhotse内部最基本的通讯路径是——Runner向Base上报心跳。在静态分配资源时代，Runner是通过写死IP的方式来发现Base。显然，这种方式在资源动态分配的Gaia云中不再可行，需要新的服务发现机制。

这种服务发现机制要满足两个需求：

1. 当服务发生迁移或者新的实例部署时，自动更新地址。

2. 当有多个服务实例时，自动执行负载均衡。

Gaia云为此采纳了基于“Etcd-Confd-HAProxy”的服务发现架构：Etcd作为服务地址的存储中心，HAProxy作为服务访问的代理中心，Confd则是连接前两者的纽带（具体实现可以参考[《服务注册与发现：腾讯Docker云V0.4版本发布》](http://km.oa.com/group/docker/articles/show/209851))。基于这种服务发现机制，Runner发现Base的过程如下图所示：

```seq
Base->Gaia: 部署
Gaia->Etcd: 注册Base地址
Confd->Etcd: 定期获取更新
Confd->HAProxy: reload
Runner->DNS: 域名解析HAProxy
Runner->HAProxy: 访问Base端口
HAProxy->Base: 转发请求
```

###总结

本文分享了Lhotse系统接入Gaia云，实现自动化运维的实践经验： 通过部署、调度自动化，释放了人工运维的压力；通过容错、服务发现自动化，避免了重复造轮子。

作为第一个抵达风口的“猪”，Lhotse乘风而起，扶摇直上云端里，吹响了全民拥抱Docker云的号角。我们会继续探索前行，为大部队开道点灯，后会有期。
