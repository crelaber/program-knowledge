我们这个[系列的文章](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUzNTY5MzU2MA==&action=getalbum&album_id=1394839706508148737&scene=173&from_msgid=2247488517&from_itemidx=1&count=3&nolastread=1#wechat_redirect)一直都在学习和掌握K8S各种组成部分在集群里的角色、作用和使用场景，那么针对今天这个主题任务「给K8S上的Web服务做域名解析」你觉得应该使用什么组件来完成呢？

如果你看过我上一篇文章[你知道K8S暴露服务的方式有哪些吗？](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247488517&idx=1&sn=f58b32757b39d617f8769e1b65cef71e&chksm=fa80c592cdf74c846016d8036898aa2f4aec4662485f40c1cf959640def1e58e6d131fc3af78&token=1465862111&lang=zh_CN&scene=21#wechat_redirect) 应该能猜到是使用`Ingress`，那么有人会问为啥不能用`NodePort`这种方式呢？今天的文章我们就来详细探讨一下这些相关的问题：

- 为什么`NodePort`这种暴露服务的方式不适合用来给服务做域名解析。
- 怎么使用`Ingress`暴露Web服务（会给大家做一个Demo进行演示）。
- 生产集群`Ingress`怎么做高可用。

## 为什么NodePort不适合做域名解析

`NodePort` 类型的`Service` 是向集群外暴露服务的最原始方式，也是最好让人理解的。`NodePort`，顾名思义，会在所有节点（宿主机或者是VM）上打开一个特定的端口，发送到这个端口的任何流量都会转发给`Service`。

`NodePort` Service 的原理可以用下面这个图表示：

NodePort Service的原理图

上图我们顺着流量的流向箭头从下往上看，流量通过`NodeIP+NodePort`的方式进入集群，上图三个节点的30001上的流量都会转发给Service，再由Service给到后端端点`Pod`。

NodePort Service的优点是简单，好理解，通过IP+端口的方式就能访问，但是它的缺点也很明显，比如：

- 每向外暴露一个服务都要占用所有Node的一个端口，如果多了难以管理。
- NodePort的端口区间固定，只能使用30000–32767间的端口。
- 如果Node的IP发生改变，负载均衡代理需要跟着改后端端点IP才行。

## 怎么使用Ingress暴露Web服务

在K8S的这些组件中`Ingress` 不是一种`Service`。相反，它位于多个`Service`的前端，给这些`Service`充当“智能路由代理”或集群的入口点（entrypoint）。

在`Service`前面加上`Ingress`，集群中需要有`Ingress-Controller`才行。如果是自建K8S集群，通常使用`nginx-ingress`作为控制器，它使用`NGINX`服务器作为反向代理来把流量路由给后面的`Service`。

通过`Ingress`可以对后端`Service`进行基于域名和URL路径的路由。例如，您可以将 foo.yourdomain.com 上的所有内容发送到 foo Service，将 yourdomain.com/bar/ 路径下的所有内容发送到 bar Service。

Ingress运行原理

我们可以把这张图和上面NodePort原理图做一个比较，看看两者的区别。上面这个示意图对应的Ingress资源声明文件差不多应该长这个样子：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  backend: // 哪个都不匹配时，走这个兜底的Service
    serviceName: other
    servicePort: 8080
  rules:
  - host: foo.mydomain.com
    http:
      paths:
      - backend:
          serviceName: foo
          servicePort: 8080
  - host: mydomain.com
    http:
      paths:
      - path: /bar/
        backend:
          serviceName: bar
          servicePort: 8080
```

### 在本地实践Ingress

上面说了很多理论，下面我们可以通过一个简单的Demo进行演示，我本地使用的是`Docker Desktop`自带的K8S集群，至于为啥用它，没别的就是简单。

要使用`Ingress`，那么集群里就得先有个`Ingress-Controller`，我们先来安装一个`Nginx-Ingress-Controller` （其本质上就是一个Nginx Server）：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.0/deploy/static/provider/cloud/deploy.yaml
```

执行上述命令后，其实会在本地集群的`ingress-nginx`命名空间里，安装三个创建`nginx-controller`的`Pod`:

```bash
➜  ~ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-n59dn        0/1     Completed   2          49d
ingress-nginx-admission-patch-hft52         0/1     Completed   3          49d
ingress-nginx-controller-68649d49b8-nlch7   1/1     Running     77         49d
```

前两个`admission`相关的`Pod`是给`nignx controller`做配置用的，执行完就退出，第三个才是运行着`nginx-controlelr`的`Pod`。

我之前几次在本地试验`Ingress`没有成功，就是因为这个`nignx-controller`没有正常启动起来。一个常见的错误是`nginx controller` 这个`Pod`拉取镜像`k8s.gcr.io/ingress-nginx/controller:v0.46.0@sha256:....` 的速度非常慢，所以我们最好是在安装控制器前先通过`docker pull`命令把这个镜像拉到本地。

在本地安装完Ingress控制器后，为了演示方便，之前本地搭建`Nacos`时做过一个`Service`，`Nacos`是阿里巴巴出的一个可以做自动配置和服务注册的组件，自带Web管理界面，正好可以拿它来演示。

下面我想能通过`dev.nacos.com`访问`nacos-service`，最终让`nacos-service`把流量路由给后端安装了`Nacos`服务的`Pod`，这个`Ingress`可以这么声明

```yaml
//ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nacos-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
    - host: dev.nacos.com
      http:
        paths:
          - path: /
            backend:
              serviceName: nacos-service
              servicePort: 8848
```

在集群里应用这个`Ingress`后

```bash
# 执行下面两个命令
# kubectl apply -f nacos-ingress.yaml
# kubectl get ingress
➜  ~ kubectl get ingress
NAME            CLASS    HOSTS           ADDRESS     PORTS   AGE
nacos-ingress   <none>   dev.nacos.com   localhost   80      54d
```

再在本地hosts文件里绑定上

```bash
127.0.0.1 dev.nacos.com # 本地K8S集群的IP是127.0.0.1
```

就能通过域名访问到`Nacos`服务的管理界面

Nacos的Web界面

为了不影响阅读，Service、Deployment这些声明文件，我放在了GitHub 仓库里，链接如下：

```bash
https://github.com/kevinyan815/LearningKubernetes
```

如果有啥疑问可以通过留言或者加微信号fsg1233210与我讨论。

## 生产集群`Ingress`怎么做高可用

上面我们聊了`Ingress`怎么暴露服务，以及在本地怎么实践演练用`Ingress`暴露服务，那么有的人肯定会好奇，在生产集群里`Ingress`是怎么做高可用的呢？域名解析应该怎么绑定呢？

正常的生产环境，因为`Ingress`是公网的流量入口，所以压力比较大肯定需要多机部署。一般会在集群里单独出几台`Node`，只用来跑`Ingress-Controller`，可以使用`deamonSet`的让节点创建时就安装上`Ingress-Controller`，在这些Node上层再做一层负载均衡，把域名的DNS解析到负载均衡IP上。

考虑到多业务线服务相互不影响的话，还需要让不同的业务线的转发规则注入到不同的`Ingress`里，这个通过我们上面声明Ingress时的注解`annotations:kubernetes.io/ingress.class`就能实现。

其实每家公司的方案肯定不一样，尤其是解析链路里加上高防Waf的话，会更复杂，由于我不是专业运维，也只是知道一些大概的思路，如果有专业的大佬欢迎留言，让我们共同进步一下。

## 最后

其实这个主题我一直想写，之前断断续续尝试了两三次才在本地把`Ingress`这套跑通，搞明白。希望今天的文章能帮大家进行一下`Ingress`知识的分析，如果想自己掌握、想明白，还是需要把文章好好看看，亲自实践一下演示的例子，以及多复习复习前面关于[Service](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486082&idx=1&sn=42a9bc8fcfc9da09445e9e2f4cf2fb96&chksm=fa80db15cdf752039494992f71a3bc488cf386841bd1aaaa44115f5e7f155ba55ce468ec89ee&token=2075750696&lang=zh_CN&scene=21#wechat_redirect)的知识才行。