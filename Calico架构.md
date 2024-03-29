# calico架构
![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/calico.jpg)

### Felix：
calico agent，运行在每台主机上。它的主要功能包括:
- （1）接口管理：把接口相关的信息编程到内核中，方便内核处理endpoint发送的消息。
- （2）路由编程：把路由信息写入到FIB（Forwarding Information Base）表中
- （3）ACL编程：ACL权限写入到kernel中
- （4）状态报告：提供网络监控的相关数据
### etcd：
calico可以外挂ectd存储，把数据存储在etcd中
### BIRD：
BIRD是一个BGP client，它从FIB中读取路由信息，并把它分发给数据中心其他节点。一般和Felix部署在一起
### Confd: 
从etcd中读取数据，生成bird配置文件。
### BGP Route Reflector (BIRD)：
小规模网络下，BGP client是相互连接node to node mesh，组成一个网状网络。这种方式不适合于大规模的组网场景。而RR则用在大规模的网络模式下，这样BGP client直接和RR进行互联，极大减少网络连接的数量。RR接收到BGP client端路由新后，会自动把路由信息广播到数据中心其他路由节点。
### calicoctl: 
calcio的命令行工具，用来配置和查询calico信息。可以部署在一台或者多台主机上。
### The Orchestrator plugin：
这个编排插件没有体现在上面的结构图中，它和具体的编排服务（openstack，k8s）相关。编排插件实现api转换（比如把k8s的网络模型转换为calico的网络模型），calico节点的状态反馈（比如把bird存活状态反馈给k8s）
# calico部署
calico的部署，参考官方文档，支持k8s,openshift,openstack,物理机方式部署。我们选择k8s的部署方式

部署要求，参考：

https://docs.projectcalico.org/v3.7/getting-started/kubernetes/requirements

部署方式，采用etcd外置方式

https://docs.projectcalico.org/v3.7/getting-started/kubernetes/installation/calico#installing-with-the-etcd-datastore)

基于etcd方式部署，有如下节点需要注意：
- 1：注意更改POD_CIDR范围，需要和k8s的POD范围保持一致
- 2：注意etcd信息的配置，把etcd的配置信息替换为实际的证书信息
- 3：如果有多网卡的需要指定calico使用的网卡信息，否则会报错

部署完成后，可以看到类似如下信息：
```
# kubectl get pod -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7f866ddd89-2fvhb   1/1     Running   4          10d
calico-node-gglhh                          2/2     Running   6          10d
calico-node-qfkxq                          2/2     Running   4          5d4h
```
# calico部署架构
calico从部署架构来讲，主要是包括两个节点：calico controller节点，calico node节点。
## calico controller
controller包含如下几个能力：
- policy controller: watches network policies and programs Calico policies.
- namespace controller: watches namespaces and programs Calico profiles.
- serviceaccount controller: watches service accounts and programs Calico profiles.
- workloadendpoint controller: watches for changes to pod labels and updates Calico workload endpoints.
- node controller: watches for the removal of Kubernetes nodes and removes corresponding data from Calico.
## calico node
每个主机上部署一个,DaemonSet类型.集成了felix，bird，confd，cni，ipam的能力。
下面，我们找一个pod，深入观察一下calico node的内部信息
我们 docker inspect calico node的image，可以看到image的启动脚本为：start_runit
```
    "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"start_runit\"]"
            ],
```
```
#  kubectl exec -it calico-node-gglhh   -n kube-system /bin/sh

/ # vi /sbin/start_runit
```
可以看到环境变量信息放到：
env > /etc/envvars
```
/ # cat /etc/envvars
可以看到类如下信息：
FELIX_IPV6SUPPORT=false
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.254.0.1:443

其中etcd的相关配置类似于信息：
/ # cat /etc/envvars |grep ETCD
ETCD_CA_CERT_FILE=/calico-secrets/etcd-ca
ETCD_CERT_FILE=/calico-secrets/etcd-cert
ETCD_ENDPOINTS=https://xxx:2379,https://xxxx:2379
ETCD_KEY_FILE=/calico-secrets/etcd-key
```
查看一下系统运行的进程：
```
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
 ```
系统服务由4个可用服务：bird，bird6，confd，felix

进入/etc/calico/confd/config，可以看到对应bird的配置文件
```
/etc/calico/confd/config # ls
bird.cfg        bird6.cfg       bird6_aggr.cfg  bird6_ipam.cfg  bird_aggr.cfg   bird_ipam.cfg
/etc/calico/confd/config# cat bird.cfg
可以看到类似内容：
# ------------- Node-to-node mesh -------------

# Node-to-node mesh disabled

# ------------- Global peers -------------

# For peer /global/peer_v4/192.168.56.40
protocol bgp Global_192_168_56_40 from bgp_template {
  neighbor 192.168.56.40 as 64512;
}

# ------------- Node-specific peers -------------

# No node-specific peers configured.

```
备注：
- Node-to-node mesh：如果开启了该模式，这会生成对应的记录。这个calico默认的方式。会针对除了本机外的其他主机，生成一条记录
- Global peers：这里的例子中，设置了全局的bgp peer，这样所有的node节点直接和该peer交换信息。
- Node-specific peers：指针对单个node设置peer信息

cni部分：根据calico.yaml安装文件，在每个物理机的/etc/cni/net.d目录下，可以看到cni的配置信息
```
[root@k8s01 net.d]# pwd
/etc/cni/net.d
[root@k8s01 net.d]# ll
total 8
-rw-r--r--  1 root root  730 Jul  1 17:29 10-calico.conflist
-rw-------. 1 root root 3022 Jul  1 17:29 calico-kubeconfig
drwxr-xr-x. 2 root root   54 Jul  1 17:29 calico-tls
```
bin文件放到/opt/cni/bin，用到的是calico-ipam
```
[root@k8s01 ca]# ll  /opt/cni/bin
total 66648
-rwxr-xr-x 1 root root 26889120 Jul  1 17:29 calico
-rwxr-xr-x 1 root root 26233504 Jul  1 17:29 calico-ipam
-rwxr-xr-x 1 root root  2814104 Jul  1 17:29 flannel
-rwxr-xr-x 1 root root  2991965 Jul  1 17:29 host-local
-rwxr-xr-x 1 root root  3026388 Jul  1 17:29 loopback
-rwxr-xr-x 1 root root  3470464 Jul  1 17:29 portmap
-rwxr-xr-x 1 root root  2808402 Jul  1 17:29 tuning

```

# 数据路径

calico通过BGP协议交换路由信息，路由信息拿到后，再把相关信息写入os kernel的路由表和iptables，没有额外的动作。
![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/datapath.JPG)
