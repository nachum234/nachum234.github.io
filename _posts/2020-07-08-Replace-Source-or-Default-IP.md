---
layout: post
title: "Replace Source/Default IP"
date: 2020-07-08
---

## Tested On
OS: CentOS Linux release 7.7.1908 (Core)

If you have more than one IP assign to your server and you want to change your default outgoing IP you can do the following

* Configure new IP in new network interface alias (don't forget to change to the correct IP and interface name)

```
ifconfig eno1:0 1.2.3.4 netmask 255.255.255.255 up
```

* to make the change permanent you can create the following file

```
vi /etc/sysconfig/network-scripts/ifcfg-eno1:0
```

```
NAME="eno1:0"
DEVICE="eno1:0"
ONBOOT="yes"
TYPE="Ethernet"
IPADDR="1.2.3.4"
NETMASK=255.255.255.255
```

* get current default gateway

```
netstat -rn

Kernel IP routing table
Destination       Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0           213.202.233.1   0.0.0.0         UG        0 0          0 eno1
172.17.0.0        0.0.0.0         255.255.0.0     U         0 0          0 docker0
213.202.233.1     0.0.0.0         255.255.255.255 UH        0 0          0 eno1
213.202.233.180   0.0.0.0         255.255.255.255 UH        0 0          0 eno1
```

* change the source address on OS default route

```
ip route replace default via 213.202.233.1 src 89.163.130.118
```

* check new default IP

```
curl ipinfo.io
```
