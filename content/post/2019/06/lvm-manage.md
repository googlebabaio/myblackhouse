---
title: "记录一下用lvm的步骤"
subtitile: "主要用作挂载目录方便扩展"
date: 2019-06-10
draft: false
tags: ["linux"]
---

经常使用lvm进行目录的扩展,记录一下相关命令.
<!--more-->

## 1.创建pv
```
lvm> pvcreate /dev/vdb
  Physical volume "/dev/vdb" successfully created.
```

## 2.创建vg
```
lvm> vgcreate vgdata
  No command with matching syntax recognised.  Run 'vgcreate --help' for more information.
  Correct command syntax is:
  vgcreate VG_new PV ...

lvm> vgcreate vgdata /dev/vdb
  Volume group "vgdata" successfully created
lvm> vgdisplay
  --- Volume group ---
  VG Name               vgdata
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <350.00 GiB
  PE Size               4.00 MiB
  Total PE              89599
  Alloc PE / Size       0 / 0
  Free  PE / Size       89599 / <350.00 GiB
  VG UUID               Et2eCf-amVd-wDln-6NK1-IDza-YcnC-8XC2Dk

  --- Volume group ---
  VG Name               centos
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <49.00 GiB
  PE Size               4.00 MiB
  Total PE              12543
  Alloc PE / Size       12543 / <49.00 GiB
  Free  PE / Size       0 / 0
  VG UUID               3sPMh3-cxGR-wlD3-AhL4-uGfZ-HmCj-kmrPwU
```

## 3.创建lv
```
lvm> lvcreate -L 100G -n dockerdir vgdata
  Logical volume "dockerdir" created.
lvm> lvcreate -L 100G -n harbordir  vgdata
  Logical volume "harbordir" created.
lvm> lvcreate -L 120G -n nfsdir  vgdata
  Logical volume "nfsdir" created.

lvm> lvdisplay vgdata
  --- Logical volume ---
  LV Path                /dev/vgdata/dockerdir
  LV Name                dockerdir
  VG Name                vgdata
  LV UUID                NxYNDV-TS7Q-igO3-um3Q-ByHk-GZv2-tomFS0
  LV Write Access        read/write
  LV Creation host, time host-192-168-3-6, 2019-05-27 11:17:48 +0800
  LV Status              available
  # open                 0
  LV Size                100.00 GiB
  Current LE             25600
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2

  --- Logical volume ---
  LV Path                /dev/vgdata/harbordir
  LV Name                harbordir
  VG Name                vgdata
  LV UUID                m1lNfZ-QYqv-Lo1n-hTSu-WBy9-yAuv-fGsfNA
  LV Write Access        read/write
  LV Creation host, time host-192-168-3-6, 2019-05-27 11:18:29 +0800
  LV Status              available
  # open                 0
  LV Size                100.00 GiB
  Current LE             25600
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:3

  --- Logical volume ---
  LV Path                /dev/vgdata/nfsdir
  LV Name                nfsdir
  VG Name                vgdata
  LV UUID                D02GtI-MPdw-Tzuz-tKmD-1TTc-pPOo-j4vGRk
  LV Write Access        read/write
  LV Creation host, time host-192-168-3-6, 2019-05-27 11:18:47 +0800
  LV Status              available
  # open                 0
  LV Size                120.00 GiB
  Current LE             30720
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:4

lvm> exit
  Exiting.
```

## 4.格式化磁盘
```
mkfs.ext4 /dev/vgdata/dockerdir
mkfs.ext4 /dev/vgdata/nfsdir
mkfs.ext4 /dev/vgdata/harbordir
```

## 5.创建挂载点,并挂载
```
[root@host-192-168-3-6 ~]# mkdir /nfsdata
[root@host-192-168-3-6 ~]# mkdir /harbordata
[root@host-192-168-3-6 ~]# mkdir /dockerdata
[root@host-192-168-3-6 ~]# vim /etc/fstab

[root@host-192-168-3-6 ~]# cat /etc/fstab
#
# /etc/fstab
# Created by anaconda on Fri Apr 26 16:10:55 2019
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=4ec1bdd6-fcd8-48c5-ab4e-f905e9e4df27 /boot                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
/dev/vgdata/dockerdir /dockerdata ext4 defaults 0 0
/dev/vgdata/nfsdir /nfsdata ext4 defaults 0 0
/dev/vgdata/harbordir /harbordata ext4 defaults 0 0


[root@host-192-168-3-6 ~]# mount -a
[root@host-192-168-3-6 ~]# df -h
文件系统                      容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root        44G  1.3G   43G    3% /
devtmpfs                       16G     0   16G    0% /dev
tmpfs                          16G     0   16G    0% /dev/shm
tmpfs                          16G  8.5M   16G    1% /run
tmpfs                          16G     0   16G    0% /sys/fs/cgroup
/dev/vda1                    1014M  142M  873M   14% /boot
/dev/mapper/vgdata-dockerdir   99G   61M   94G    1% /dockerdata
/dev/mapper/vgdata-nfsdir     118G   61M  112G    1% /nfsdata
/dev/mapper/vgdata-harbordir   99G   61M   94G    1% /harbordata
```
