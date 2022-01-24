---
title: "瞎折腾：利用Nuc11打造Nas与个人影音中心"
description: 
date: 2022-01-17T16:35:05+08:00
tags: ["Nas", "Nuc", "Jellyfin", "Linux"]
categories: ["Techonlogy"]
image: 59474391_p0.jpeg
math: 
license: 
hidden: false
comments: true
draft: true
---

## 写在前面

为什么我会突然想搞一台Nas呢，我这么问自己，仔细回想一下大概是因为自从番剧需要进行审核才能上映的政策出台后，我就准备了一块1t的ssd接一块败家之眼的m2硬盘盒用来存自己下载的番剧，同时也方便和朋友分享；其次是去年阿b推出了放映室的功能，可以很方便的和朋友一同看番看电影，但这样终究是不方便，好在我本身就是个爱折腾的人，刚好前段时间给群友搞了一个ts服务器来让群友们吹水聊天以及游戏语音，我就开始想能不能通过自建的服务器也实现一个类似的流媒体平台，随时和朋友分享自己私藏的电影番剧。假如有这样一个场景：某天有小姐姐想看个电影但苦于没有资源

## 方案探索



## 安装配置

### 配置UPS

```
sudo apt install nut
```

[使用NUT解决BK650M2-CH失联问题（一） - 简书 (jianshu.com)](https://www.jianshu.com/p/bb3a52916d79)

### 配置邮件提醒

```
sudo apt install mailutils
```



### 磁盘分区 fdisk，parted和gdisk

fdisk:

```
➜  ~ sudo fdisk /dev/sda

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
The size of this disk is 3.7 TiB (4000787030016 bytes). DOS partition table format cannot be used on drives for volumes larger than 2199023255040 bytes for 512-byte sectors. Use GUID partition table format (GPT).

Created a new DOS disklabel with disk identifier 0x277fe383.
```

parted:

```
➜  ~ sudo parted /dev/sda
GNU Parted 3.3
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Error: /dev/sda: unrecognised disk label
Model: ST4000VN 008-2DR166 (scsi)
Disk /dev/sda: 4001GB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags:
(parted) mklabel gpt
(parted) print
Model: ST4000VN 008-2DR166 (scsi)
Disk /dev/sda: 4001GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start  End  Size  File system  Name  Flags

(parted) mkpart primary 0GB 4001GB
(parted) print
Model: ST4000VN 008-2DR166 (scsi)
Disk /dev/sda: 4001GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 1      33.6MB  4001GB  4001GB               primary

(parted) quit
Information: You may need to update /etc/fstab.
```

当使用mkfs的时候

```
➜  ~ sudo mkfs.ext4 /dev/sda1
mke2fs 1.45.5 (07-Jan-2020)
/dev/sda1 alignment is offset by 512 bytes.
This may result in very poor performance, (re)-partitioning suggested.
Creating filesystem with 976741831 4k blocks and 244187136 inodes
Filesystem UUID: be74b765-92e7-4d99-a203-36863909129b
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
	102400000, 214990848, 512000000, 550731776, 644972544
```

parted的默认起始扇区为65535，并不是8的整数倍，并且存在不少浪费，转用gdisk

```
Command (? for help): p
Disk /dev/sda: 7814037168 sectors, 3.6 TiB
Model: 008-2DR166
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): C32F7F72-31A6-4EAC-BB7A-5E9EF5E88865
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 7814037134
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048      7814037134   3.6 TiB     8300  Linux filesystem
```

lsblk:

```
sda                         8:0    0   3.7T  0 disk
└─sda1                      8:1    0   3.7T  0 part
```

mkfs

```
➜  ~ sudo mkfs.ext4 /dev/sda1
mke2fs 1.45.5 (07-Jan-2020)
Creating filesystem with 976754385 4k blocks and 244195328 inodes
Filesystem UUID: 678f5019-216a-4d94-95ab-ca4b219b0a95
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
	102400000, 214990848, 512000000, 550731776, 644972544

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

### 设置硬盘自动休眠

[设置hdparm让硬盘在空闲时间休眠。 - Lain的学习博客 (lxf87.com.cn)](http://my.lxf87.com.cn:30002/archives/18/)

### 自动挂载

[Linux下mount挂载新硬盘和开机自动挂载 - sirdong - 博客园 (cnblogs.com)](https://www.cnblogs.com/sirdong/p/11969148.html)



### 配置nuc11显卡驱动

[GPGPU: Ubuntu 20.04 (focal) (intel.com)](https://dgpu-docs.intel.com/installation-guides/ubuntu/ubuntu-focal.html)

```
sudo apt-get install \
  intel-opencl-icd \
  intel-level-zero-gpu level-zero \
  intel-media-va-driver-non-free libmfx1
```

iris xe 显卡有一些额外的问题需要解决

[Ubuntu 20.04 no driver loaded for Intel Iris Xe Graphics - Ask Ubuntu](https://askubuntu.com/questions/1299067/ubuntu-20-04-no-driver-loaded-for-intel-iris-xe-graphics)

```
sudo apt update
sudo apt install linux-oem-20.04
sudo reboot
```



### snap docker

[ubuntu - Docker - mkdir read-only file system - Stack Overflow](https://stackoverflow.com/questions/52526219/docker-mkdir-read-only-file-system)

```
➜  nas docker-compose up -d
Starting jellyfin ...
Starting webdav      ... error
Starting qbittorrent ...

Starting qbittorrent ... error
Starting jellyfin    ... error
ERROR: for qbittorrent  Cannot start service qbittorrent: error while creating mount source path '/data/config/qbittorrent': mkdir /data: read-only file system

ERROR: for jellyfin  Cannot start service jellyfin: error while creating mount source path '/data/config/jellyfin': mkdir /data: read-only file system

ERROR: for webdav  Cannot start service webdav: error while creating mount source path '/data/share': mkdir /data: read-only file system

ERROR: for qbittorrent  Cannot start service qbittorrent: error while creating mount source path '/data/config/qbittorrent': mkdir /data: read-only file system

ERROR: for jellyfin  Cannot start service jellyfin: error while creating mount source path '/data/config/jellyfin': mkdir /data: read-only file system
ERROR: Encountered errors while bringing up the project.
```

### WOL

[Enabling Wake-On-LAN (In Ubuntu 20.10) | The Cloistered Monkey (necromuralist.github.io)](https://necromuralist.github.io/posts/enabling-wake-on-lan/)



### exfat in ubuntu

```
sudo apt install exfat-utils
```



### Jellyfin 硬解

[Hardware Acceleration | Documentation - Jellyfin Project](https://jellyfin.org/docs/general/administration/hardware-acceleration.html#tips-for-intel-gen9-and-gen11-when-using-vaapi-or-qsv-on-linux)

```
For Intel Comet Lake or newer iGPUs, the legacy i965 VA-API driver is incompatible with your hardware. Please follow the instructions from Configuring Intel QuickSync(QSV) on Debian/Ubuntu to get the newer iHD driver.
```

11代cpu核显需要使用新的驱动，而且因为licenses的原因docker镜像里不能包含驱动，所以需要手动安装

```
sudo apt update
sudo apt install vainfo intel-media-va-driver-non-free -y
```

解码 2160p HEVC 10bit视频测试一下

安装intel-gpu-tool查看占用：

```
sudo apt install intel-gpu-tools
sudo intel-gpu-top
intel-gpu-top - 1100/1299 MHz;    0% RC6; ----- (null);     3610 irqs/s

      IMC reads:   ------ (null)/s
     IMC writes:   ------ (null)/s

          ENGINE      BUSY                                                                                                                    MI_SEMA MI_WAIT
     Render/3D/0  100.00% |████████████████████████████████████████████████████████████████████████████████████████████████████████████████▉|     36%      0%
       Blitter/0    0.00% |                                                                                                                 |      0%      0%
         Video/0   46.35% |████████████████████████████████████████████████████▎                                                            |      0%      0%
         Video/1   39.64% |████████████████████████████████████████████▊                                                                    |      0%      0%
  VideoEnhance/0   38.52% |███████████████████████████████████████████▌                                                                     |      0%      0%


```



### qBittorrent

直接访问页面会提示Unauthorized

需要在配置文件qBittorrent.conf加一行

```
WebUI\HostHeaderValidation=false
```

[qBittorent UI returns 401 (blank page) when accessed through Kubernetes proxy · Issue #8095 · qbittorrent/qBittorrent (github.com)](https://github.com/qbittorrent/qBittorrent/issues/8095#issuecomment-472740702)

然后访问页面，使用默认账户admin，密码adminadmin登陆

## 最终效果