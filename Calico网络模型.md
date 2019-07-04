# 概述
calico 针对不同的应用场景，有不同的网络模型，需要分为两大类：
- 1：二层网络模型：适合于二层组网和互通的情况，endpoint位于通过二层交换机互通，不需要通过额外路由。
- 2：三层网络模型：如果需要跨越多个子网，则需要三层交换机进行路由交换，实现跨子网通信。
下面分别介绍这两种网络模型：

# 二层网络模型：
https://docs.projectcalico.org/v3.7/networking/design/l2-interconnect-fabric

![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/l2-rr-spine-planes.png)

该模式介绍：
- 1： blue 、green 、 orange、red 代表4个网络平面，可以用4个不同vlan进行区分
- 2： 每个平面有独立的RR，RR之间不需要组成对等集群
- 3： 每个主机和主机上的endpoint构成一个自治域，这样同一主机的endpoint之间的通信，不需要经过TOR交换机。当然，只能访问同一vlan的。
- 4： 每个主机至少加入4个中的一个RR，也可以4个平面全部加入
- 5： RR 可以启用在物理机交换机，也可以是单独的软件系统

优点：
- 1： 有4个网络平面，网络的可靠性提高了。一个网络平面不通时，可以切换到另外一个网络平面
- 2： 直接二层网络通信，通信效率高
缺点：
- 1： 考虑到网络风暴影响，一个vlan网络的主机数量不能搞太大。
- 2： 仅适合于二层组网，不能满足复杂的组网要求

# 三层网络模型

https://docs.projectcalico.org/v3.7/networking/design/l3-interconnect-fabric

三层网络模型，有多种方案可供选择
## The AS Per Rack model（每个机架一个自治域）
这个是calico官方推荐的组网方案，
### 二层Peer
![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/l3-fabric-diagrams-as-rack-l2-spine.png)
该模式介绍：
- 每个机架一个RR
- 机架上的所有主机和机架RR构成Peer集群
- 机架RR和其他机架RR构成Peer集群

优点：
- 1: 网络模型比较清晰和明了，网络路径短。

缺点：
- 2：如果机架数量较多，则机架RR的之间连接数也会指数型增加，不适合较大规模组网

### 三层Peer
![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/l3-fabric-diagrams-as-rack-l3-spine.png)
模式介绍：
- 1：每个机架一个RR
- 2：机架上的所有主机和机架RR构成Peer集群
- 3：机架RR和核心交换机的RR组成Peer集群

优点：
- 1： 可以适合较大规模网络组网
- 2:  可扩展性好

缺点：
- 1: 需要注意集群规模，防止超过路由表最大数量

## The AS per Compute Server model(每台主机一个AS自治域)
calico官方不推荐的方案，缺点比较明显：
- 1： 占用大量AS号
- 2： 维护和配置较为复杂

虽然不推荐，还是要介绍一下：
### 二层Peer
![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/l3-fabric-diagrams-as-server-l2-spine.png)
模式介绍：
- 1：每台主机是一个独立的AS域
- 2：每个机架一个独立的AS域
- 3: 主机和所在机架的AS组成对等Peer集群，交换机机架内的路由信息
- 4：机架的AS域之间组成BGP Peer集群，交换机架间的路由信息。

### 三层Peer
![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/l3-fabric-diagrams-as-server-l3-spine.png)
模型介绍：
相对于前面的模式，它的不同在于：
机架AS域和核心交换机的AS域组成BGP Peer集群

## The Downward Default model（缺省向下模式）
![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/l3-fabric-downward-default.png)

为什么采用这种模式呢，主要是以上模式存在一些问题：
- 1： 每个node、每个TOR交换机、每个核心交换机都需要记录全网路由。这样会导致路由表数量激增

为了减少路由表的数量，可以考虑采用该模式，该模式的特点如下：
- 1: 所有的node 使用相同的AS号，但是A1,A2可以是不同的网络平面
- 2：所有的下级节点向上节点发布所有的路由信息，比如node-->TOR-->core交换机
- 3：上级节点向下级节点发布一个默认路由

优点：
- 1：每个点仅有自己所属的路由信息，路由数量极大减少了

缺点：
- 1: 路径长，通信效率不如前面的高
