登录账号 ： tianjitiangou (您已通过实名认证) 第三方账号绑定
账号ID ： 1733699546440601

域名ID
100000079668
2063630935


172.28.245.158  关机

cdn.xachgd.cn

```````
kubectl get node 
```````

## 重新引用yaml文件

````
kubectl apply -f  *.yaml
````

## 获取所有的namespaces
kubectl -n namespaces 

## 查看状态
kubectl -n qtt-frontend-qa describe  pod qukan-web-50-b99b9fd5-z28m2 

## 将 Namespace qtt-backend-prd 下 deployment contentview 扩容至 8 个 Pod

kubectl scale --replicas=8 deploy/contentview -n qtt-backend-prd

## 驱逐节点
kubectl drain cn-beijing.172.20.65.66 --delete-local-data --force  --ignore-daemonsets

## 将节点设为不可调度 但节点上运行中的Pod不受影响
kubectl cordon 172.128.0.25

## 将节点设为可调度
kubectl uncordon 172.128.0.25

## 查看日志
kubectl logs qukan-web-50-b99b9fd5-z28m2 -n qtt-frontend-qa 
## 查看日志 指定
kubectl logs qukan-web-50-b99b9fd5-z28m2 -n qtt-frontend-qa  -c qukan-web

## 拿dashpod令牌
kubectl get secret -n kube-system

kubectl get secret admin-rulin-token-7zzbz -n kube-system -o yaml

## 拿到的token再用 echo *** |base64 -d 转一下
#拿dashpod令牌

## 进入容器
docker ps 取到对应的CONTAINER ID

## 进入容器
kubectl exec -it  【CONTAINER ID】 /bin/bash 

## 手动扩容
kubectl scale deployment ewx-service --replicas=4 -n qtt-ee-prd  

## 手动设置自动扩容
kubectl autoscale deploy servername --max=16 --min=8 --cpu-percent=40 -n namespace

## 手动更改yaml文件
kubectl edit deploy name -n ns

容器内部编译路径 /data/app/jenkins/home/workspace  jenkins-K8S

## 不能启动的镜像在这里查看（跑在jenkinsbuild上 镜像在jenkins编译里面取）
docker run -it --rm registry.qtt6.cn/qtt-content/video-duplication-detect:origin-pcc_test  /bin/sh

## 存储权限
ceph

## 查看机器标签
kubectl get node --show-labels

## 具有特定标签的节点添加污点
kubectl taint node -l ops=true ops=true:PreferNoSchedule 尽量不要调度过来 
kubectl taint node -l ops=true ops=true:NoSchedule  不调度过来
kubectl taint node -l ops=true ops=true:NoExecute   驱逐不能调度过来的

## 添加标签
kubectl label node $IP role=backend

## cp容器里的内容
kubectl cp /data/web/video-watermark-remove/conf/config.toml

## ipvsadmin -Ln
endpoint 策略

#cat /proc/net/nf_conntrack
链接跟踪

## 一键生成key
echo $(kubectl get secret `kubectl get secret -n qtt-base-prd | grep user-diweihua | awk '{print $1}'` -n db -o yaml | grep 'token: ' | awk  -F ': ' '{print $2}') | base64 -d

##root-KEY-test##
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi1ydWxpbi10b2tlbi03enpieiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbi1ydWxpbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImE3YTAxZGE4LTZmMDYtMTFlOC04YzVjLTAwMTYzZTA4ZjQ2ZCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTphZG1pbi1ydWxpbiJ9.kF_1jHU93lWx9nn0k96iFzT6j88zbgtD8szDmIzOHTBIeuibBg5p8GUF_834rsA28E2-4NlwEvM46ee5utUHUlXhZ9_iuhb6v_IyUWVZ-Hzc_jdW8PReY99fEJvk1B63CSo5NxbDBYfQJqWJTjG4-xbZW7XzLXcVNbvfl3EZD-_qcICiT5rGb9mlAZPj8ZpyM5z3WxunPtA0IFTdO-0BzwJbCi-ZNAyZV8v4znTSG3DhjpXq953XiyKC-PGlnf2VyML7F5mOapJR_iYzy7k0js6OgyUJpF9swo3ktNLQ8v5UuXjEmz1D61RDFnul-HZrjKBIWkYLnF17mxq52XU-MQZQm8NGgUX6Fp8JNFoWhfdXSmr3ffqnbuEuWwWn7Ww-J2rCzDUzCb4jXwSPL_rZCnkHd_jxEi7mtieFWVPCqZNpVGzjFLqjzSxgL7HztlC6uE33P1vGyr_vyQDa5pvP76I76DKD3LO0rI5LA2kljB9vb5MVTMb-jmXdT25ZP0by

