# Vmware虚拟机中的docker容器配置IP桥接主机物理网络

- 拓扑结构

    外网 - 路由器（dhcp） - 宿主机（windows10） - 虚拟机（vmware/centos7） - Docker容器（ubuntu）

- VMWARE的虚拟机桥接到宿主机

    采用桥接模式将虚拟机直连到宿主机的物理网络，此时在虚拟机内部可以看到虚拟机ip地址和宿主机处于同一个网段，虚拟网卡：ens33。

- 在虚拟机内安装并启动docker，docker创建默认bridge虚拟网卡：docker0。

- 创建一个虚拟网桥：br0，并将虚拟网卡ens33连接到这个虚拟网桥，虚拟网桥则采用ens33虚拟网卡的ip配置。

```text
# /etc/sysconfig/network-scripts/ifcfg-ens33
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
NAME="ens33"
UUID="867c88ba-d64d-4dbb-a05d-3a548e1e733a"
DEVICE="ens33"
ONBOOT="yes"
IPV6_PRIVACY="no"
BRIDGE="br0"
```
```text
# /etc/sysconfig/network-scripts/ifcfg-br0
DEVICE="br0"
TYPE="Bridge"
ONBOOT="yes"
BOOTPROTO="static"
IPADDR="192.168.31.159"
NETMASK="255.255.255.0"
GATEWAY="192.168.31.1"
DNS1="223.5.5.5"
DNS2="114.114.114.114"
```

- 重启网络使配置生效，如果配置失败，ssh会失去连接，此时可通过vmware管理程序操作还原或修复

```bash
systemctl restart network
```

- 安装pipework，并给容器配置IP地址

```bash
git clone https://github.com/jpetazzo/pipework
cp pipework/pipework /usr/local/bin/

# 网络模式使用none
docker run -itd --net=none --name test centos bash

# 使用pipework配置网络地址，注意不要和宿主机网络内的其他设备地址冲突即可
pipework br0 docker_bridge 192.168.31.159/24
```

- 配置完成后，虚拟机网络如下：

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP group default qlen 1000
    link/ether 00:0c:29:51:17:44 brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:de:85:77:fa brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
14: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:0c:29:51:17:44 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.159/24 brd 192.168.31.255 scope global noprefixroute br0
       valid_lft forever preferred_lft forever
    inet6 fe80::2890:5bff:fef5:b268/64 scope link 
       valid_lft forever preferred_lft forever
16: veth1pl50591@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP group default qlen 1000
    link/ether 86:f7:9b:39:d6:16 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::84f7:9bff:fe39:d616/64 scope link 
       valid_lft forever preferred_lft forever
```

- 容器内部网络如下：

```text
eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.31.160  netmask 255.255.255.0  broadcast 192.168.31.255
        ether a6:4b:42:1a:3e:49  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
```