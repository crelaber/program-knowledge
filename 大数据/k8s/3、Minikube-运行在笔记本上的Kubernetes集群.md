## 前言

`Minikube`是一个可以在本地电脑上运行`Kubernetes`的工具。`Minikube`会在笔记本电脑中的虚拟机上运行一个单节点的`Kubernetes`集群，让用户能对`Kubernetes`进行体验或者在之上进行`Kubernetes`的日常开发。

`Windows`，`MacOS`和`Linux`系统上都可以安装`Minikube`，不过在安装前需要确认系统的版本已经支持虚拟化（一般只要不是太老的系统版本都支持虚拟化）

## kubectl

在电脑上安装`Minikubne`前需要先安装`kubectl`，它是`Kubernetes`的命令行工具，可以使用`kubectl`部署应用程序，检查和管理集群资源以及查看日志。

### 安装kubectl

文章里我们演示的安装步骤都是macOS上的，如果是Linux和Windows系统只需要下载相应系统的二进制文件就行，我会在文章后边贴上官方的安装指南。

首先下载最新的稳定版本的`kubectl`二进制文件。

```
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl"
```

为`kubectl`授予可执行权限，然后将可执行文件放到系统的`PATH`目录中

```
chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl
```

## 安装MiniKube

如果你的`macOS`上没有安装虚拟机监控程序的话在第一次启动`minikube`的时候会自动选择安装`HyperKit`作为虚拟机驱动，如果是以前电脑上有安装过`VirtualBox`那么可以在`Minikube`启动时加上`--vm-driver=virtualbox`来选择虚拟机驱动。

安装`minikube`的过程跟`kubectl`的过程差不多，也是下载`minikube`的二进制文件，赋予可执行权限后将其放入系统环境变量`PATH`对应的目录中。

不过由于大家都知道的网络访问原因，很多朋友无法直接使用`Kubernetes`官方提供的`minikube`进行实验，所以这里选择使用阿里云提供的`minikube`版本

```
curl -Lo minikube https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.11.0/minikube-darwin-amd64 \ 
&& chmod +x minikube \ 
&& sudo mv minikube /usr/local/bin/
```

如果是Linux和Window系统，安装流程类似只是软件的版本不同，具体可以参照官方文档里给的MiniKube的安装指南：

https://kubernetes.io/docs/tasks/tools/install-minikube

## 运行Minikube

启动`minikube`的方法非常简单，只要使用下面的命令

```shell
minikube start  --image-mirror-country='cn' --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'    
```

启动minikube

在最新的`Minikube`中，已经提供了配置化的方式，可以帮助大家利用阿里云的镜像地址来获取所需的`Docker`镜像和配置。

## 测试Minikube

下面我们通过`minikube status`命令查看一下它的运行状态测试我们安装的`minikube`。

```shell
➜  minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

通过`kubectl`查看集群的一些信息。

```shell
➜  kubectl get pods -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-67c766df46-59rtb           1/1     Running   0          17m
kube-system   coredns-67c766df46-jxmvf           1/1     Running   0          17m
kube-system   etcd-minikube                      1/1     Running   0          16m
kube-system   kube-addon-manager-minikube        1/1     Running   0          16m
kube-system   kube-apiserver-minikube            1/1     Running   0          16m
kube-system   kube-controller-manager-minikube   1/1     Running   0          17m
kube-system   kube-proxy-ljppw                   1/1     Running   0          17m
kube-system   kube-scheduler-minikube            1/1     Running   0          16m
kube-system   storage-provisioner                1/1     Running   0          17m

➜   kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   18m   v1.18.3


➜   kubectl get namespaces
NAME              STATUS   AGE
default           Active   18m
kube-node-lease   Active   18m
kube-public       Active   18m
kube-system       Active   18m
```

## 接下来

安装完`Minikube`后我们的电脑上就有了`Kubernetes`的基础运行环境，通过最近几篇关于`Kubernetes`的文章相信大家都已经对`Kubernetes`有了初步的认识，不过都是概念性的知识，到现在来说`Kubernetes`还是一个比较抽象的东西，说实话这么学下去的话我会觉得太枯燥，需要一些实操性的练习给自己一些正反馈才能坚持下去。所以我准备尝试做一个简单的用`Go`语言写的应用程序的`Docker`镜像，把它放到本地电脑上的`Kubernetes`集群（`Minikuebe`）上运行。具体的步骤会在下周推送的文章里告诉大家，祝大家假期愉快！