##root-KEY-prd##
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi1qaW5ob25neWktdG9rZW4tbTVqbTIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiYWRtaW4tamluaG9uZ3lpIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMmIwOGFhYWEtOWZiYy0xMWU4LTgwOTAtMDAxNjNlMDZiN2I4Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmFkbWluLWppbmhvbmd5aSJ9.FLgBo2Tqk0ZYJX11gxbttjSskddY8JEeOmqM0ceB2FIZDzL6AzJfdmCNzcfWwDezu9840Nz6lJn-S0ssGOeCNdIeZZgM_TtQBgIYbxLnJ1dhSWrW8COsjhYlTYcKFyd2n9qmY0xbTU5MKnveAJNm-m7Z0ky6TewfJnc8jitBeYu8lDYQFPllZFFnVG6J6W0GgQY6gO-4Y2q3ED3X32tw3qooSh_UkittfXHJObdtr9Qh9TEAOXHRrjgKVknDRqnF5q3OmqyZl3sBEN7T0hlRpw1fLpsRUSbnmJit9GKr0dPcc7na18mAzWp63jdDRYHaVpjn28-hQYmYtfJmhCt6XpzyhVy31apj6otnf-fR51usvfqnoGueb_iQskKeLKC-4rmNGbdMsfs89Pmi01arU5VAi-9t4I-xMHvA95B219rHQ-3zcIvMVikRECfsc8W_9bTsra29xkhqlwdU8ErdEtpu9kmNO2gMIDCCbqJihGqwBDH-Dc0j5sbHXe30UdKn

## 测试环境
如使用 ingress , 则解析域名到 :
115.28.253.118

2345.qutoutiao.net

如使用 Kong, 则在 Konga 进行配置
Konga:        http://kong.k8s.tst.qutoutiao.net/#!/dashboard
访问地址:    http://raja.qutoutiao.net:8080/
本地访问:    请在系统hosts中绑定到IP 115.28.252.71

## 生产环境(旧，60段)
如使用 ingress , 则解析域名到
    39.107.212.41


如使用 Kong, 则在 Konga 进行配置
Konga:      http://konga.qutoutiao.net/#!/dashboard
访问地址:  http://kong.qtt.prd/

## 生产环境(新，63段)
如使用 ingress , 则解析域名到
39.105.226.164

kubectl create secret tls admin-midukanshu-com --key xxx.key --cert xxx.pem -n NS


kubectl create secret tls 8qtt-cn --key 8qtt.key --cert 8qtt.pem -n infra-prd


kubectl delete secrets 8qtt-cn  -n infra-prd
kubectl create secret tls 8qtt-cn --key 8qtt.key --cert 8qtt.pem -n infra-prd

## ansible基本命令
ansible -i all_hosts_gin qtt02  -m systemd -a "name=nginx enabled=true"
ansible -i newideanewfiction_hosts_gin app -m shell -a "rm -rf /data/logs/fiction_service_apptt_18100*"
ansible-playbook cront_liangliang.yaml -i newidea_api2
ansible-playbook  /root/masonglin/init/init.yml  -i newidea_api2 -l app -e type=php -t php_fpm_exporterprometheus_for_php
ansible-playbook /data/masonglin/user.yml -i '10.0.2.195,'

ansible  -i newidea_newbs app -m shell -a "crontab -l" -S -R work

for((i=1;i<21;i++)) ;do ansible -i hosts qtt-api-$i -m shell -a 'php /data/web/api.1sapp.com/current/src/public/cli.php Once\\Config.buildLocalCache' && sleep 10 && echo "$i is deon"; done

## strace 跟踪调用

## awk 命令
tail -100000000 /data/logs/nginx/home.1sapp.com.log |awk '{state[$9]++}END{for (A in state) print Astate[A]}'

