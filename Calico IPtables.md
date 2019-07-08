Calico 会生成大量的iptable，用来进行网络策略的控制。iptables是个坑，大量的iptable导致故障定位和排查复杂度大大增加。
掌握calico的iptables生成规则，必须具备iptables的基础知识和技能。下面这个图特别提一下，因为我们分析calico iptables生成规则时，需要重点参考这个图：
# iptables 链和表

![image](https://github.com/zhaoshouzhong/Calico/raw/master/images/iptables.png)

# PREROUTING 链：
该链为入口链，有三张表，raw表，mangle表，nat表。
calico也在这三张表中生成一系列规则。
输入，iptables-save，我们可以得到完整的iptables。 
## PREROUTING@raw 规则
```
*raw
:PREROUTING ACCEPT [3849262:930150343]
:OUTPUT ACCEPT [3499061:891217635]
:cali-OUTPUT - [0:0]
:cali-PREROUTING - [0:0]
:cali-failsafe-in - [0:0]
:cali-failsafe-out - [0:0]
:cali-from-host-endpoint - [0:0]
:cali-to-host-endpoint - [0:0]
-A PREROUTING -m comment --comment "cali:6gwbT8clXdHdC1b1" -j cali-PREROUTING
---
-A cali-PREROUTING -m comment --comment "cali:XFX5xbM8B9qR10JG" -j MARK --set-xmark 0x0/0xf0000
-A cali-PREROUTING -i cali+ -m comment --comment "cali:EWMPb0zVROM-woQp" -j MARK --set-xmark 0x40000/0x40000
-A cali-PREROUTING -m comment --comment "cali:Ek_rsNpunyDlK3sH" -m mark --mark 0x0/0x40000 -j cali-from-host-endpoint
-A cali-PREROUTING -m comment --comment "cali:nM-DzTFPwQbQvtRj" -m mark --mark 0x10000/0x10000 -j ACCEPT
```
cali-PREROUTING:
- 第一条：设置标记0x0/0xf0000
- 第二条：对所有从 cali+ 的网卡，设置标记 0x40000/0x40000
- 第三条：对其他非 cali+ 流量（匹配0x0/0x40000）的，调整到 cali-from-host-endpoint 规则处理。cali-from-host-endpoint目前内容为空
- 第四条：允许标记为  0x10000/0x10000的消息通过

## PREROUTING@mangle 规则
```
*mangle
:PREROUTING ACCEPT [29734:1775653]
:INPUT ACCEPT [3440903:727101783]
:FORWARD ACCEPT [401054:202610260]
:OUTPUT ACCEPT [3499061:891217635]
:POSTROUTING ACCEPT [3896368:1093548003]
:cali-PREROUTING - [0:0]
:cali-failsafe-in - [0:0]
:cali-from-host-endpoint - [0:0]
-A PREROUTING -m comment --comment "cali:6gwbT8clXdHdC1b1" -j cali-PREROUTING
-A cali-PREROUTING -m comment --comment "cali:6BJqBjBC7crtA-7-" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A cali-PREROUTING -m comment --comment "cali:KX7AGNd6rMcDUai6" -m mark --mark 0x10000/0x10000 -j ACCEPT
-A cali-PREROUTING -i cali+ -m comment --comment "cali:i3igoQZv8mRXgdz5" -j ACCEPT
-A cali-PREROUTING -m comment --comment "cali:lgSUN0vEjQ4dIHbp" -j cali-from-host-endpoint
-A cali-PREROUTING -m comment --comment "cali:iqLD7MJ2v-mVyDd4" -m comment --comment "Host endpoint policy accepted packet." -m mark --mark 0x10000/0x10000 -j ACCEPT
```
 cali-PREROUTING :
- 第一条规则：允许连接状态为 RELATED,ESTABLISHED 的通过
- 第二条规则：允许 mark为 0x10000/0x10000 的通过
- 第三条规则：所有从cali+的流量允许通过
- 第四条规则：跳转到 cali-from-host-endpoint。目前cali-from-host-endpoint 规则为空
- 第五条规则：允许 mark为 0x10000/0x10000 的通过
 
## PREROUTING@nat 规则
```
*nat
:PREROUTING ACCEPT [23:1380]
:INPUT ACCEPT [11:660]
:OUTPUT ACCEPT [8:496]
:POSTROUTING ACCEPT [12:736]
:DOCKER - [0:0]
:KUBE-FIREWALL - [0:0]
:KUBE-LOAD-BALANCER - [0:0]
:KUBE-MARK-DROP - [0:0]
:KUBE-MARK-MASQ - [0:0]
:KUBE-NODE-PORT - [0:0]
:KUBE-POSTROUTING - [0:0]
:KUBE-SERVICES - [0:0]
:cali-OUTPUT - [0:0]
:cali-POSTROUTING - [0:0]
:cali-PREROUTING - [0:0]
:cali-fip-dnat - [0:0]
:cali-fip-snat - [0:0]
:cali-nat-outgoing - [0:0]
-A PREROUTING -m comment --comment "cali:6gwbT8clXdHdC1b1" -j cali-PREROUTING
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER

---
-A cali-PREROUTING -m comment --comment "cali:r6XmIziWUJsdOK6Z" -j cali-fip-dnat

```
规则：
- 第一条：跳转到 cali-PREROUTING，而 cali-PREROUTING 跳转到 cali-fip-dnat，目前cali-fip-dnat 为空
- 第二条：跳转到 KUBE-SERVICES，这个是K8s生成的规则
- 第三条：跳转到 DOCKER ，这个是Docker生成的规则

参考iptables的链和表关系，可以逐步梳理出其他的iptables信息。不再详述。
