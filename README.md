# Calico介绍
calico 是一个开源的网络/网络安全解决方案，用在容器，虚拟机，物理机上。calico支持Kubernetes, OpenShift, Docker EE, OpenStack,  bare metal 等。
## Calico的优势：
- 1：提供了丰富的网络策略模型，实现了细粒度的网络安全控制。
- 2：高性能：基于bgp分发和交换路由信息，基于linux 内核的route和iptables机制进行网络通信。没有额外的封包和拆包动作，通信效率高。
- 3：可扩展性：可以平滑的支持大（1k+）中（几个节点）小（10个节点）各种网络模型
- 4：可操作性好：无法集成k8s,openstack等开源组件

劣势：
- 1：calico的缺点是路由的数目与容器数目相同，非常容易超过路由器、三层交换、甚至node的处理能力，从而限制了整个网络的扩张。
- 2：calico的每个node上会设置大量（海量)的iptables规则、路由，运维、排障难度大。
- 3：calico的原理决定了它不可能支持VPC，容器只能从calico设置的网段中获取ip。
- 4：calico目前的实现没有流量控制的功能，会出现少数容器抢占node多数带宽的情况。


参考资料：

https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/04/11/calico-usage.html

https://www.jianshu.com/p/f0177b84de66

https://feisky.gitbooks.io/sdn/container/calico/