## sidecar的sudo权限
chmod 655 /etc/sudoers; echo 'work ALL=(ALL) NOPASSWD: /bin/cp /data/etc/negri/negri.service /etc/systemd/system/' >> /etc/sudoers; chmod 440 /etc/sudoers;
chmod 655 /etc/sudoers; echo 'work ALL=(ALL) NOPASSWD: /bin/cp /opt/apps/ccagent/etc/ccagent.service /etc/systemd/system/' >> /etc/sudoers; chmod 440 /etc/sudoers;
## 米读go架构sodu权限
chmod 655 /etc/sudoers; echo 'work ALL=(ALL) NOPASSWD: /bin/cp /data/etc/systemd/*.service /etc/systemd/system/' >> /etc/sudoers; chmod 440 /etc/sudoers;

## kong的sodu权限
chmod 655 /etc/sudoers; echo 'work ALL=(ALL) NOPASSWD:SETENV: /bin/bash run.sh' >> /etc/sudoers; chmod 440 /etc/sudoers;

chmod 655 /etc/sudoers; echo 'work ALL=(ALL) NOPASSWD:SETENV: /bin/bash build.sh' >> /etc/sudoers; chmod 440 /etc/sudoers;

## 判断jeson nginx 状态码非200
tail -f access.log |awk -F ',|"|:' '{if($32!=200){print $0}}'

## 注册consul
./miduconsul --caddr="http://172.16.43.121:8500" batchadd midu_pushmsgservice jjjj.j

./miduconsul --caddr="http://172.16.43.121:8500" batchadd midu_node-exporter jjjj.j #9100

./miduconsul --caddr="http://172.16.43.121:8500" batchadd midu_nginx-exporter jjjj.j #9145

./miduconsul --caddr="http://172.16.43.121:8500" batchadd midu_nginx-exporter jjjj.j #8085

./miduconsul --caddr="http://172.16.43.121:8500" batchadd midu_contentbase jjjj.j #9091

./miduconsul --caddr="http://172.16.43.121:8500"  delete  miduwz_go-gateway_bjg-miduwz-backend-gateway-03

docker rm -vf $(docker ps -aq)
docker rmi -f $(docker images -aq)
docker volume prune -f

## 现有ci接入K8S
def BUILD_COMMAND = "CGO_ENABLED=0 go build -mod=vendor -o newidea_content_base_cmd ./app/cmd"
def PUBLISH_DIR = "./"
def RUN_COMMAND = './newidea_content_base_cmd consumer --runMode=$ENV --healthPort=8080'

## 容器监控
GO:
thanos.qtt5.cn
默认:服务名-metric
monitoring:
  name: uqulive-user-fields
  port: 9009
  path: /metrics
  labels:
    prometheus: prome01
NGINX:

# 监控
默认:服务名-metric
monitoring:
  port: 9145
  path: /metrics
  labels:
    prometheus: prome01

# fpm监控
默认:服务名-metric
fpmMonitoring:
  port: 8085
  path: /metrics
  labels:
    prometheus: prome01


midu-backend-midu-admin-gateway-qa-1.midukanshu.com

## ansible rsync
ansible -i gateway-go.txt app02 -m synchronize -a "src=logrotate.d/ dest=/etc/logrotate.d/"

qtt-newidea-browser-ecs-qa.qutoutiao.net 139.129.198.74

editcap -c 300000 ttt-9090.cap tobkend.cap
editcap -c 300000 ttt-443.cap frondend.pcap
mergecap -w merged.pcap 500.pcapng frondend_00002_20190328233254.cap

type=php nginx_server_name=api.midukanshu.com nginx_server_app_name=midu-gateway app_pm_max_children=600

## git更换使用自定义KEY
自己写个 KEY 到 ~/.ssh/jinhongyi.key
权限chmod 600 ~/.ssh/jinhongyi.key
写一个config文件
vim ~/.ssh/config
 Host qtt
    User jinhongyi
    HostName git.qutoutiao.net
    IdentityFile ~/.ssh/gitjinhongyi.key
 git init 
 git remote add origin git@git.qutoutiao.net:ops-rundeck/midu-rundeck-auth.git
 git commit -m "Initial commit"
 git remote set-url origin git@qtt:ops-rundeck/midu-rundeck-config.git
 git push -u origin master

## 加sleep 延迟pod失败
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

iptables -I INPUT -s 172.16.6.1 -j DROP

