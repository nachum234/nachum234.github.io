---
layout: post
title: "Configure Time Zone Using Shell Commands"
date: 2012-02-27
---

## Tested on
OS: CentOS 6.2
Hardware: Virtual Machine (VirtualBox 4.1.8)

## Configure Time Zone Using Shell Commands

* Configure /etc/sysconfig/clock file to the requested time zone, all time zones can be find in /usr/share/zoneinfo

```
vi /etc/sysconfig/clock
```

```
ZONE="Israel"
```
* Remove existing localtime file and creating new localtime symbolic link to the requested time zone file

```
rm -f /etc/localtime
ln -s /usr/share/zoneinfo/Israel /etc/localtime
```

For more information on time, date and time zone configuration please check the following: <http://www.vanemery.com/Linux/RH-Linux-Time.html>
