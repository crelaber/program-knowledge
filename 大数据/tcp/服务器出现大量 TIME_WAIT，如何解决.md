经常在服务器发现一些连接出现 TIME_WAIT 状态，那么为什么会有 TIME_WAIT状态，它是如何产生的？大量的 TIME_WAIT 有什么危害？如何排查？如何优化？带着这些问题逐步分析：

![图片](https://mmbiz.qpic.cn/mmbiz_png/PsfadNsSc5pJLtx6bbicqSgoaZofDjw6cTfFDxPvrRcF0XTicls0YR8icRDdqO1JNO26vRRTa6iae9qy4Qcnd9IuMg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



## **1. TCP 连接回顾**

### **TCP 创建连接（三次握手）**

① 客户端发送连接请求 SYN 报文。

② 服务端接受连接后回复 ACK 报文，并为这次连接分配资源。

③ 客户端收到 ACK 报文后，向服务端发送 ACK 报文，分配资源，TCP 连接建立。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PsfadNsSc5pJLtx6bbicqSgoaZofDjw6c2bEof9wUpAG0VBIVbtmr9kXXPmDQYv09h6I7fvxpbP3hib5RY2iaq0yA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**（图片来源于网络）**

### **TCP 关闭连接（四次挥手）**

① 客户端发送一个 FIN，关闭客户端到服务端的数据传送，客户端进入 FIN_WAIT_1 状态。

② 服务端收到 FIN 后，发送一个 ACK 给客户端，服务端进入 CLOSE_WAIT 状态。

③ 服务端发送一个 FIN，关闭服务端到客户端的数据传送，服务端进入 LAST_ACK 状态。

④ 客户端收到 FIN 后进入 **TIME_WAIT** 状态，发送 ACK 给服务端，服务端进入CLOSED 状态，至此完成四次握手。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PsfadNsSc5pJLtx6bbicqSgoaZofDjw6c80eJRiaO0Agzx1PWmIEgSMPEnRKcV0qPbB1rraTZhq5F3HkMicBqClHA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**（图片来源于网络）**

注意：TCP 连接的任一方均可以调用 close() 以发起主动关闭。上面的 TCP 建立/释放连接的过程，忽略了各种原因引起的重传、拥塞控制等协议细节。

根据上边的分析，TIME_WAIT 状态出现在 TCP 四次挥手中主动关闭连接方发送完最后一次挥手（ACK 的信号结束后），主动关闭连接方所处的状态。

## **2. 为什么有 TIME_WAIT**

**① 可以保证可靠地终止 TCP 连接**，如果处于 TIME_WAIT 的客户端发送给服务器确认报文段丢失的话，服务器将重新发送 FIN 报文段，所以客户端必须处于一个可接收的状态 TIME_WAIT 而不是 CLOSED 状态。

**② 可以保证迟来的 TCP 报文段有足够的时间被识别并丢弃**，某些情况，TCP 报文可能会延迟到达，为了避免迟到的 TCP 报文被误认为是新 TCP 连接的数据，需要在允许新创建 TCP 连接之前，保持一个不可用的状态，等待所有延迟报文的处理。

## **3. 如何确定 TIME_WAIT 时间**

TIME_WAIT 通常也被称为 **2MSL** 等待状态，当切换到 TIME_WAIT 状态的 socket 会保持 2 倍的最大段生命周期（MSL）的延迟时间。

**MSL**（Maximum Segment Lifetime）是 TCP 协议数据报中任意一段数据在网络上被丢弃之前保持可用的最大时间，不同的实现为 MSL 设置了不同的值。

*为什么是 2MSL 的时长呢？相当于至少允许报文丢失一次。比如，若 A**CK 在一个 MSL 内丢失，这样被动方重发的 FIN 会在第 2 个 MSL 内到达，TIME_WAIT 状态的连接可以应对。*

## **4. 如何调整 TIME_WAIT 时间**

① Windows 可以修改注册表修改 TIME_WAIT 的值，（注册表项，TcpTimedWaitDelay）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PsfadNsSc5oXX2pDEpr0fqnNuhq8uicoia9NvGj76A3Gzm9qxfg2p7NpFOntwJ3ELA6xY3YROTPk7pNGwza5cVOQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

② 在 Linux 默认的 TIME_WAIT 时长一般是 60 秒，定义在内核的 include/net/tcp.h 文件中，如果想修改，需要修改内核宏定义重新编译。

```c
#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
                  * state, about 60 seconds */
#define TCP_FIN_TIMEOUT   TCP_TIMEWAIT_LEN/* BSD style FIN_WAIT2 deadlock breaker.
                  * It used to be 3min, new value is 60sec,
                  * to combine FIN-WAIT-2 timeout with
                  * TIME-WAIT timer.
                  */
```

HZ 是用来定义每一秒有几次 timer interrupts，HZ 可在编译核心时设定。

注意：/proc/sys/net/ipv4/tcp_fin_timeout 并不是 TIME_WAIT 时间，而是 **Fin-WAIT-2** 状态超时时间。**TIME_WAIT** 和 **FIN_WAIT2** 状态的最大时长都是 2 MSL，由于在 Linux，MSL 的值固定为 30 秒，所以都是 60 秒。

## **5. TIME_WAIT 有什么影响**

前面介绍过 TIME_WAIT 状态的必要性，但毕竟会消耗系统资源，产生影响。

**① 客户端受端口资源限制**：如果客户端 TIME_WAIT 过多，就会导致端口资源被占用，因为端口就 65536 个，被占满就会导致无法创建新的连接。

**② 服务端受系统资源限制**：理论上服务端可以建立很多连接，虽然只需监听一个端口但会把连接扔给处理线程，所以当服务端出现大量 TIME_WAIT 时，系统资源被占满时，会导致处理不过来新的连接。

## **6. 如何排查 TIME_WAIT**

① 查看状态为 TIME_WAIT 的 TCP 连接

```bash
netstat -tan |grep TIME_WAIT
② 统计 TCP 各种状态的连接数
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(i in S) print i, S[i]}'
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/PsfadNsSc5qOclnibXAjwppNs87HB7NickgkaLLRufCb5beD4CFRB0RVVGmBo3X2MLM29xpS2F07icibjIMkleiaWUw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## **7. 常见 TIME_WAIT 出现场景**

例如， 在 HTTP/1.0 协议中默认使用短连接。浏览器每次请求都需要与服务器建立一个 TCP 连接，服务器处理完成以后立即断开 TCP 连接。

例如，如果 HTTP 请求中，connection 被设置成 close，那就相当于通知 Server 执行完请求后去主动关闭连接。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PsfadNsSc5qOclnibXAjwppNs87HB7Nick4VgH8d3lYPqXz1wRkkWSo83FDOvjUlyg9X56LMic8NXTkvfOyErSdicw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



## **8. 如何优化**

### 调整短链接为长链接

长连接比短连接从根本上减少了关闭连接的次数和 TIME_WAIT 的数量，在高并发的系统中优化效果明显。当使用 nginx 作为反向代理时，为了支持长连接，需要注意：

① **从 Client 到 nginx 的连接是长连接，**默认 nginx 已经开启了对 Client 连接的 keepalive 支持，调整相关参数：

```shell
http {

    keepalive_timeout 120s;    # 客户端连接的超时时间。为 0 的时候禁用长连接。

    keepalive_requests 10000;  # 在一个长连接上的最大请求数。

}
```

**②** **从 nginx 到 Server 的连接是长连接：**配置与上游服务器的长连接需要在 upstream 模块中配置 keepalive

```shell
http {
upstream backend{
  server 10.10.0.1：8080 weight=1;
  server 10.10.0.2：8080 weight=1;
  keepalive 300; #关键配置
 }   

server {
...
location / {
  proxy_pass http://backend;
  proxy_http_version 1.1;         #设置 HTTP 版本为 1.1
  proxy_set_header Connection ""; #设置 Connection 为长连接（默认为no）
  }
 }
}
```

### 调整服务器内核参数

① Linux 提供了 tcp_max_tw_buckets 参数，当 TIME_WAIT 的连接数量超过该参数时，新关闭的连接就不再经历 TIME_WAIT 而直接关闭：

```
[root@kvm-10-115-88-47 ~]# cat /proc/sys/net/ipv4/tcp_max_tw_buckets
32768
```

② 另外 Linux 提供了 tcp_tw_reuse 参数，允许将 TIME_WAIT sockets 重新用于新的 TCP 连接，复用连接。但需要注意，该参数是只用于客户端（建立连接的发起方）。

```
net.ipv4.tcp_timestamps = 1

net.ipv4.tcp_tw_reuse = 1
```

③ Linux 4.12 之前还提供了 tcp_tw_recycle 参数，表示开启TCP连接中TIME-WAIT sockets的快速回收。

注意，当开启了它，比如使用了 NAT，LVS 时，没法保证时间戳单调递增的，会出现部分数据包被服务端拒绝的情况。该参数在内核 4.12 之后被移除，生产环境不建议开启。