*|and not status: 200|select request_url,COUNT(*) GROUP   by request_url

## oss命令
cat .ossutilconfig
[Credentials]
language=EN
endpoint: http://oss-cn-beijing-internal.aliyuncs.com
accessKeyID: LTAI4FdfMJBCp3vEgaZWFxEo
accessKeySecret: wp0ly2dlXIiZGutbl349TRtlVohwaF

./ossutil64 cp jjj.cap oss://public-downloads/


app-202103171829
app-202103161626
app-202103161135
app-202103150945

#批量查看cpu负载
top -bn1 | grep load | awk '{printf "CPU Load: %.2f\n", $(NF-2)}'


1 对语言场景
2 服务可用性上线熔断限流
3 监控可视化
4 动态配置管理

chain 与 invocation 概念 免予理解协议的框架体

实例体

请求特征
service name
header 
consumer
metadata

边缘服务

业务流向和部署态

上帝级服务

可定义的源数据？

roule ingres 方案
invocation？
circuit?

#kong 日志过滤
tail -f /data/logs/kong/access.log |awk -F ',|:|"' '{if($32!=200) print $44,$32}'

pushgateway + consul
172.25.20.105 4c8g
promtheus + rule + alertmanager + grafana
172.25.20.84 8c16g
prometheus:http://172.25.20.84:9090/graph
alertmanager:http://172.25.20.84:9093/#/alerts
consul:http://172.25.20.105:8500
pushgateway:http://172.25.20.105:9091/
grafaba:http://172.25.20.84:3000/?orgId=1

172.25.21.67


external-traffic-nginx

node_systemd_unit_state{name="nginx.service",state="active"} != 1
negri_http_request_total{projectkey1="midu-yl-broswerservice"}

topk(10,sum(irate(redis_error_count{projectkey1="midu-yl-member-go"}[1m])) by (schema))


crd
哦普瑞特尔

case1 todo:  
1、所有均有监控但无报警
2、排查QPS下降的原因
case2 todo:  
1、服务停止无报警
case3 todo:
1、每个服务配置 RT
2、新增RT 增长报警


容器工单需求：
1、框架识别 自动识别QMS框架
2、pedestal框架自动识别 run.sh 中command命令
3、自动check negri 是否已初始化 - 袁帅显示下
4、勾选后自动提交negri工单 -海生
5、自动确认容器工单提交的negri开通工单 - 徐鹏
5、check CCanget 是否已开通 - 海生
6、自动确认容器工单提交的negri开通工单 
7、展示现有应用的ECS资源情况


sum by (host, scheme) (irate(kong_route_http_status{service=~"kong_midu",host="android-logerr.1sapp.com|midu-logserver.1sapp.com|api-platform.1sapp.com",scheme=~"http"}[1m]))


sum(irate(POOL_TotalConn{job="vtm2",pool_name="android-logerr.1sapp.com"}[5m] )) by (pool_name)



url=~"contentbase(.*)



sls 报警配置

panic | select __source__ as serverip,servername,count(*) as errornumber GROUP BY serverip,servername


#回调地址
http://alert-acceptor-wan.qutoutiao.net/api/v1/open_alarm/alarm_info/create_alarm_info_by_aliyun_sls

 {
  "origin": "aliyun_sls",
  "message_type": "alter",
  "location_type": "project",
  "location": "cpc-other-lot-porject",
  "condition": "${condition}",
  "alarm_time": "${FireTime}",
  "level": "high",
  "title": "服务器:${Results[0].FireResult.path}:${Results[0].FireResult.hostname},predict process error 1分钟内日志数小于10当前值：${condition}",
  "Dashboard\t": "${Dashboard}",
  "description": "${DashboardUrl}",
  "url": "cpc-other-lot-porject"
}


{
  "origin": "aliyun_sls",
  "message_type": "alter",
  "location_type": "project",
  "location": "cpc-other-lot-porject",
  "condition": "${condition}",
  "alarm_time": "${FireTime}",
  "level": "disaster",
  "title": "click log 请求量5分钟《跌幅》超过30%  请查看 当前值为：${growth_percent}",
  "Dashboard\t": "${Dashboard}",
  "description": "${DashboardUrl}",
  "url": "cpc-other-lot-porject"
}

