# Ranger

标签（空格分隔）： 未分类

---

## 前言

2016年，Hadoop迎来了自己十周岁生日。过去的十年，Hadoop雄霸武林盟主之位，号令天下，引领大数据技术生态不断发展壮大，一时间百家争鸣，百花齐放。然而，兄弟多了不好管，为了抢占企业级市场，各家都迭代出自己的一套访问控制体系，不管是老牌系统(比如HDFS、HBase)，还是生态新贵(比如Kafka、Alluxio)，ACL(Access Control List)支持都是Roadmap里被关注最高的issue之一。

历史证明跳出混沌状态的最好方式就是——出台标准。于是，Hadoop两大厂Cloudera和Hortonworks先后发起标准化运动，分别开源了Sentry和Ranger，在centralized访问控制领域展开新一轮的角逐。

Ranger在0.4版本的时候被Hortonworks加入到其Hadoop发行版HDP里，目前作为Apache incubator项目，最新版本是0.6。它主要提供如下功能:

 - 基于策略(Policy-based)的访问权限模型
 
 - 通用的策略同步与决策逻辑，方便控制插件的扩展接入
 
 - 内置常见系统(如HDFS、YARN、HBase)的控制插件，且可扩展

 - 内置基于LDAP、文件的用户同步机制，且可扩展
 
 - 统一的管理界面，包括策略管理、审计查看、插件管理等
 
本文旨在从技术层面剖析Ranger如何实现Centralized访问控制，将从权限模型、总体架构、插件实现三个方面来展开。

## 权限模型

访问权限无非是定义了"__用户-资源-权限__"这三者间的关系，Ranger基于策略来抽象这种关系，进而延伸出自己的权限模型。为了简化模型，便于理解，我用以下表达式来描述它：

    Policy = Service + List<Resource> + AllowACL + DenyACL
    AllowACL = List<AccessItem> allow + List<AccssItem> allowException
    DenyACL = List<AccessItem> deny + List<AccssItem> denyException
    AccessItem = List<User/Group> + List<AccessType>


接下来从"用户-资源-权限"的角度来详解上述表达：

 - 用户：由User或Group来表达；User代表访问资源的用户，Group代表用户所属的用户组。
 
 - 资源：由(Service, Resource)二元组来表达；一条Policy唯一对应一个Service，但可以对应多个Resource。

 - 权限：由(AllowACL, DenyACL)二元组来表达，两者都包含两组AccessItem。而AccessItem则描述一组用户与一组访问之间的关系——在AllowACL中表示允许执行，而DenyACL中表示拒绝执行。

下图列出了几种常见系统的模型实体枚举值：
![Ranger模型实体枚举值][1]

关于权限这个部分，还有一点没有解释清楚：为什么AllowACL和DenyACL需要分别对应两组AccessItem？这是由具体使用场景引出的设计：

以AllowACL为例，假定我们要将资源授权给一个用户组G1，但是用户组里某个用户U1除外，这时只要增加一条包含G1的AccessItem到AllowACL_allow，同时增加一条包含U1的AccessItem到AllowACL_allowException即可。类似的原因可反推到DenyACL。

既然现在一条Policy有(allow, allowException, deny, denyException)这么四组AccessItem，那么判断用户最终权限的决策过程是怎样的？

总体来说，这四组AccessItem的作用优先级由高到低依次是：

    denyException > deny > allowException > allow


决策树可以用以下流程图来描述：

```flow
st=>start: 用户请求资源
io=>inputoutput: verification
op=>operation: 找到所有资源
               匹配的Policy
cond10=>condition: 有deny
AccessItem匹配
cond11=>condition: 有denyException
AccessItem匹配
cond20=>condition: 有allow
AccessItem匹配
cond21=>condition: 有allowException
AccessItem匹配
                  
sub=>subroutine: Your Subroutine
e10=>end: 拒绝访问
e20=>end: 允许访问
e30=>end: 拒绝访问
/向后传递

st->op->cond10
cond10(yes)->cond11
cond11(no)->e10
cond10(no)->cond20
cond11(yes)->cond20
cond20(yes)->cond21
cond21(no)->e20
cond20(no)->e30
cond21(yes)->e30
```

## 总体架构

下图列出了几种常见系统的模型实体枚举值：
![Ranger总体架构][2]

## 插件实现



[1]: https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/RangerConceptTable.png
[2]: https://raw.githubusercontent.com/tragicjun/tragicjun.github.io/master/images/RangerArchitecture.png
