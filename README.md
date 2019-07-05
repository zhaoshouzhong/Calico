# Calico介绍
calico 是一个开源的网络/网络安全解决方案，用在容器，虚拟机，物理机上。calico支持Kubernetes, OpenShift, Docker EE, OpenStack,  bare metal 等。
## Calico的优势：
- 1：提供了丰富的网络策略模型，实现了细粒度的网络安全控制。
- 2：高性能：基于bgp分发和交换路由信息，基于linux 内核的route和iptables机制进行网络通信。没有额外的封包和拆包动作，通信效率高。
- 3：可扩展性：可以平滑的支持大（1k+）中（几百个节点）小（几十个节点）各种网络模型
- 4：可操作性好：无缝集成k8s,openstack等开源组件

劣势：
- 1：calico的每个node上会设置大量的iptables规则、路由，运维、排障难度大。

整体来说calico还是一款比较优秀的网络虚拟化系统。如果想要比较好的掌握calico，可以从如下几点入手：
- 1: calico架构：了解calico组件和部署架构
- 2: calico网络模型：了解几种常见的calico组网模型，以及每种模型的优劣。到时候结合自己公司的现状进行网络模型选择
- 3：calico路由：了解calico的route生成规则
- 4：calico iptables：了解calico 生成iptables的规则

参考资料：

https://docs.projectcalico.org/v3.7/reference/architecture/

https://docs.projectcalico.org/v3.7/networking/design/l3-interconnect-fabric

https://docs.projectcalico.org/v3.7/networking/design/l2-interconnect-fabric

https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/04/11/calico-usage.html

https://www.jianshu.com/p/f0177b84de66

https://feisky.gitbooks.io/sdn/container/calico/
