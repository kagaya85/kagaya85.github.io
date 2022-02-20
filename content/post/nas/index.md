---
title: "瞎折腾：利用Nuc11打造NAS与个人影音中心"
description: "年轻人的第一个nas"
date: 2022-01-17T16:35:05+08:00
tags: ["NAS", "NUC", "Jellyfin", "Linux"]
categories: ["Techonlogy"]
image: 59474391_p0.jpeg
hidden: false
comments: true
draft: true
---

## 写在前面

为什么我会突然想搞一台NAS呢，我这么问自己，仔细回想一下大概是因为自从番剧需要进行审核才能上映的政策出台后，我就准备了一块海康威视的1T ssd接一块败家之眼的m2硬盘盒用来存自己下载的番剧，同时也方便和朋友分享；后来阿b推出了放映室的功能，在网页上就可以很方便的和朋友一同看番看电影。但这样终究是不方便，首先放映室的内容只能先定于阿b已经拥有版权的内容，其次，你的朋友最好还要有大会员。好在我本身就是个爱折腾的人，刚好前段时间给群友搞了一个ts服务器来让群友们吹水聊天以及游戏语音，我就开始想能不能通过自建的服务器也实现一个类似的流媒体平台，随时和朋友分享自己私藏的电影番剧。

假如有这样一个场景：某天有朋友（小姐姐）想看个电影但苦于没有资源，又不想去各种牛皮癣广告横飞的小网站，只得求助于你。你虽然早已掌握科学上网快速通过各种途径网罗了种子资源，熟练地运用bt下载器下载到了最新的4K HDR 10bit资源，那么你该怎么和远在网线另一端的她分享呢？当然你可以把bt文件丢过去耐心地教她怎么用FDM来下载，但是，如果此时我们能有一个自己的流媒体服务器，不就可以随时随地的给任何人分享硬盘里的片了吗？

之前发现租的房子是有公网ip的，在暴力破解了宽带账号把光猫改成桥接以后，就可以直通外网了，尽管上行带宽并不富裕，但这么好的资源不拿来搭服务器简直浪费，而又“恰好”实验室发电脑是一台最新nuc11，配了11代的酷睿i7-1165G7，平时都用mac的我正愁该如何在毕业前发挥它的剩余价值，这波天时地利就被赶上了，直接开搞。

## 方案探索

基于nas的媒体中心其实已经有不少成熟解决方案，Emby，Plex以及开源的Jellyfin都是最常用的。我们只需要吧nas里存的片共享给流媒体服务器，然后在外网暴露端口就行。这里其实涉及到几个问题：

### 硬件

硬件选择比较简单，主要思想通过主机+硬盘盒raid的方式来实现：

主机用的是白嫖的Intel NUC11PAHi7 猎豹峡谷，40w的TDP，硬盘从海鲜市场捡了两块4T的希捷酷狼，硬盘盒同样也是从海鲜市场捡的二手世特力双盘位带raid的硬盘盒，两块硬盘组raid1

### nas底层系统选择

在选择系统之前首先还是要明确需求，对我来说，需求主要有以下几点：

1. 可以7x24稳定运行
2. 可以硬件直通方便媒体服务器使用硬件编解码
3. 可以开虚拟机
4. 对Docker容器能有较好的支持
5. 可以方便备份迁移

现在市面上几乎所有系统都可以用来做nas，因为对linux较为熟悉且为了获得原生的docker性能和体验，我在一开始就排除的windows系统，另外Free所以主要的选择在一下

exsi

omv

ubuntu

unraid

### 硬件直通



### 迁移与备份

目前的方案肯定是临时的

## 安装过程