###${Results[0].RawResults[1].serverip}

sls: 语法demo
* and __tag__:__hostname__: adx-pre-001  and __tag__:__path__: "/data/app/data_bus/output_01/log/data_bus.INFO" |select hostname,path,if(diff[1] is null, 0, diff[1]) as this_5_min, if(diff[2] is null, 0, diff[2]) as last_5min from (select hostname,path,compare(pv, 3600) as diff from (select count(*) as pv,hostname,path from log group by hostname,path) group by hostname,path)

18511800212,18916504750,13821546293,18210725812,18001273583,18827418627,15764225772,13051181521
服务器:${Results[0].FireResult.serverip}:${Results[0].FireResult.servername} as panic 一分钟panic大于1个

#内存强行用by
(((sum(node_memory_MemTotal_bytes{projectkey0="techcenter_cpc_databus"}) by (instance) - sum(node_memory_MemAvailable_bytes{projectkey0="techcenter_cpc_databus"}) by(instance))
/ sum(node_memory_MemTotal_bytes{projectkey0="techcenter_cpc_databus"}) by (instance) )*100) 



#!/bin/bash
cd /opt/

wget https://di-dataos.oss-cn-beijing.aliyuncs.com/fileserver/jdk/jdk-8u212-linux-x64.tar.gz -O jdk-8u212-linux-x64.tar.gz

tar zxvf jdk-8u212-linux-x64.tar.gz
echo -e "export JAVA_HOME=/opt/jdk1.8.0_212\nexport PATH=\$JAVA_HOME/bin:\$PATH" > /etc/profile.d/java.sh

source /etc/profile

#进程类问题排查命令
sar -u -p 13319
perf top -p 13319
strace -c -p 13319
strace -ttt -p 13319
##统计进程的调用时间
strace -p 13075 -c -f

#获取本地dns
https://www.tx022.com/news/tjdns.html

#请求查看线上版本接口
http://cd.qutoutiao.net/cd-hostinfo/v1/query/app-version?appID=techcenter_architecture_fsync-agent&env=prd

url = '/show' | select diff[1] as this_5_min, diff[2] as last_5_min, diff[3]*100-100 as growth_percent from(select compare(pv, 300) as diff from(select count(*) as pv from log))


url = '/show' | select diff[1] as this_5_min, diff[2] as last_5_min, diff[3]*100-100 as growth_percent from(select compare(pv, 300) as diff from(select  count(*) as pv from log))


EMR数据磁盘重新初始化挂载命令
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdb /mnt/disk1 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdc /mnt/disk2 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdd /mnt/disk3 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vde /mnt/disk4 


rec01-kafka
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdb /mnt/disk1 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdc /mnt/disk2 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdd /mnt/disk3 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vde /mnt/disk4 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdf /mnt/bigdata 


qtt-bigdata
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdb  /mnt/data1 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdc  /mnt/data2 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdd  /mnt/data3 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vde  /mnt/data4 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdf  /mnt/data5 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdg  /mnt/data6 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdh  /mnt/data7 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdi  /mnt/data8 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdj  /mnt/data9 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdk  /mnt/data10 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdl  /mnt/data11 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdm  /mnt/data12 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdn  /mnt/data13 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdo  /mnt/data14 &
/var/lib/ecm-agent/sbin/mnt_disk.sh /dev/vdp  /mnt/data15 


umount /mnt/data1
umount /mnt/data7
umount /mnt/data9
umount /mnt/data2
umount /mnt/data6
umount /mnt/data3
umount /mnt/data15
umount /mnt/data12
umount /mnt/data5
umount /mnt/data4
umount /mnt/data14
umount /mnt/data13
umount /mnt/data11
umount /mnt/data8
umount /mnt/data10


#consul
#删除
curl   --request PUT   http://consul.lotks.cn/v1/agent/service/deregister/nginx-php-172.20.65.204
curl   --request PUT   http://consul.lotks.cn/v1/agent/service/deregister/nodes-172.20.65.210



#注册
 ssh root@39.106.187.159
 cd ansible-newdc 
ansible-playbook -l xk-clickhouse service-register.yml  -e "appname=xk-clickhouse   projectprot=9100  servicenamev1=nodes"
ansible-playbook -l adv-php service-register.yml  -e "appname=adv-php   projectprot=9145  servicenamev1=nginx-php"
ansible-playbook -l adv-php service-register.yml  -e "appname=adv-php  projectprot=8085  servicenamev1=nginx-php"

