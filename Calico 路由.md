Calico 部署完成后，会生成对应的路由表。
# 主机路由表
登录一个物理机，输入 route 指令可以看到：
```javascript
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
` - name: CALICO_IPV4POOL_IPIP `
`   value: "off" `

或者CrossSubnet，则route表变为类似内容：

```javascript
# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
111.xx.73.69    0.0.0.0         255.255.255.255 UH    0      0        0 cali8eb4bab48a8
111.xx.236.128  k8s02           255.255.255.192 UG    0      0        0 enp0s8
```
则Use Iface部分变为主机的物理网卡名称

# calico IP分配规则
从上面可以看出，针对不同的物理机，calico分别不同的ip路由规则。这些物理机和ip网段的对应规则怎么可以查到呢？答案在etcd中。
查询etcd
etcdctl xx  get /  --prefix --keys-only |grep calico
xx为etcd key和秘钥相关信息
查询信息类似于：
/calico/ipam/v2/assignment/ipv4/block/111.xx.236.128-26
/calico/ipam/v2/assignment/ipv4/block/111.xx.73.64-26

/calico/ipam/v2/host/k8s01/ipv4/block/111.xx.73.64-26
/calico/ipam/v2/host/k8s02/ipv4/block/111.xx.236.128-26
可以看出对应主机K8s01，分配的网段为111.xx.73.64/26
k8s02 为111.xx.236.128/26
这个网段和前面的主机route表也是匹配的

在看一下assignment的信息
```javascript
 calico get /calico/ipam/v2/assignment/ipv4/block/111.xx.236.128-26
/calico/ipam/v2/assignment/ipv4/block/111.10.236.128-26
{"cidr":"111.xx.236.128/26","affinity":"host:k8s02","strictAffinity":false,"allocations":[null,null,null,null,null,null,null,null,0,1,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null],"unallocated":[10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,1,0,2,3,4,5,6,7],"attributes":[{"handle_id":"k8s-pod-network.e908404f358d0e88231393cf2e19b0328f05ccc162743e18a87e0192beef0e0d","secondary":null},{"handle_id":"k8s-pod-network.c4367bf4e8732d6dc33ed192f9ee9ce07f9e6731a98e1fca58cd2d1c1b887c5c","secondary":null}]}
```
allocations:表示已经分配的
unallocated：表示未分配的
calico分配规则是一次分配一个64个ip的网段，如果不够，calico会持续分配。这样一个物理机就可以查到多个网络的分配情况。
备注：如果有异常操作，会导致etcd的allocations的字段记录的信息可能不准，但是unallocated记录的是准的。