这里简单记录一下当时主要的安装配置过程，想看最终效果的可以直接点[这里](#最终效果)

### 配置UPS

ups几乎是nas必备了

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

mkfs:

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

验证了一下，并不会自动休眠，反正硬盘不多，索性就让它7x24小时转吧

### 自动挂载

[Linux下mount挂载新硬盘和开机自动挂载 - sirdong - 博客园 (cnblogs.com)](https://www.cnblogs.com/sirdong/p/11969148.html)



### 配置nuc11显卡驱动

intel对ubuntu 20.04的支持还是比较好的，可以按官方文档来装驱动：[GPGPU: Ubuntu 20.04 (focal) (intel.com)](https://dgpu-docs.intel.com/installation-guides/ubuntu/ubuntu-focal.html)

```shell
sudo apt-get install \
  intel-opencl-icd \
  intel-level-zero-gpu level-zero \
  intel-media-va-driver-non-free libmfx1
```

安装驱动后没有看到/dev/dri文件夹，发现iris xe 显卡有一些额外的包需要安装：

[Ubuntu 20.04 no driver loaded for Intel Iris Xe Graphics - Ask Ubuntu](https://askubuntu.com/questions/1299067/ubuntu-20-04-no-driver-loaded-for-intel-iris-xe-graphics)

```shell
sudo apt update
sudo apt install linux-oem-20.04
sudo reboot
```

### WOL配置

[Enabling Wake-On-LAN (In Ubuntu 20.10) | The Cloistered Monkey (necromuralist.github.io)](https://necromuralist.github.io/posts/enabling-wake-on-lan/)



### exfat in ubuntu

由于我之前为了在多平台之间通用，移动硬盘被格式化为零exfat格式，linux默认读写exfat的时候偶尔会有问题，安装额外的包来获得更稳定的兼容：

```shell
sudo apt install exfat-utils
```

### snap docker

配置好docker compose后，启动遇到了下面的问题：

```
➜ docker-compose up -d
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

查了一下，是snap自带docker的问题，删除ubuntu自带的docker重新安装官方的docker即可

[ubuntu - Docker - mkdir read-only file system - Stack Overflow](https://stackoverflow.com/questions/52526219/docker-mkdir-read-only-file-system)

### Jellyfin 硬解

实验室给的这个nuc11的这块1165g7自带来一个蓝厂最新的iris Xe核显，96EU，性能应该是同代核显里最强的了，据说能有mx350的水平？不拿来当解码器真是浪费了。

但问题也就出在这块核显实在太新了，需要新的驱动程序，配置方式也有一些不同，因此考虑显卡直通之前，一定先要调研好软硬件的兼容性。

Jellyfin的官方文档：[Hardware Acceleration | Documentation - Jellyfin Project](https://jellyfin.org/docs/general/administration/hardware-acceleration.html#tips-for-intel-gen9-and-gen11-when-using-vaapi-or-qsv-on-linux)，中有这么一段话：

```
For Intel Comet Lake or newer iGPUs, the legacy i965 VA-API driver is incompatible with your hardware. Please follow the instructions from Configuring Intel QuickSync(QSV) on Debian/Ubuntu to get the newer iHD driver.
```

所以不能在使用之前的VAAPI，转而使用最新的QuickSync方式：

![ScreenShot_2022-01-27_at_18.26.01@2x](ScreenShot_2022-01-27_at_18.26.01@2x.png)

11代cpu核显需要使用新的驱动，而且因为licenses的原因docker镜像里不能包含驱动，所以需要手动安装：

```shell
sudo apt update
sudo apt install vainfo intel-media-va-driver-non-free -y
```

安装好驱动后，`ls /dev/dri`检查一下是否有对应的目录

```shell
➜  ~ ls /dev/dri
by-path  card0  renderD128
```

然后就可以在docker-compose文件中把显卡驱动共享给容器，同时记得需要给容器添加到render的用户组里

```yaml
version: "3.9"
services:
  jellyfin:
    image: linuxserver/jellyfin:10.7.7
    container_name: jellyfin
    ports:
    	- ...
    group_add:
      - 109 # add to render group, see: cat /etc/group | grep render
    environment:
    	- ...
    devices:
      - /dev/dri:/dev/dri # Intel 集显驱动
```

或者把当前用户添加到render组同时通过设置容器内的启动用户为当前用户来实现统一的目的

```shell
sudo usermod -a -G render [username]
```

解码 2160p HEVC 10bit视频测试一下，安装intel-gpu-tool查看占用，可以看到确实有用到显卡的解码性能：

```shell
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

直接访问页面会提示Unauthorized：[qBittorent UI returns 401 (blank page) when accessed through Kubernetes proxy · Issue #8095 · qbittorrent/qBittorrent (github.com)](https://github.com/qbittorrent/qBittorrent/issues/8095#issuecomment-472740702)

需要在配置文件`qBittorrent.conf`加一行（注意需要通过映射目录做好配置的持久化）

```
WebUI\HostHeaderValidation=false
```

重启容器，然后访问页面，使用默认账户admin，密码adminadmin登陆

### 私有云盘



## 最终效果

做好端口映射后公网就可以访问网页：

我在这里顺手为域名去加了证书来支持https访问

![ScreenShot_2022-01-27_at_18.45.53@2x](ScreenShot_2022-01-27_at_18.45.53@2x.png)

直接访问网页是最方便的方式，但是受限与浏览器的原因，Chromium系的浏览器不能使用hevc的硬解，win10下的Edge应该可以通过额外的插件来实现hevc硬解，因此，想要通过客户端硬解来节省可怜的小水管带宽的话，可以使用Jellyfin的客户端：[jellyfin/jellyfin-media-player: Jellyfin Desktop Client based on Plex Media Player (github.com)](https://github.com/jellyfin/jellyfin-media-player)，或者用safari等浏览器

最好提前用tmm等工具搜刮一下相关的介绍/图片补充元数据，然后统一重命名（为了不妨碍做种，我选择用`cp -al`硬连接到Jellyfin的目录，再改名，这样也不会占用额外的空间）：

![ScreenShot_2022-01-27_at_17.55.35@2x](ScreenShot_2022-01-27_at_17.55.35@2x.png)

![ScreenShot_2022-01-27_at_17.59.21@2x](ScreenShot_2022-01-27_at_17.59.21@2x.png)

可以看到Jellyfin演员配图质量还是挺高的：

![ScreenShot_2022-01-27_at_17.59.49@2x](ScreenShot_2022-01-27_at_17.59.49@2x.png)

![ScreenShot_2022-01-27_at_18.02.26@2x](ScreenShot_2022-01-27_at_18.02.26@2x.png)

转码速度还是挺快的，瓶颈主要是在家里的上行带宽只有可怜的10m多，所以网页端转码，最多只能开到1080p，码率也不敢太高更不可能多人一起看了， 而如果用客户端或者safari浏览器硬解，4K direct几乎可以十分流畅的播放：

![ScreenShot_2022-01-27_at_18.49.54@2x](ScreenShot_2022-01-27_at_18.49.54@2x.png)



![ScreenShot_2022-01-27_at_18.51.17@2x](ScreenShot_2022-01-27_at_18.51.17@2x.png)