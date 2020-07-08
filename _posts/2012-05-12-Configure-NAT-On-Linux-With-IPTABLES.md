---
layout: post
title: "Configure NAT On Linux With IPTABLES"
date: 2012-05-12
---

## Tested On
OS: CentOS 6.2 i386  
iptables version: v1.4.7  
Hardware: Virtual Machine (VirtualBox 4.1.14)  

## About

Network Address Translation (NAT) is a technology that translate private addresses to public and vice versa. In this guide I will show how to implement the main types of NAT using linux and iptables.

## Network Address Port Translation (NAPT)/Port Address Translation (PAT)

NAPT is the most common type of NAT. This type of NAT on a traditional outbound transaction change the source IP and the source port, and because it change also the source port multiple devices can share the same IP simultaneously.

* Add the following rule to the NAT table of iptables in order to configure NAPT

```
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 192.168.10.1
```

```
-t nat - Use NAT table of iptables
-A POSTROUTING - Append the rule to POSTROUTING chain on the NAT table
-o interface - Specify on which outgoing interface apply this rule
-j SNAT - Change the source address
--to-source - Source addresses list to change the original source address
```

* If you have a dynamic IP use the following rule

```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

## Traditional/Outbound NAT

Traditional NAT share public IP addresses with local devices that use private IP addresses.

Traditional NAT is implemented in iptables like NAPT with multiple source IP addresses.

* Add the following rule to the NAT table of iptables in order to configure traditional NAT

```
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 192.168.10.1-192.168.10.20
```

## Bidirectional/Inbound NAT

Bidirectional NAT is used when a device from the outside network needs to initiate a session with server on the inside network.

* Add the following rule to the NAT table of iptables in order to configure bidirectional NAT

```
iptables -t nat -A PREROUTING -i eth0 -j DNAT --to-destination 192.168.10.7
```

* If the NAT gateway is the only server how can access to the inside server then you may need to enter this rule also to make sure he will send the traffic using the  NAT gateway

```
iptables -t nat -A POSTROUTING -d 192.168.10.7/32 -j MASQUERADE
```

```
-i interface - Name of an interface via a packet was received
--to-destination local_ip_address - IP address of a local server
```

* If you want to configure a specific public address use the following rule

```
iptables -t nat -A PREROUTING -d 46.116.47.237 -j DNAT --to-destination 192.168.10.7
```

* If you want to DNAT only a single port use the following

```
iptables -t nat -A PREROUTING -p tcp -d 46.116.47.237 --dport 80 -j DNAT --to-destination 192.168.10.7
```
