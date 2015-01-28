---
layout: post
title: 全民拥抱Docker云--Lhotse系统经验分享
published: true
---

###前言

“只要站在风口，猪也能飞起来”，这碗心灵鸡汤不知道激励了多少英雄豪杰踏上寻风口之路。而现如今，Docker这阵龙卷风强势来袭，更令众人产生迎风而上、直冲云霄的欲望。为了找到这风口，数据平台部开始全面拥抱Docker，依托多年的大数据集群管理经验，倾力打造DockerOnGaia云平台（简称Gaia云），并将数平自身的核心系统Lhotse、Hermes、Hive、TRE、TDBank等全面接入Gaia云。

Lhotse系统作为先锋部队，经过一段时间的改造-验证-灰度，目前现网已经完全接入、稳定运营。本文旨在分享Lhotse接入Gaia云的一些经验与看法，抛砖引玉，期待更多的系统加入进来，获得实实在在的好处。

###背景介绍

Lhotse是一个大数据任务调度系统，从架构上看是典型的Master-Agent分布式架构，如下图所示，作为调度核心的Base统筹分配任务，交由对应类型的Runner执行：

![Lhotse架构图](https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/LhotseArch.png)

到目前为止，Lhotse线上支持68种Runner，分别对应68种不同的任务类型，集群总机器数将近50台。面对这种复杂多样的系统环境，Lhotse接入Gaia云的核心诉求是*自动化*，通过自动化来提高运维效率。这体现在如下两个方面：

1. 机器操作取代人工操作，将复杂枯燥的工作（比如资源分配、程序部署）交给云平台处理。

2. 通用实现取代重复实现，云平台将一些通用性的基础工作（比如进程监控、自动拉起）抽象成标准化服务，不必重复造轮子。

下面我们将分别从部署、调度、容错、扩缩容及服务发现四个纬度来讨论Lhotse如何在Gaia云中获得自动化。

###自动部署

Lhotse部署遇到的最大难点在于，每个Runner类型所依赖的运行环境是有差异的。例如，Pig Runner需要机器上预装PigClient、Hadoop，而Compute Runner需要预装PLC。极端的情况是，68种Runner对应68种不同的运行环境依赖，且要在50台机器里交叉部署，还要保证各个运行环境完整一致，相互不受影响，这对运维来说是极大的挑战。

所幸的是，Docker正是为了解决此类问题而生的。我们将每个Runner及其运行环境build成一个Docker镜像，push到Gaia云的Registry，通过Registry自动分发到集群的任意机器上。标准化的镜像分发将保证Runner的运行环境跑到哪里都是一致的，且不受其他环境的影响。

![Lhotse镜像分发](https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/LhotseRunnerImages.png)

###自动调度

假定我们已经将所有的Lhotse镜像都提交到了Registry，

    - 机器资源管理麻烦，提高资源利用率，资源多租户共享
    
###自动容错
    - 异常监测与迁移重试: 对比之前的手工脚本

###自动扩缩容
    - 根据规则动态变化

###自动服务发现

###总结
