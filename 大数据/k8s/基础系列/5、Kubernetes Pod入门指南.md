`Pod`是`Kubernetes`项目里定义的最小可调度单元，是`Kubernetes`对应用程序的抽象。在这篇文章里我将会介绍`Kubernetes`里`Pod`的基本概念，使用方式，生命周期以及如何使用`Pod`部署应用。读这篇文章的朋友我会默认你已经了解`Kubernete`是用来解决什么问题的，以及电脑上已经安装了`Minikube`这个能试验`Kubernetes`功能的工具。如果尚未做好这些准备工作，推荐先去看下面的两篇文章做好准备工作后再来学习这里的内容。

**[你一定要了解的Kubernetes](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485178&idx=1&sn=eab6e986d85af18fe0129bf52d4a6991&chksm=fa80d76dcdf75e7b5a0f29275c1e895b9a9664dc65b66500c890ec76b39b5122108cfa10c3c0&token=1122599271&lang=zh_CN&scene=21#wechat_redirect)**

**[运行在笔记本上的Kubernetes集群](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485235&idx=1&sn=0cdd31c25b13790b336e0a8222f00b64&chksm=fa80d6a4cdf75fb27a64c7f06f9a33ed4e644624d3fcf73d791274d03b071467ebb4e292ac95&token=1122599271&lang=zh_CN&scene=21#wechat_redirect)**

## 什么是Pod

在`Kubernetes`的`API`对象模型中，`Pod`是最小的`API`对象，换一个专业点的的说法可以这样描述：`Pod`，是 `Kubernetes` 的原子调度单位。在集群中，`Pod`表示正在运行的应用进程。`Pod`的内部可以有一个或多个容器，同属一个`Pod`的容器将会共享：

- 网络资源
- 相同的IP
- 存储
- 应用到Pod上的自定义配置

可以看到`Pod`是`Kubernetes`定义出来的一个逻辑概念，可以用另外一种方式来理解`Pod`：一种特定于应用程序的“逻辑主机”，其中包含一个或多个紧密协作的容器。例如，假设我们在Pod中有一个应用程序容器和一个日志记录容器。日志记录容器的唯一工作是从应用程序容器中提取日志。将两个容器放置同一个`Pod`里可消除额外的通信时间，因为它们位于同一个"主机"，因此所有内容都是本地的并且它们共享所有资源，就跟在同一台物理服务器上执行这些操作一样。

此外也不是所有有“关联”的容器都属于同一个`Pod`。比如，应用容器和数据库虽然会发生访问关系，但并没有必要、也不应该部署在同一台机器上，它们更适合做成两个`Pod`。

## Pod的模型

根据`Pod`里的容器数量可以将`Pod`分为两种类型：

- 单容器模型。由于`Pod`是`Kubernetes`可识别的最小对象，`Kubernetes`管理调度`Pod`而不是直接管理容器，所以即使只有一个容器也需要封装到`Pod`里。
- 多容器模型。在这个模型中，`Pod`可以容纳多个紧密关联的容器以共享`Pod`里的资源。这些容器作为单一的，凝聚在一起的服务单元工作。

每个`Pod`运行应用程序的单个实例。如果需要水平扩展/缩放应用程序（例如运行多个副本），则可以为每个实例使用一个`Pod`。这与在单个`Pod`中运行同一应用程序的多个容器不同。

还需要提的一点是，`Pod`本身不具有调度功能。如果所在的节点发生故障或者你要维护节点，则`Pod`是不会自动调度到其他节点了。`Kubernetes`用一系列控制器来解决`Pod`的调度问题，`Deployment`就是最基础的控制器。通常我们都是在定义的控制器的配置里通过`PodTemplate`定义要控制的`Pod`，让控制器和所管控的`Pod`一起被创建出来（这部分内容后面单独写文章讨论）。

## Pod生命周期的阶段

一个`Pod`的状态会告诉我们它当前正处于生命周期的哪个阶段，`Pod`的生命周期有5个阶段：

- Pending：等待状态表明至少有一个`Pod`内的容器尚未创建。
- Running：所有容器已经创建完成，并且`Pod`已经被调度到了一个Node上。此时`Pod`内的容器正在运行，或者正在启动或重新启动。
- Succeeded：`Pod`中的所有容器均已成功终止，并且不会重新启动。
- Faild: 所有容器均已终止，至少有一个容器发生了故障。失败的容器以非零状态退出。
- Unknown：无法获得`Pod`的状态。

## 在实践中使用Pod

我们已经讨论了`Pod`在理论上的含义，现在让我们看看它在实际中长什么样。我们将首先浏览一个简单的`Pod`定义`YAML`文件，然后部署一个示例应用程序来展示如何使用它。

### Pod的YAML文件

`Kubernetes`里所有的`API`对象都由四部分组成：

- apiVersion -- 当前使用的`Kubernetes`的API版本。
- kind -- 你想创建的对象的种类。
- metadata -- 元数据，用于唯一表示当前的对象，比如name、namespace等。
- spec -- 我们的`Pod`的指定配置，例如镜像名称，容器名称，数据卷等。

`apiVersion`，`kind`和`metadata`是必填字段，适用于所有`Kubernetes`对象，而不仅仅是`pod`。`spec`里指定的内容（`spec`也是必需字段）会因对象而异。下面的示例显示了Pod的YAML文件大概长什么样子。

```yaml
apiVersion: "api version"             
kind: "object to create"                 
metadata:                   
  name: "Pod name"
  labels:
    app: "label value"
spec:                       
  containers:
  - name: "container name"
    image: "image to use for container"
```

关于YAML的语法可以参考前面的文章：**[YAML，另一种标记语言？不止是标记语言！](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485412&idx=1&sn=de72afc9da20cd28afbb73696f4f0f70&chksm=fa80d673cdf75f659bc5a213774308444a8ac45f971ecbffd96855bbf015914c4d3261b9f844&token=1122599271&lang=zh_CN&scene=21#wechat_redirect)**

理解了`Pod`配置文件的模板后，接下来我们看看如何使用配置文件创建上面说的两种模型的`Pod`。

### 单容器Pod

下面的`pod-1.yaml`是个单容器`Pod`的清单文件。它会运行一个`Nginx`容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: first-pod
  labels:
    app: myapp
spec:
  containers:
  - name: my-first-pod
    image: nginx
```

接下来，我们通过运行`Kubectl create -f pod-1.yaml`将清单文件部署到本地的`Kubernetes`集群中。然后，我们运行`kubectl get pods`以确认我们的`Pod`运行正常。

```shell
kubectl get pods
NAME                                          READY     STATUS    RESTARTS   AGE
first-pod                                      1/1       Running   0          45s
```

可以到`Pod`的`Nginx`容器里执行以下`service nginx status`命令确保`Nginx`在正常运行。

```shell
kubectl exec first-pod -- service nginx status
nginx is running.
```

这会在`Pod`里执行`service nginx status`指令，类似`docker exec`命令。

现在，我们通过运行`kubectl delete pod first-pod`删除刚才创建的`Pod`。

```shell
kubectl delete pod first-pod
pod "firstpod" deleted
```

### 多容器Pod

下面我们将部署一个更复杂的`Pod`：一个拥有两个容器的`Pod`，这些容器相互协作作为一个实体工作。其中一个容器每10秒将当前日期写入一个文件，而另一个`Nginx`容器则为我们展示这些日志。这个`Pod`的`YAML`如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod # pod的名称
spec:
  volumes:
  - name: shared-date-logs  # 为Pod里的容器创建一个共享数据卷
    emptyDir: {}
  containers:
  - name: container-writing-dates # 第一个容器的名称
    image: alpine # 容器的镜像
    command: ["/bin/sh"]
    args: ["-c", "while true; do date >> /var/log/output.txt; sleep 10;done"] # 每10秒写入当前时间
    volumeMounts:
    - name: shared-date-logs
      mountPath: /var/log # 将数据卷挂在到容器的/var/log目录
  - name: container-serving-dates # 第二个容器的名字
    image: nginx:1.7.9 # 容器的镜像
    ports:
      - containerPort: 80 # 定义容器提供服务的端口
    volumeMounts:
    - name: shared-date-logs
      mountPath: /usr/share/nginx/html # 将数据卷挂载到容器的/usr/share/nginx/html 
```

上面通过`volumes`指令定义了Pod内的数据卷

```yaml
  volumes:
  - name: shared-date-logs  # 为Pod里的容器创建一个数据卷
    emptyDir: {}
```

第一个容器将数据卷挂载到了`/var/log/`每隔10秒往`output.txt`文件里写入时间，而第二个容器通过将数据卷挂载到`/usr/share/nginx/html`伺服了这个日志文件。

执行`kubectl create -f pod-2.yaml`创建这个多容器`Pod`：

```shell
kubectl create -f pod-2.yaml
pod "multi-container-pod" created
```

然后确保`Pod`已经正确部署：

```shell
kubectl get pods
NAME                                          READY     STATUS    RESTARTS   AGE
multi-container-pod                           2/2       Running   0          1m
```

通过运行`kubectl describe pod podName`，查看Pod的详细信息，里面会包含两个容器的信息。（下面的内容只截取了容器相关的信息）

```yaml
Containers:
  container-writing-dates:
    Container ID:  docker://e5274fb901cf276ed5d94b...
    Image:         alpine
    Image ID:      docker-pullable://alpine@sha256:621c2f39...
    Port:          
    Host Port:     
    Command:
      /bin/sh
    Args:
      -c
      while true; do date >> /var/log/output.txt; sleep 10;done
    State:          Running
      Started:      Sat, 1 Aug 2020 11:31:44 +0800
    Ready:          True
    Restart Count:  0
    Environment:    
    Mounts:
      /var/log from shared-date-logs (rw)
      /var/run/secrets/Kubernetes.io/serviceaccount from default-token-8dl5j (ro)
    container-serving-dates:
    Container ID: docker://f9c85f3fe3...
    Image:          nginx:1.7.9
    Image ID:       docker-pullable://nginx@sha256:e3456c851...
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 1 Aug 2020 11:31:44 +0800
    Ready:          True
    Restart Count:  0
    Environment:    
    Mounts:
      /usr/share/nginx/html from shared-date-logs (rw)
      /var/run/secrets/Kubernetes.io/serviceaccount from default-token-8dl5j (ro)
```

两个容器都在运行，下面我们将进到`Pod`里确保两个容器都在执行分配的作业。

通过运行`kubectl exec -it multi-container-pod -c container-serving-dates -- bash`连接到`Nginx`容器里。

在容器内运行`curl'http://localhost:80/output.txt'`，它应该返回时间日志文件的内容给我们。（如果容器中未安装curl，请先运行`apt-get update && apt-get install curl`，然后再次运行`curl'http://localhost:80/output.txt'`。）

```shell
curl 'http://localhost:80/app.txt'
Sat Aug 1  11:31:44 CST 2020
Sat Aug 1  11:31:54 CST 2020
Sat Aug 1  11:32:04 CST 2020
```

## SideCar模式

除了上面说的那些之外，我们可以在一个`Pod`中按照顺序启动一个或多个辅助容器，来完成一些独立于主进程（主容器）之外的工作，完成工作后这些辅助容器会依次退出，之后主容器才会启动，这种容器设计模式叫做`sidecar`。

比如对于前端`Web`应用，如果把构建后的`Js`项目放到`Nginx`镜像的`/usr/share/nginx/html`目录下，`Nginx`和`Js`应用做成一个镜像运行容器，每次应用有更新或者`Nginx`要做升级、更新配置操作都需要重新做一个镜像，非常麻烦。

有了`Pod`之后，这样的问题就很容易解决了。我们可以把前端`Web`应用和`Nginx`分别做成镜像，然后把它们作为一个`Pod`里的两个容器"组合"在一起。这个`Pod`的配置文件如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-2
spec:
  initContainers:
  - image: kevinyan/front-app:v2
    name: front
    command: ["cp", "/www/application/*", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: nginx:1.7.9
    name: nginx
    ports:
      - containerPort: 80 # 定义容器提供服务的端口
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: app-volume
  volumes:
  - name: app-volume
    emptyDir: {}
```

所有`spec.initContainers`定义的容器，都会比`spec.containers`定义的用户容器先启动。并且，`Init`容器会按顺序逐一启动，直到它们都启动并且退出了，用户容器才会启动。所以，这个`Init`类型的容器启动后，执行了一句"cp /www/application/* /app"，把应用包拷贝到"/app"目录下，然后退出。这个"/app"目录，挂载了一个名叫`app-volume` 的`Volume`。接下来`Nginx`容器，同样声明了挂载`app-volume`到自己的"/usr/share/nginx/html"目录下。由于这个`Volume` 是被`Pod`里的容器共享的所以等`Nginx`容器启动时，它的目录下就一定会存在前端项目的文件。这个文件正是上面的`Init`容器启动时拷贝到`Volume`里面的。

这就是容器设计模式里最常用的一种模式：`sidecar`。顾名思义，`sidecar`指的就是我们可以在一个`Pod`中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。

## 总结

`Pod`把多个紧密关联的容器组织在一起，让他们共享自己的资源，这点有些像是这些容器的"主机"，只不过这个"主机"是个逻辑概念。当你需要把一个运行在虚拟机里的应用迁移到容器中时，一定要仔细分析到底有哪些进程（组件）运行在这个虚拟机里。然后，你就可以把整个虚拟机想象成为一个 Pod，把这些进程分别做成容器镜像，把有顺序关系的容器，定义为 Init Container。这才是更加合理的、松耦合的容器编排诀窍，也是从传统应用架构，到“微服务架构”最自然的过渡方式。

最后关于Docker In Docker这种把整个应用塞到一个容器里的方法的弊端请查看之前的文章：**[Docker容器的"单进程模型"](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485436&idx=1&sn=7d61ada12981bb77a5f4d84dfa36c6e3&chksm=fa80d66bcdf75f7df5d3ea8943a149f2b5946e580dfc49ecd2ce9fded1de9bd0896494ae07a7&token=169010608&lang=zh_CN&scene=21#wechat_redirect)。**