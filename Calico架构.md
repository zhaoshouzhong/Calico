## 1：架构

- Felix：calico agent，运行在每台主机上。它的主要功能包括:
（1）接口管理：把接口相关的信息编程到内核中，方便内核处理endpoint发送的消息。
（2）路由编程：把路由信息写入到FIB（Forwarding Information Base）表中
（3）ACL编程：ACL权限写入到kernel中
（4）状态报告：提供网络监控的相关数据
- The Orchestrator plugin：编排插件，和具体的编排服务（openstack，k8s）相关。
- etcd：calico数据存储在etcd中
- BIRD：BGP client读取路由信息，并把它分发给数据中心其他节点。一般和Felix部署在一起
- BGP Route Reflector (BIRD)：小规模网络下，BGP client是相互连接，组成一个网状网络。这种方式不适合于大规模的组网场景。而RR则用在大规模的网络模式下，这样BGP client直接和RR进行互联，极大减少网络连接的数量。RR接受到BGP client端路由新后，
会自动把路由信息广播到数据中心其他路由节点。

## 2：实例

我们找一个pod查看一下calico node的内部信息
```javascript
# kubectl exec -it calico-node-f44t7 -n kube-system /bin/sh
Defaulting container name to calico-node.
Use 'kubectl describe pod/calico-node-f44t7 -n kube-system' to see all of the containers in this pod.
```

查看calico node信息，它的启动脚本为/sbin/start_runit
```javascript
/ # vi /sbin/start_runit 
环境变量信息放到：
env > /etc/envvars

/ # cat /etc/envvars 
FELIX_IPV6SUPPORT=false
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.254.0.1:443

可以看到etcd的相关配置信息：
/ # cat /etc/envvars |grep ETCD
ETCD_CA_CERT_FILE=/calico-secrets/etcd-ca
ETCD_CERT_FILE=/calico-secrets/etcd-cert
ETCD_ENDPOINTS=https://xxx:2379,https://xxxx:2379
ETCD_KEY_FILE=/calico-secrets/etcd-key

查看一下系统运行的进程：
/ # ps
PID   USER     TIME   COMMAND
    1 root       0:02 /sbin/runsvdir -P /etc/service/enabled
   73 root       0:00 runsv felix
   74 root       0:00 runsv bird
   75 root       0:00 runsv bird6
   76 root       0:00 runsv confd
   77 root      10:17 confd -confdir=/etc/calico/confd
   78 root       0:18 bird -R -s /var/run/calico/bird.ctl -d -c /etc/calico/confd/config/bird.cfg
   79 root       0:16 bird6 -R -s /var/run/calico/bird6.ctl -d -c /etc/calico/confd/config/bird6.cfg
   80 root      32:13 calico-felix
 9316 root       0:00 /bin/sh
 9611 root       0:00 /bin/sh
 9616 root       0:00 ps
 
 系统服务由4个可用服务：bird，bird6，confd，felix
其中：
confd：is a simple configuration management tool. It reads values from etcd and writes them to files on disk。
```

==3：数据路径

calico通过BGP协议交换路由信息，路由信息拿到后，再把相关信息写入os kernel的路由表和iptables，没有额外的动作。因此，calico的效率高很显然了。
