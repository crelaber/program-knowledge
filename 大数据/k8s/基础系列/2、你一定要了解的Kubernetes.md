## 前言

随着容器技术的发展，`Docker`近几年突然崛起，变得炙手可热，已经成为容器技术的事实标准。然而想要在生成环境中成功部署和操作容器的关键是容器编排技术，市场上有各种各样的容器编排工具（如`Docker`原生的`Swarm`），其中谷歌公司开发的`Kubernetes`得到开源社群的全力支援，IBM、惠普、微软、RedHat等业界巨头纷纷加入，`Kubernetes`已经成为`GitHub`上的明星开源解决方案

## 简介

`Kubernetes`这个名字源于希腊语，是舵手的意思，所以它的Logo既像一张渔网，又像一个罗盘。有意思的是`Docker`的Logo为驮着集装箱在大海上遨游的鲸鱼，`Kubernetes`与`Docker`的关系可见一斑。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5ZBy18OgpbfZ8uyulmak66hrhaIK1tPGQ6z9lwswSlCD30lsQun3icjDRjNm3j3ge4W38UwAicMqIA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)Docker和Kubernetes

`Kubernetes`（也常称`K8s`，用8代替8个字元“ubernete”而成的缩写。）是一个全新的基于容器技术的分散式架构方案，它源自`Google`内部大规模集群管理系统——`Borg`，也是CNCF（Cloud Native Computing Foundation，今属Linux基金会）最重要的解决方案之一，旨在让部署容器化的应用简单并且高效。

`Kubernetes`具备完善的集群管理能力，包括多层次的安全防护和准入机制、多租户应用支撑能力、透明的服务注册和服务发现机制、内建负载均衡器、故障发现和自我修复能力、服务滚动升级和线上扩容、可扩充套件的资源自动调度机制、多粒度的资源配额管理能力。还提供完善的管理工具，涵盖开发、部署测试、运维监控等各个环节。

Kubernetes在全球范围的搜索指数

`Kubernetes`于2015年7月22日迭代到`v1.0`并正式对外公布，无论是英文还是中文社群都非常活跃，全球开源解决方案领导者`RedHat`公司已经将自己`Paas`产品`OpenShift V3`的底层技术换成了`Kubernetes`与`Docker`。`Kubernetes`俨然已经成为全新容器生态的领导者。

## 架构

`Kubernetes`使用`Go`语言开发，集群采用`Master/Node`（最初称为`Minion`，后改名`Node`） 的结构，`Master`（主节点）控制整个集群，`Node`（从节点）为集群提供计算能力。使用者可以通过命令列或者`Web`页面的方式来操作集群。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5ZBy18OgpbfZ8uyulmak668ALYnQ5SBRM855EDwMbKzpqMh5oicohxPo0R5OoyQERUmLtD9ibSPF4Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)Kubernetes架构

**Master**是`Kubernetes`集群的大脑，负责公开集群的`API`，部署和管理整个集群。集群至少有一个`Master`节点，如果在生产环境中要达到高可用，还需要配置`Master`集群。

`Master`主要包含`API Server`、`Scheduler`、`Controller`三个组成部分，需要`etcd`来储存整个集群的状态。

- **etcd：** 由`CoreOS`开发，是一个高可用、强一致性的服务发现储存仓库，为`Kubernetes`集群提供储存服务，类似于`zookeper`。
- **API Server：** `kubernetes`最重要的核心元件之一，提供资源操作的唯一入口（其他模组通过`API Server`查询或修改资料，只有`API Server`才能直接操作`etcd`），并提供认证、授权、访问控制、API注册和发现等机制。
- **Scheduler：** 负责资源的调度，按照预定的调度策略将`Pod`（k8s中调度的基本单位）调度到相应的`Node`上。
- **Controller：** 通过`API Server`来监控整个集群的状态，并确保集群处于预期的工作状态，比如故障检测、自动扩充套件、滚动更新等。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5ZBy18OgpbfZ8uyulmak66Lficm17WqFiakPDGCSeaLTmb01qyecqUo4AFcLxokK1PKVLg9Vc2mpAA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)Kubernetes主节点

`Node`是`Kubernetes`集群的工作节点，可以是物理机也可以是虚拟机器。`Node`需要包含容器、`kubelet`、`kube-proxy`等元件。`Fluentd`用来提供日志收集功能（可选）。

