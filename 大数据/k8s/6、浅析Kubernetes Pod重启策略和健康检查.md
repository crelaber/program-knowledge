使用`Kubernetes`的主要好处之一是它具有管理和维护集群中容器的能力，几乎可以提供服务零停机时间的保障。在创建一个`Pod`资源后，`Kubernetes`会为它选择worker节点，然后将其调度到节点上运行`Pod`里的容器。`Kubernetes`强大的功能可使应用程序的容器保持连续运行，还可以根据需求的增长自动扩展系统。除此之外在`Pod`或容器出现故障时`Kubernetes`还可以让系统实现"自愈"。在本文中，我们将介绍如何使用`Kubernetes`内置的`livenessProbe`和`readinessProbe`来管理和控制应用程序的运行状况。

## Pod的重启策略

`Kubernetes`自身的系统修复能力有一部分是需要依托`Pod`的重启策略的， 重启策略也叫`restartPolicy`。它是`Pod` 的`Spec`部分的一个标准字段（pod.spec.restartPolicy），默认值是Always，即：任何时候这个容器发生了异常，它一定会被重新创建。

可以通过设置 restartPolicy，改变 Pod 的恢复策略。除了 Always，它还有 OnFailure 和 Never 两种情况。

- Always：在任何情况下，只要容器不在运行状态，就自动重启容器；
- OnFailure: 只在容器 异常时才自动重启容器；
- Never: 从来不重启容器。

在实际使用时，我们需要根据应用运行的特性，合理设置这三种恢复策略。

对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数，比如：

```shell
$ kubectl get pod test-liveness-exec
NAME           READY     STATUS    RESTARTS   AGE
liveness-exec   0/1       Running   1          1m
```

如果一个 Pod 里只有一个容器，然后这个容器异常退出了。那么，只有当 restartPolicy=Never 时，这个 Pod 才会进入 Failed 状态。而其他情况下，由于 Kubernetes 都可以重启这个容器，所以 Pod 的状态保持`Running` 不变，RESTARTS信息统计了`Pod`的重启次数。需要注意的是：虽然是重启，但背后其实是`Kubernetes`用重新创建的容器替换了旧容器。

## Pod怎么实现自我修复？

将`Pod`调度到某个节点后，该节点上的Kubelet将运行其中的容器，并在`Pod`的生命周期内保持它们的运行。如果容器的主进程崩溃，`kubelet`将重新启动容器。但是，如果容器内的应用程序抛出错误导致其不断重启，则`Kubernetes`可以通过使用正确的诊断程序并遵循`Pod`的重启策略来对其进行修复。

`Kubernetes`可以对两种健康检查做出应对：

- Liveness：活性检查，`kubelet`使用活性探针（livenessProbe）的返回状态作为重新启动容器的依据。一个Liveness探针用于在应用运行时检测容器的问题。容器进入此状态后，`Pod`所在节点的`kubelet`可以通过`Pod`策略来重启容器。
- Readiness：就绪检查，这种类型的探测（readinessProbe）用于检测容器是否准备好接受流量。你可以使用这种探针来管理哪些`Pod`会被用作服务的后端。如果`Pod`尚未准备就绪，则将其从服务的后端列表中删除。

`Kubernetes`把放在`Pod`里的健康检查处理程序叫做探针（Probe），比喻成医学手术上探测病变的探针，还是很形象的。

## 探针处理程序

为了使健康检查能够对`Pod`的运行状况进行诊断，`kubelet`会调用容器中为探针实现的处理程序，这些处理程序分为三大类：

- Exec：在容器内执行命令。
- TCPSocket：对指定端口上，容器的IP地址执行TCP检查。
- HTTPGet：在容器的IP上执行HTTP GET请求。

处理程序的返回状态也分为三种：

- Success：容器通过诊断。
- Failed：容器无法通过诊断。
- Unknown：诊断失败，状态不确定，将不采取任何措施。

聊完了探针程序的种类和返回值接下来我们来了解一下这两种探针的使用案例。

## 使用案例

活性和就绪探针都在`Pod`的`YAML`文件中配置。每种类型都有不同的用例。

### livenessProbe

如前所述，活性探针用于诊断不健康的容器。他们可以在服务无法继续进行时检测到服务中的问题，并会根据其重启策略重启有问题的容器，期望通过这种方式来解决服务的问题。

例如，在容器内包含一个Exec活性探针，以检测应用程序何时转换为`Broken`状态。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

这个`YAML`定义一个单容器的`Pod`。它表示`kubelet`在容器启动完成后5秒进行第一次健康检查（initialDelaySeconds：5），之后每5秒都会执行一次检查（periodSeconds: 5）。`kubelet`在容器中执行命令"cat/tmp/healthy"，如果成功，则返回0，指示它是健康的。如果返回非零值，则`kubelet`将kill掉该容器并重新启动它。

这个例子是官方教程里给的，除了在容器中执行命令外，发起 HTTP 或者 TCP 请求的livenessProbe，教程里也给出了示例：

```yaml
...
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
...
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

在官方教程里可以找到更多示例，链接：https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

### readinessProbe

就绪探针与活性探针（GET请求，TCP连接和命令执行）进行相同类型的检查。但是，纠正措施有所不同。它不会重启未通过检查的容器的`Pod`，而是从`Service`上摘除`Pod`，暂时将其与流量隔离。

比如，有一个`Pod`可能正在做大量计算或正在进行繁重的操作，从而增加了服务的响应延迟。让我们定义一个就绪探针，通过探针的timeoutSeconds参数定义超过两秒响应GET请求的`Pod`的状态为非健康状态

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: nodedelayed
spec:
 containers:
   - image: k8s.gcr.io/liveness
     name: liveness
     ports:
       - containerPort: 8080
         protocol: TCP
     readinessProbe:
       httpGet:
         path: /
         port: 8080
       timeoutSeconds: 2
```

当探测响应时间超过两秒钟时，kubelet不会重新启动Pod，而是将流量路由到其他健康的`Pod`上。等到`Pod`不再过载后，`kubelet`会将`Pod`重新加回到原来的`Service`中。

## 总结

默认情况下，`Kubernetes`提供两种健康检查：readinessProbe 和 livenessProbe。它们都使用相同类型的探针处理程序（HTTP GET请求，TCP连接和命令执行）。他们对未通过检查的`Pod`做出的纠错措施有所不同。livenessProbe将重新启动容器，预期重启后错误不再发生。readinessProbe会将Pod与流量隔离，直到故障原因消失。

通过在同一个`Pod`中使用这两种健康检查，可以确保流量不会到达尚未准备就绪的`Pod`，并且确保`Pod`在发生故障时能重新启动。

良好的应用程序设计应同时记录足够的信息，尤其是在引发异常时。它还应公开必要的API端点，这些端点将会传达重要的运行状况和状态指标，以供监控系统（如Prometheus）使用。

### 参考链接：

**kubernetes-best-practices-health-probes**[1]

**liveness-and-readiness-probes-in-kubernetes**[2]

### 参考资料

[1]kubernetes-best-practices-health-probes: https://www.magalix.com/blog/kubernetes-and-containers-best-practices-health-probes[2]liveness-and-readiness-probes-in-kubernetes: https://www.weave.works/blog/resilient-apps-with-liveness-and-readiness-probes-in-kubernetes