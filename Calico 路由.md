Calico 部署完成后，会生成对应的路由表。
# 主机路由表
登录一个物理机，输入 route 指令可以看到：
```
# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
172.xx.75.5     0.0.0.0         255.255.255.255 UH    0      0        0 calie74c69f09c4
172.xx.75.11    0.0.0.0         255.255.255.255 UH    0      0        0 cali79e3e1514e7
172.xx.238.192  host19          255.255.255.192 UG    0      0        0 tunl0
```
对应calixx的ip就是docker的ip地址
对应host19 的为物理机host19 对应的 calico网段起始ip。
登录每台物理机，都可以看到类似的路由表。可以看出calico的路由生成规则：
- 1：生成本机的docker ip到路由表中
- 2：生成其他物理机的路由信息，则通过该路由规则可以访问到其他物理机的docker

备注：Use Iface部分，物理机的对应的为tunl0，这表明我们采用的是默认的node to node mesh 组网方式。
如果我们改变一下策略，比如采用
```
 - name: CALICO_IPV4POOL_IPIP 
   value: "off" 
```
或者CrossSubnet，则route表变为类似内容：

```
# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
172.xx.73.69    0.0.0.0         255.255.255.255 UH    0      0        0 cali8eb4bab48a8
172.xx.236.128  k8s02           255.255.255.192 UG    0      0        0 enp0s8

# ip route
172.xx.236.128/26 via 192.168.56.102 dev enp0s8 proto bird
```
则Use Iface部分变为主机的物理网卡名称。

192.168.56.102 为k8s02的主机ip
这样就很清晰了，通过k8s02（192.168.56.102），可以访问到172.xx.236.128/26的地址段。那么172.xx.236.128/26这个段是怎么分配的？

# calico IP分配规则
从上面可以看出，针对不同的物理机，calico分别不同的ip路由规则。这些物理机和ip网段的对应规则怎么可以查到呢？答案在etcd中。
查询etcd
```
etcdctl get /  --prefix --keys-only |grep calico
/calico/ipam/v2/assignment/ipv4/block/172.xx.236.128-26
/calico/ipam/v2/assignment/ipv4/block/172.xx.73.64-26

/calico/ipam/v2/host/k8s01/ipv4/block/172.xx.73.64-26
/calico/ipam/v2/host/k8s02/ipv4/block/172.xx.236.128-26
```
可以看出对应主机K8s01，分配的网段为172.xx.73.64/26
k8s02 为172.xx.236.128/26
这个网段和前面的主机route表也是匹配的

在看一下assignment的信息
```
 calico get /calico/ipam/v2/assignment/ipv4/block/172.xx.236.128-26
/calico/ipam/v2/assignment/ipv4/block/172.10.236.128-26
{"cidr":"111.xx.236.128/26","affinity":"host:k8s02","strictAffinity":false,"allocations":[null,null,null,null,null,null,null,null,0,1,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null],"unallocated":[10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,1,0,2,3,4,5,6,7],"attributes":[{"handle_id":"k8s-pod-network.e908404f358d0e88231393cf2e19b0328f05ccc162743e18a87e0192beef0e0d","secondary":null},{"handle_id":"k8s-pod-network.c4367bf4e8732d6dc33ed192f9ee9ce07f9e6731a98e1fca58cd2d1c1b887c5c","secondary":null}]}
```
- allocations:表示已经分配的
- unallocated：表示未分配的
- calico分配规则是一次分配一个64个ip的网段，如果不够，calico会持续分配。这样一个物理机就可以查到多个网络的分配情况。

备注：如果有异常操作，会导致etcd的allocations的字段记录的信息可能不准，但是unallocated记录的是准的。

ip段定义好后，对应主机上的容器在创建时，会自动的从该ip pool中分配ip。

# 容器路由
进入一个容器内部，比如
```
/ # ip route
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0 scope link 

/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if24: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1440 qdisc noqueue state UP 
    link/ether 3e:4d:b3:31:41:71 brd ff:ff:ff:ff:ff:ff
    inet 172.xx.236.129 /32 scope global eth0
       valid_lft forever preferred_lft forever

/ # ip neigh
169.254.1.1 dev eth0 lladdr ee:ee:ee:ee:ee:ee ref 1 used 0/0/0 probes 1 REACHABLE

```
可以看出：
- 本地路由统一指向 169.254.1.1。所有的容器都是这样的
- apr 地址统一为ee:ee:ee:ee:ee:ee
为啥是这样，参考：

https://docs.projectcalico.org/v3.5/usage/troubleshooting/faq#why-do-all-cali-interfaces-have-the-mac-address-eeeeeeeeeeee

我们看一下物理机的ifconfig：接口mac地址统一为 ee:ee:ee:ee:ee:ee
```
# ifconfig
cali5a5991829b4: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether ee:ee:ee:ee:ee:ee  txqueuelen 0  (Ethernet)
        RX packets 543  bytes 82503 (80.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 548  bytes 47602 (46.4 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

那么，如何查询容器和虚拟网卡的对应关系呢？
- 方法一：可以通过calicoctl指令查询
```
# calicoctl get workloadEndpoint
WORKLOAD                    NODE    NETWORKS            INTERFACE
my-nginx-79c95d84d4-dljtn   k8s01   172.xx.73.65/32     cali5a5991829b4
my-nginx-79c95d84d4-2jbcl   k8s02   172.xx.236.129/32   calif682b25fe10

```
- 方法二：登录到容器所在的物理机支持路由表。然后结合kubectl get pod -o wide 查询容器ip，结合起来查询。
```
# ip route
172.xx.73.65 dev cali5a5991829b4 scope link
```

# 总结
- 可以通过calicoctl指令查询容器和cali+ 虚拟网卡的对应关系
- 可以通过查询物理机路由表，找到本机容器的路由信息，以及其他物理机容器ip段的路由关系
- 物理机容器的ip段由calico在etcd中定义和分配的。