- **kubelet：** 维护容器的生命周期，同时也负责`Volume`（CVI）和网络（CNI）的管理。每个节点上都会执行一个`kubelet`服务程序，接收并执行`Master`发来的指令，管理`Pod`及`Pod`中的容器。每个`kubelet`程序会在`API Server`上注册节点自身的信息，定期向`Master`节点汇报自身节点的资源使用情况，并通过`cAdvisor`监控节点和容器的资源。
- **kube-proxy：** 为`Service`提供集群内部的服务发现和负载均衡，监听`API Server`中`service`和`endpoint`的变化情况，并通过`iptables`等方式来为服务配置负载均衡。
- **Docker：** 每台`Node`上需要安装`Docker`来执行镜像，但是`Docker`不是唯一选择，`Kubernetes`支援多种容器，比如`CoreOS`公司的`Rkt`容器（之前称为`Rocket`，现更名为`Rkt`）。

Kubernetes从节点

## 资源物件

API物件是`Kubernetes`集群中的管理操作单元。集群中的众多技术概念分别对应着API物件，每个API物件都有3大类属性：

- `metadata`（元资料）：用来标识API物件，包含`namespace`、`name`、`uid`等。
- `spec`（规范）：描述使用者期望达到的理想状态，所有的操作都是宣告式（Declarative）的而不是命令式（Imperative），在分布式系统中的好处是稳定，不怕丢操作或执行多次。比如设定期望3个执行`Nginx`的`Pod`，执行多次也还是一个结果，而给副本数加1的操作就不是宣告式的，执行多次结果就错了。
- `status`（状态）：描述系统当前实际达到的状态，比如期望3个`Pod`，现在实际建立好了2个。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5ZBy18OgpbfZ8uyulmak66QEAzQefxaxLfDpWgfBS4Qo00OpnZS8znanECoBvibL5WL2Kbe7wLSLQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)资源组件

在`Kubernetes`众多的API物件中，`Pod`是最重要的也是最基础的，是`kubernetes`中可以建立的最小部署单元。`Pod`就像是豌豆荚一样，它可以由一个或者多个容器组成，这些容器共享储存、网路和配置项。

目前`Kubernetes`中的业务型别可以分为长期伺服型（long-running）、批处理型（batch）、节点后台支撑型（node-daemon）和有状态应用型（stateful application）这四种类型，而这四种类型的业务又可以由不同类别的`Pod`控制器来完成，分别为：**Deployment**、**Job**、**DaemonSet**和**StatefulSet**。

- **Deployment:** 复制控制器（Replication Controller，RC）是集群中最早的保证`Pod`高可用的API物件，副本集（Replica Set，RS）是它的升级，能支援更多种类的匹配模式。部署(Deployment)又是比`RS`应用模式更广的API物件，以`Kubernetes`的发展方向，未来对所有长期伺服型的的业务的管理，都会通过Deployment来管理。
- **Service：** Deployment保证了`Pod`的数量，但是没有解决如何访问`Pod`的问题，一个`Pod`只是一个执行服务的选项，随时可能在一个节点上停止，在另一个节点以一个新的`IP`启动一个新的`Pod` ，因此不能以确定的`IP`和端口号提供服务。要稳定地提供服务需要服务发现和负载均衡能力，`Service`可以稳定为使用者提供服务。
- **Job：** 用来控制批处理型任务，`Job`管理的`Pod`根据使用者的设定把任务成功完成就自动退出了。
- **DaemonSet：** 后台支撑型服务的核心关注点在集群中的`Node`，要保证每个`Node`上都有一个此类`Pod`在运行。比如用来收集日志的`Pod`。
- **StatefulSet：** 不同于`RC`和`RS`，`StatefulSet`主要提供有状态的服务，`StatefulSet`中`Pod`的名字都是事先确定的，不能更改，每个`Pod`挂载自己独立的储存，如果一个`Pod`出现故障，从其他节点启动一个同样名字的`Pod`，要挂载上原来`Pod`的储存继续以它的状态提供服务。比如数据库服务`MySQL`，我们不希望一个`Pod`故障后，`MySQL`中的数据即丢失。

## 小结

事实上，`Kubernetes`的内部构件不止这些，要熟练的操作`Kubernetes`并了解其原理还有很长的路要走。本文只是从宏观视角上的介绍了`Kubernetes`，对于想要了解`Kubernetes`的朋友是个不错的开始。