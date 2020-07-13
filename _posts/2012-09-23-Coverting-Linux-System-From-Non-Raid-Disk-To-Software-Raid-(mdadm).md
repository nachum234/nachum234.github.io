---
layout: default
title: "Coverting Linux System From Non Raid Disk To Software Raid (mdadm)"
date: 2012-09-23
---

## Tested On
OS: CentOS 6.3 i386  
mdadm version: v3.2.3  
Hardware: Virtual Machine (VirtualBox 4.1.22)  

## Introduction

mdadm is a linux software raid implementation. With mdadm you can build software raid from different level on your linux server, in this post I will show how to convert an existing OS system to raid 1 for fault tolerance.

## Pre conversion system

* Partition sda1 for /boot and one logical volume for /

```
df -h
```

```
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_root
                      2.5G  812M  1.6G  34% /
tmpfs                 250M     0  250M   0% /dev/shm
/dev/sda1             485M   85M  375M  19% /boot
```

## Converting to raid 1

* Insert new disk to the system and create linux raid partition on it using fdisk

```
fdisk /dev/sdb

Command (m for help): n -> **enter**
Command action
   e   extended
   p   primary partition (1-4)
p -> enter
Partition number (1-4): 1 -> **enter**
First cylinder (1-1044, default 1): **enter**
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-1044, default 1044): +3G **enter**

Command (m for help): **t**
Selected partition 1
Hex code (type L to list codes): **fd**
Changed system type of partition 1 to fd (Linux raid autodetect)

Command (m for help): **a**
Partition number (1-4): **1**

Command (m for help): **w**
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

* create new raid 1 in a degraded mode using one disk (sdb1)

```
yum install mdadm -y
mdadm --create /dev/md0 --level raid1 --raid-disks 2 missing /dev/sdb1 --metadata 1.0
echo "MAILADDR root@localhost" >> /etc/mdadm.conf
mdadm --detail --scan >> /etc/mdadm.conf
```

* Create new file system on the new raid device and copy the data from the original disk to the raid

```
mkfs.ext4 /dev/md0
mount /dev/md0 /mnt
cd /
find . -xdev | cpio -pm /mnt/
cd /boot
find . -xdev | cpio -pm /mnt/boot/
```

* Change /etc/fstab on the raid device

```
vi /mnt/etc/fstab
```

```
/dev/md0        /                       ext4    defaults        1 1
tmpfs           /dev/shm                tmpfs   defaults        0 0
devpts          /dev/pts                devpts  gid=5,mode=620  0 0
sysfs           /sys                    sysfs   defaults        0 0
proc            /proc                   proc    defaults        0 0
```

* Here is the old fstab file

```
cat /etc/fstab
/dev/mapper/VolGroup-lv_root                    /                       ext4    defaults        1 1
UUID=d898d82b-bb81-4ff5-9261-681b66bc9c7d       /boot                   ext4    defaults        1 2
/dev/mapper/VolGroup-lv_swap                    swap                    swap    defaults        0 0
tmpfs                                           /dev/shm                tmpfs   defaults        0 0
devpts                                          /dev/pts                devpts  gid=5,mode=620  0 0
sysfs                                           /sys                    sysfs   defaults        0 0
proc                                            /proc                   proc    defaults        0 0
```

* Setup grub on /dev/sdb

```
grub
grub> device (hd0) /dev/sdb
grub> root (hd0,0)
grub> setup (hd0)
grub> quit
```

* Change grub.conf file on the new raid device

```
vi /mnt/boot/grub/grub.conf
```

```
default=0
timeout=5
splashimage=(hd0,0)/boot/grub/splash.xpm.gz
hiddenmenu
title CentOS (2.6.32-279.5.2.el6.i686)
        root (hd0,0)
        kernel /boot/vmlinuz-2.6.32-279.5.2.el6.i686 ro root=/dev/md0 rd_NO_LUKS LANG=en_US.UTF-8 SYSFONT=latarcyrheb-sun16 rhgb crashkernel=auto quiet KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM
        initrd /boot/initramfs-2.6.32-279.5.2.el6.i686.img
```

* I add the /boot prefix because I remove the /boot partition. here is the old grub.conf file

```
cat /boot/grub/grub.conf
default=0
timeout=5
splashimage=(hd0,0)/grub/splash.xpm.gz
hiddenmenu
title CentOS (2.6.32-279.5.2.el6.i686)
        root (hd0,0)
        kernel /vmlinuz-2.6.32-279.5.2.el6.i686 ro root=/dev/mapper/VolGroup-lv_root rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD rd_LVM_LV=VolGroup/lv_swap SYSFONT=latarcyrheb-sun16 rhgb crashkernel=auto quiet rd_LVM_LV=VolGroup/lv_root  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM
        initrd /initramfs-2.6.32-279.5.2.el6.i686.img
```

* Create new initrd file with raid 1 support

```
cd /mnt/boot
mkinitrd --preload raid1 --with=raid1 raid-ramdisk 2.6.32-279.5.2.el6.i686
mv initramfs-2.6.32-279.5.2.el6.i686.img initramfs-2.6.32-279.5.2.el6.i686.img.bck
mv raid-ramdisk initramfs-2.6.32-279.5.2.el6.i686.img
```

* Reboot the system from the new drive (sdb)

```
reboot
```

* After we mounted the root folder from the raid device we can destroy the original disk and prepare it to the raid

```
fdisk /dev/sda

Command (m for help): d -> enter
Partition number (1-4): 2 -> enter

Command (m for help): d -> enter
Selected partition 1

Command (m for help): n -> enter
Command action
   e   extended
   p   primary partition (1-4)
p -> enter
Partition number (1-4): 1 -> enter
First cylinder (1-522, default 1): enter
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-522, default 522): +3G -> enter

Command (m for help): t -> enter
Selected partition 1
Hex code (type L to list codes): fd -> enter
Changed system type of partition 1 to fd (Linux raid autodetect)

Command (m for help): a -> enter
Partition number (1-4): 1 -> enter

Command (m for help): w -> enter
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

* Reread the partition table of sda using partprobe

```
yum install parted -y
partprobe /dev/sda
Add /dev/sda1 to the array
mdadm /dev/md0 --add /dev/sda1
```

* Wait for the rebuild process to complete. check the status using /proc/mdstat file

```
cat /proc/mdstat
```

* Setup grub on /dev/sda

```
grub
grub> device (hd0) /dev/sda
grub> root (hd0,0)
grub> setup (hd0)
grub> quit
```

* Reboot to check that the system come up from sda

```
reboot
```

# Testing Our New Raid 1

* Check the array status

```
mdadm --detail /dev/md0
```

* Simulate disk sda1 failure

```
mdadm --manage --set-faulty /dev/md0 /dev/sda1
```

* Check syslog for new failure messages

```
tail /var/log/messges

Sep 23 21:27:36 centos-62-1 kernel: md/raid1:md0: Disk failure on sda1, disabling device.
Sep 23 21:27:36 centos-62-1 kernel: md/raid1:md0: Operation continuing on 1 devices.
```

* Check array status

```
mdadm --detail /dev/md0
cat /proc/mdstat
```

* Remove sda1 from the array and re-add it

```
mdadm /dev/md0 -r /dev/sda1
mdadm /dev/md0 -a /dev/sda1
```

* Check array status

```
mdadm --detail /dev/md0
cat /proc/mdstat
```

Please visit <https://raid.wiki.kernel.org> for more information about linux software raid configuration and usage.