ansible-playbook -l kafka01 service-register.yml  -e "appname=kafka01  projectprot=7072  servicenamev1=bigdata"

ansible-playbook -l kafka01 service-register.yml  -e "appname=kafka01  projectprot=7072  servicenamev1=bigdata"

ansible-playbook -l adx service-register.yml  -e "appname=adx  projectprot=9587  servicenamev1=app"
ansible-playbook -l xk-as service-register.yml  -e "appname=xk-as  projectprot=9002  servicenamev1=app"
ansible-playbook -l xk-bs service-register.yml  -e "appname=xk-bs  projectprot=9003  servicenamev1=app"

ansible-playbook -l adx-pre service-register.yml  -e "appname=adx-pre  projectprot=9100  servicenamev1=nodes"

ansible-playbook -l adx-pre service-register.yml  -e "appname=adx-pre  projectprot=9020  servicenamev1=app"

adv-go adv-mkt  adx

=============大数据==============
consul注册
机器：172.16.6.1
说明：/data/gaoshilong/sh/readme-aliyun-nsq
脚本：aliyun-nsq
命令：./aliyun-nsq  --caddr="http://172.16.182.245:8500"  batchadd node_exporter nsq.txt

备注：
172.16.182.245:8500，consul 基础监控节点
batchadd 批量添加
node_exporter，consul中服务名
nsq.txt，批量ip:port格式

consul删除
172.16.182.245，这台机器，/data/app/consul/data  这个目录
把要下线的ip  写到nsq.txt
执行，clear.sh这个脚本
拿到id号后，去services 目录去删除id
删除完以后，执行，consul reload


大数据监控（机器信息）
https://km.qutoutiao.net/pages/viewpage.action?pageId=385158863

大数据监控（业务方使用）
https://km.qutoutiao.net/pages/viewpage.action?pageId=115088649

大数据consul注册
https://km.qutoutiao.net/pages/viewpage.action?pageId=132427750

大数据初始化脚本（salt）
https://km.qutoutiao.net/pages/viewpage.action?pageId=150737199

大数据hadoop基础安装
https://km.qutoutiao.net/pages/viewpage.action?pageId=263751990

大数据kafka交接
https://km.qutoutiao.net/pages/viewpage.action?pageId=406765882

kafka-manager: 
http://di-kakfaadmin.1sapp.com/
用户名：admin   密码：fhxiEwmjsgpOGIo4

1p 1W

#登录 172.16.152.78 初始化服务器
salt -L "172.16.190.216,172.16.191.145" state.sls saltenv='hadoopali' init.init-all

#登录 172.16.182.252 安装hadoop
salt -L "di-h1-dn5-84.h.ab1.qttsite.net,di-h1-dn5-180.h.ab1.qttsite.net" state.sls hadoop   （重置不需要）
salt -L "di-h1-dn5-84.h.ab1.qttsite.net,di-h1-dn5-180.h.ab1.qttsite.net" state.sls hadoop.conf_init （重置不需要）
salt -L "di-h1-dn5-176.h.ab1.qttsite.net,di-h1-dn5-116.h.ab1.qttsite.net" state.sls hadoop.dn_init_15 

#登录 172.16.182.252 安装Oss依赖
sh /root/autoDi/deployOss/deployOss.sh di-h1-dn5-84.h.ab1.qttsite.net 

## 阿里磁盘在线扩容

yum install cloud-utils-growpart -y
yum install xfsprogs -y

growpart /dev/vda 1
resize2fs /dev/vda1   


深入理解计算机系统
深入理解linux内核
程序员的自我修养
https://developer.ibm.com/tutorials/l-completely-fair-scheduler/



google   anthos -> anthos service mesh

