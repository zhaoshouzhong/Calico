Calico 会生成大量的iptable，用来进行网络策略的控制。iptables是个坑，大量的iptable导致故障定位和排查复杂度大大增加。
因此，iptables也必须掌握。
掌握calico的iptables生成规则，必须具备iptables的基础知识和技能。下面这个图特别提一下，因为我们分析calico iptables生成规则时，需要重点参考这个图：
# iptables 链和表

![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/iptables.png)

# PREROUTING 链：
该链为入口链，有三张表，raw表，mangle表，nat表。
calico也在这三张表中生成一系列规则
## PREROUTING@raw 规则

## PREROUTING@mangle 规则

## PREROUTING@nat 规则

