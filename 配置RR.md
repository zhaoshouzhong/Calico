在规模比较大时，不能采用node-to-node mesh模型，必须引入RR。RR可以配置在交换机上，也可以以软件方式部署。
calico提供RR的能力。
# 部署RR
### 前置条件：
- 找一个主机，部署了docker。
- etcd 已经部署好

### 部署calicoctl
curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.7.4/calicoctl
chmod 755 calicoctl 
cp calicoctl /usr/bin

### 定义calicoctl的配置文件
定义calicoctl的配置文件，同时需要把etcd相关证书也复制到对应的目录下：
```
[root@calico calico]# cat /etc/calico/calicoctl.cfg
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "etcdv3"
  etcdEndpoints: "https://192.168.56.xxx:2379,https://https://192.168.56.xxx:2379"
  etcdKeyFile: "/etc/etcd/cert/etcd-key.pem"
  etcdCertFile: "/etc/etcd/cert/etcd.pem"
  etcdCACertFile: "/etc/kubernetes/cert/ca.pem"
```
### 启动一个calico node
calicoctl node run --name=calico-rr --node-image=calico/node:v3.7.3 --as=64512 --ip=192.168.56.40

运行后，可以看到类似如下信息：
```
[root@calico calico]# calicoctl node run --name=calico-rr --node-image=calico/node:v3.7.3 --as=64512 --ip=192.168.56.40
Running command to load modules: modprobe -a xt_set ip6_tables
Enabling IPv4 forwarding
Enabling IPv6 forwarding
Increasing conntrack limit
Removing old calico-node container (if running).
Running the following command to start calico-node:

docker run --net=host --privileged --name=calico-node -d --restart=always -e IP=192.168.56.40 -e ETCD_ENDPOINTS=https://192.168.56.xxx:2379,https://https://192.168.56.xxx:2379 -e ETCD_CA_CERT_FILE=/etc/calico/certs/ca_cert.crt -e NODENAME=calico-rr -e CALICO_LIBNETWORK_ENABLED=true -e ETCD_KEY_FILE=/etc/calico/certs/key.pem -e ETCD_CERT_FILE=/etc/calico/certs/cert.crt -e CALICO_NETWORKING_BACKEND=bird -e AS=64512 -v /var/log/calico:/var/log/calico -v /var/run/calico:/var/run/calico -v /var/lib/calico:/var/lib/calico -v /lib/modules:/lib/modules -v /run:/run -v /run/docker/plugins:/run/docker/plugins -v /var/run/docker.sock:/var/run/docker.sock -v /etc/kubernetes/cert/ca.pem:/etc/calico/certs/ca_cert.crt -v /etc/etcd/cert/etcd-key.pem:/etc/calico/certs/key.pem -v /etc/etcd/cert/etcd.pem:/etc/calico/certs/cert.crt calico/node:v3.7.3

Image may take a short time to download if it is not available locally.
Container started, checking progress logs.

2019-07-09 03:07:29.533 [INFO][8] startup.go 256: Early log level set to info
2019-07-09 03:07:29.533 [INFO][8] startup.go 272: Using NODENAME environment for node name
2019-07-09 03:07:29.533 [INFO][8] startup.go 284: Determined node name: calico-rr
2019-07-09 03:07:29.562 [INFO][8] startup.go 97: Skipping datastore connection test
2019-07-09 03:07:29.571 [INFO][8] startup.go 367: Building new node resource Name="calico-rr"
2019-07-09 03:07:29.571 [INFO][8] startup.go 382: Initialize BGP data
2019-07-09 03:07:29.571 [INFO][8] startup.go 476: Using IPv4 address from environment: IP=192.168.56.40
2019-07-09 03:07:29.572 [INFO][8] startup.go 509: IPv4 address 192.168.56.40 discovered on interface enp0s8
2019-07-09 03:07:29.572 [INFO][8] startup.go 452: Node IPv4 changed, will check for conflicts
2019-07-09 03:07:29.578 [INFO][8] startup.go 642: Using AS number specified in environment (AS=64512)
2019-07-09 03:07:29.591 [INFO][8] startup.go 536: FELIX_IPV6SUPPORT is true (defaulted) through environment variable
2019-07-09 03:07:29.591 [INFO][8] startup_linux.go 79: IPv6 supported on this platform: true
2019-07-09 03:07:29.591 [INFO][8] startup.go 536: CALICO_IPV6POOL_NAT_OUTGOING is false (defaulted) through environment variable
2019-07-09 03:07:29.591 [INFO][8] startup.go 796: Ensure default IPv6 pool is created. IPIP mode: Never, VXLAN mode: Never
2019-07-09 03:07:29.613 [INFO][8] startup.go 806: Created default IPv6 pool (fd6f:30e:5e12::/48) with NAT outgoing false. IPIP mode: Never, VXLAN mode: Never
2019-07-09 03:07:29.628 [INFO][8] startup.go 181: Using node name: calico-rr
Starting libnetwork service
Calico node started successfully

```

启动后，calicoctl get node 可以看到该node的信息
```
[root@calico calico]# calicoctl get node
NAME
calico-rr

```
### 设置calico-rr为RR
calicoctl get node calico-rr -o yaml
获取节点的yaml文件，然后复制下来。最终调整后的信息类似如下：
```
[root@k8s01 calico3.7]# cat node-calico-rr.yaml
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  name: calico-rr
spec:
  bgp:
    asNumber: 64512
    ipv4Address: 192.168.56.40/32
    routeReflectorClusterID: 192.168.56.40

```

calicoctl apply -f node-calico-rr.yaml

### 关闭node-to-node mesh模型
```
[root@k8s01 calico3.7]# cat BGPConfiguration.yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false
  asNumber: 64512
  ```
  
 calicoctl apply -f BGPConfiguration.yaml

### 设置全局bgp peer
```
[root@k8s01 calico3.7]# cat  BGPPeer.yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-40
spec:
  peerIP: 192.168.56.40
  asNumber: 64512

```

calicoctl apply -f BGPPeer.yaml

### 查看RR设置结果
查看node 状态：
```
[root@k8s01 calico3.7]# calicoctl  node status
Calico process is running.

IPv4 BGP status
+---------------+-----------+-------+----------+-------------+
| PEER ADDRESS  | PEER TYPE | STATE |  SINCE   |    INFO     |
+---------------+-----------+-------+----------+-------------+
| 192.168.56.40 | global    | up    | 10:33:38 | Established |
+---------------+-----------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```
查看route信息是否正确
```
[root@k8s01 calico3.7]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gateway         0.0.0.0         UG    102    0        0 enp0s3
10.0.2.0        0.0.0.0         255.255.255.0   U     102    0        0 enp0s3
10.100.73.64    0.0.0.0         255.255.255.255 UH    0      0        0 cali417a5adc43c
10.100.73.64    0.0.0.0         255.255.255.192 U     0      0        0 *
10.100.236.128  k8s02           255.255.255.192 UG    0      0        0 enp0s8

```
可以看到已经正确生成对其他主机的路由信息。
然后再ping一下pod ip，检查看看是否可以ping通。
如果没有问题，则表示RR配置成功。
