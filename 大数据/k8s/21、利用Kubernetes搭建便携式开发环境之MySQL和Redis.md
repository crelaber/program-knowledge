之前给大家介绍过， 我自己用的开发环境都是容器化的，只不过前两年不会用K8，大部分都是用的 Docker 或者 Docker-Compose。随着这一年多对 K8 的学习和工作中的使用，一直有想法用K8 做一套便携式开发环境，以后换电脑就不用再愁数据库、缓存、队列这些基础软件的安装了。正好下个月我就能换新的办公电脑啦，也不能拿『能用就行』的理由再拖延下去了。

如果问你 “ 平时开发程序依赖最多的基础软件时啥？”，我猜大部分人会回答：“MySQL 和 Redis”，毕竟万物皆是增删改查，整天做CURD的我们怎么能离开它们呢。

## 准备工作

### 工具选择

既然是要在本地 Kubernetes 上搭建开发环境，那电脑上得先有 Kubernetes 集群才行。目前可以在本地运行 Kubernetes 集群的工具有：Minikube 、Kind 和 K3d ，我们的MySQL和Redis都是靠先编写资源定义YAML文件，再通过 kubectl 交给Kubernetes 集群执行的，所以这三种工具用哪种都行。

我自己在本地使用的是Minikube，这是 Kubernetes 官方提供的工具，说实话运行起来后电脑有点卡，Minikube的安装步骤可以参考我以前写的文章「[Minikube-运行在笔记本电脑上的Kubernetes集群](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485235&idx=1&sn=0cdd31c25b13790b336e0a8222f00b64&scene=21#wechat_redirect)」。另外两种 Kind 和 K3d 则是轻量级集群，支持多节点部署。其中我比较推荐K3d，尤其是使用 M1芯片MacBook的同学，现在暂时只能使用K3d安装Kubernetes集群。

> K3d 是使用 docker 容器在本地运行 k3s 集群，k3s 是由 Rancher Lab 开源的轻量级 Kubernetes。k3d 完美继承了 k3s 的简单、快速和占用资源少的优势，镜像大小只有 100 多 M，启动速度快，支持多节点集群。虽然 k3s 对 Kubernetes 进行了轻量化的裁剪，但是提供了完整了功能，像 Istio 这样复杂的云原生应用都可以在 k3s 上顺利运行。
>
> K3d 除了启动速度快和占用资源少以外，在边缘计算和嵌入式领域也有着不俗的表现。因为 k3s 本身应用场景主要在边缘侧，所以支持的设备和架构很多，如：ARM64 和 ARMv7 处理器。很多老旧 PC 和树莓派这样的设备都可以拿来做成 k3s 集群，为本地研发测试燃尽最后的生命。

### 预备知识点

说完了安装工具的选择，我们再来说一下在Kubernetes上从零搭建开发环境需要提前做哪些知识储备，如果你已经对Kubernetes这些基础概念已经有所了解可以直接跳过去看实操环节了，如果还比较生疏的话，我建议大家先看看下面这几篇文章，这些都是我们搭建开发环境时需要用到的。

[Kubernetes Pod入门指南](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485464&idx=1&sn=00ca443bbcd4b2996efdede396b6c667&chksm=fa80d98fcdf7509944d63f618264e36cd8082a77e23aa36428a3d57a2f4189bcce4e52986967&token=2033333242&lang=zh_CN&scene=21#wechat_redirect)

[应用编排利器之Deployment](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485643&idx=1&sn=6460bf2e170e4b2e8ebb2882bfe7c60f&scene=21#wechat_redirect)

[学练结合，快速掌握Kubernetes Service](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486082&idx=1&sn=42a9bc8fcfc9da09445e9e2f4cf2fb96&chksm=fa80db15cdf752039494992f71a3bc488cf386841bd1aaaa44115f5e7f155ba55ce468ec89ee&token=2033333242&lang=zh_CN&scene=21#wechat_redirect)

[用面向对象的方式管理配置文件](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486416&idx=1&sn=20d568f93d0f39e0f3c7ef3ce42ac1d8&scene=21#wechat_redirect)

[深入理解StatefulSet，用Kubernetes编排有状态应用](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486737&idx=1&sn=e7d0689fa74b108bae734515837c68e1&chksm=fa80dc86cdf755909f2f29ee8cb9dce930b95a837cb045be5c9700a9000743c9cb7f8eed6731&token=1423297622&lang=zh_CN&scene=21#wechat_redirect)

## 安装MySQL

在正式开始在Kubernetes上安装MySQL和Redis前我先说明下安装这两个基础软件服务的思路。

1. 首先因为是用作开发环境，我们就不追求高可用了，尽量精简。文章后面我会给出安装主从和集群式数据库的一些教程链接，供大家参考。

2. 安装MySQL和Redis的思路是一样的，每个服务都由下面几个主要的部分构成：

   ① 一个单副本 Pod 作为运行MySQL或者Redis 的载体。

   ② 一个调度Pod用的Deployment控制器，因为服务里只包含一个Pod，不需要维持构建的顺序，所以不用使用StatefulSet作为Pod的控制器。

   ③一个ConfigMap对象，包含了MySQL或者Redis配置文件里需要的配置项，在创建Pod时会作为配置文件挂载到应用所在的容器中。

   ④一个 Service 对象，将应用 Pod 作为自己的后端端点，以始终保持不变的NodeId:NodePort 方式向外暴露服务。

下面这张图很好的解释了这四部分的协作关系。

MySQL on Kubernetes

解释清楚我们在Kubernetes上搭建MySQL和Redis开发环境的思路后，下面就可以进入实操环节啦，我为大家准备了可以直接拿来使用的YAML资源定义文件。

### 创建MySQL配置

我们先来创建一个名为 mysql-db-config 的ConfigMap，稍后会把这些配置作为 my.cnf 配置文件挂载到MySQL应用Pod的容器里。

```yaml
### 文件名 mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-db-config
  namespace: default
  labels:
    app: mysql-db-config
data:
  my.cnf: |
    [client]
    default-character-set=utf8mb4
    [mysql]
    default-character-set=utf8mb4
    [mysqld]
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci
    init_connect='SET NAMES utf8mb4'
    skip-character-set-client-handshake = true
    max_connections=2000
    secure_file_priv=/var/lib/mysql
    datadir=/var/lib/mysql
    bind-address=0.0.0.0
    symbolic-links=0
```

假定MySQL相关的YAML定义文件都放在 mysql-singleton 这个目录下，通过 kubectl 把这个ConfigMap 提交给Kubernetes 进行创建即可：

```
kubectl apply -f mysql-singleton/mysql-configmap.yaml
```

### 创建MySQL容器和Service

有了MySQL配置相关的 ConfigMap后，我们就能在创建运行MySQL的容器时，把他作为配置文件挂载到容器中：

```yaml
### 文件名 deployment-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: NodePort
  ports:
    - port: 3306
      nodePort: 30306
      targetPort: mysql
  selector:
    app: mysql
---
apiVersion:
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - image: mysql:5.7
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: root
            - name: MYSQL_USER
              value: user
            - name: MYSQL_PASSWORD
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
            - name: mysql-config
              mountPath: /etc/mysql/conf.d/my.cnf
              subPath: my.cnf
      volumes:
        - name: mysql-persistent-storage
          emptyDir: {}
        - name: mysql-config
          configMap:
            name: mysql-db-config
            items:
              - key: my.cnf
                path: my.cnf
```

同样的，使用kubectl 把 YAML 提交给Kubernetes后，等资源创建完毕我们的开发环境MySQL就算搭建好了

```shell
kubectl apply -f mysql-singleton/deployment-service.yaml
```

通过上面的YAML文件中，有三点需要详细说明一下：

1. 使用 mysql-db-config 这个ConfigMap 中my.cnf这个配置项以my.cnf文件名挂载到容器中去，但是因为挂载进去后会覆盖容器中conf.d 目录中的内容。通过volumeMounts.subPath可以支持选定ConfigMap中的Key-Value挂载到容器中而不覆盖容器中原有的文件。
2. Service 用 30306 端口向集群外暴露了MySQL服务，客户端从电脑上使用NodeIP:NodePort即可连接到这里创建的数据库，如果用的是Minikube创建的Kubernetes集群， 可以通过minikube ip 命令查到NodeIP。
3. emptyDir 类型的数据卷的生命周期与Pod同步，这里的同步指的是Pod被kubectl delete 主动删除时才会销毁对应的数据卷，如果是Pod自己崩溃，或者是集群Shotdown，等恢复后重建出来的Pod仍然会使用之前的数据卷，不会造成数据丢失。

在Kubernetes上创建完MySQL后，我们可以通过任意客户端或者mysql命令行，连接MySQL服务。

```
mysql -uroot -proot -h {minikube-ip} -P 30306
```

## 安装Redis

聊清楚了怎么用Kubernetes创建单节点的MySQL后，对于创建单例的Redis相信大家对大致流程也就比较清楚了，唯一少的就是定义Redis服务的这些YAML文件了。我已经帮你们踩好坑了，下面这些YAML都是我在线下调试过一段时间的，并且也能正确完成Redis数据的持久化。

由于声明 Redis 配置的 ConfigMap 篇幅太长，为了不影响文章的阅读我就把这个安装Redis需要的YAML文件都放在GitHub上了，可以点击阅读原文或者通过链接：https://github.com/kevinyan815/LearningKubernetes 访问下载。

在这个仓库里我给出了MySQL和Redis的详细安装步骤，以及各种资源的YAML定义文件，包括之前安装ETCD集群的教程也整合到了这个仓库里。

> 安装步骤详解，参考 [用Kubernetes搭建ETCD集群和WebUI](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247487400&idx=1&sn=bd6b3eca835cbe65fb70a4dfe9326037&scene=21#wechat_redirect)
>
> 关于安装过程中遇到的问题可以在留言里跟我交流，大家还想看其他基础软件在Kubernetes上的安装教程的话也可以告诉我。

## 总结

这篇文章里整理了在Kubernetes上安装MySQL和Redis这两款我们常用的基础软件的操作步骤，由于目的是在本地开发环境用，所以力求资源定义尽量简单，能做到数据可持久化就行了，高可用不再这里讨论。如果你对在Kubernetes上创建MySQL集群有兴趣，可以参考我下面给出的链接。

**MySQL Operator FOR Kubernetnes**[1]

### 资料引用

[1]MySQL Operator FOR Kubernetnes: :https://medium.com/oracledevs/getting-started-with-the-mysql-operator-for-kubernetes-8df48591f592