./mnt_disk.sh  /dev/vdb    /data1 &
./mnt_disk.sh  /dev/vdc    /data2 &
./mnt_disk.sh  /dev/vdd    /data3 &
./mnt_disk.sh  /dev/vde    /data4 &
./mnt_disk.sh  /dev/vdf    /data5 &
./mnt_disk.sh  /dev/vdg    /data6 &
./mnt_disk.sh  /dev/vdh    /data7 &
./mnt_disk.sh  /dev/vdi    /data8 &
./mnt_disk.sh  /dev/vdj    /data9 &
./mnt_disk.sh  /dev/vdk    /data10 &
./mnt_disk.sh  /dev/vdl    /data11 &
./mnt_disk.sh  /dev/vdm    /data12 &
./mnt_disk.sh  /dev/vdn    /data13 &
./mnt_disk.sh  /dev/vdo    /data14 &
./mnt_disk.sh  /dev/vdp    /data15 &
./mnt_disk.sh  /dev/vdq    /data16 &
./mnt_disk.sh  /dev/vdr    /data17 &
./mnt_disk.sh  /dev/vds    /data18 &
./mnt_disk.sh  /dev/vdt    /data19 &
./mnt_disk.sh  /dev/vdu    /data20 &
./mnt_disk.sh  /dev/vdv    /data21 &
./mnt_disk.sh  /dev/vdw    /data22 &
./mnt_disk.sh  /dev/vdx    /data23 &
./mnt_disk.sh  /dev/vdy    /data24 &
./mnt_disk.sh  /dev/vdz    /data25 &
./mnt_disk.sh  /dev/vdaa   /data26 &
./mnt_disk.sh  /dev/vdab   /data27 &
./mnt_disk.sh  /dev/vdac   /data28


rsync -avz config 172.20.65.99:/data/app/consul/config  #consul配置文件
rsync -avz ./consul-client 172.20.65.99:/data/app/consul/bin/consul-client #consul客户端
rsync -avz conf.recover.json 172.20.65.99:/data/etc/negri/conf.recover.json negri日志配置文件
rsync -avz ./negri/current/negri 172.20.65.99:/data/web/negri/current/ negri应用程序



成本多级资源池 容器跨集群调用

监控数据输出
1、有使用但是没有持续
2、监控的陈本有没有有效的提现 特别是 trace 关闭后也没啥
3、监控数据有没有稳定性的要求 需要调研



2、容器运行时容量管理
k8s集群cpu/内存水位超过80%进行扩容，水位低于30%进行缩容
联邦集群、离在线业务调度，充分利用计算资源


扩容:
业务独占（如：算法、大数据等）  
    业务提交申请 同ECS申请流程

业务托管 （如：米读、直播等）
由容器组判断是否突发情况
    突发 -> ecc -> 财务/TC审核
    正常 -> 走一键扩容 -> 业务运维 财务/TC审核

缩容:
    1、资源池利用率小于 30% 进行容器节点调度
    2、业务独占的由业务自行提出需求要求缩容

参加人员 : 褚玉巧 狄卫华 董鹏 董术 金泓宜 刘行 李尊 徐孝敏 郑庆元 等12名



大数据分摊
ecs.c5.16xlarge  * 4 6896

ecs.gn5i-c16g1.4xlarge

@all
各位大佬好
在上周篮军护网行动中我们发现很多服务器都是闲置状态，在经过一系列确认后我们进行关机处理
现已关机一周并未收到任何同学的反馈，相关服务器列表如下：
###################
https://docs.qq.com/sheet/DQUpHWFdLT2dTc2pJ?tab=BB08J2
#############

注意！！！：
1、明天18:00前还无反馈的我们将对这些ECS做回收处理
2、有任何问题可直接联系我进行备注或再次确认
3、一旦释放回收数据将不可以追回还有劳各位看一下

谢谢！

米读推荐容器化：服务代码开发已完成，目前在测试接入，成本还没提现。
米读wzuser job优化：wzuser目前总共3个任务，上周已完成代码开发，本周会上线阅读时长job任务，上线后估计可以优化115C。
米读存量ecs优化：上周下线+降配共28C。
整体CPU利用率(均值)提升至30%：fiction(60C)，readbase-go(72C)、midu-base(8C)、pgc(3C)等容器服务完成了缩容143C，利用率没有效果，反而有微降，本周也会重点再筛一批服务出来优化。


di-h1-dn-176.h.ab1.qttsite.net
di-h1-dn5-97.h.ab1.qttsite.net
di-h1-dn5-43.h.ab1.qttsite.net

每周篮球活动
参与人: 董鹏 董术 狄伟华 李尊 金泓宜 徐孝敏 闫飞虎 郑庆元 褚玉巧 等12名


