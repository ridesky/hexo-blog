---
layout: 整理
title: 挂载硬盘的踩坑流程
date: 2017-12-19 18:24:22
tags: test
categories: 测试分类
---
# 整理 CentOS 挂载硬盘的踩坑流程

#####Step.1 安装 ntfs-3g 解决 mount: unknown filesystem type 'ntfs'

通过 ntfs-3g 处理，官方下载地址：http://www.tuxera.com/community/ntfs-3g-download/

这里以2016.2.22版本为例进行编译安装。

```
wget https://tuxera.com/opensource/ntfs-3g_ntfsprogs-2016.2.22.tgz

tar zxvf  ntfs-3g_ntfsprogs-2016.2.22.tgz

cd ntfs-3g_ntfsprogs-2016.2.22

/configure

make

make install
```

#####Step.2 使用 parted 查看所有硬盘信息，并使用 mount 挂载硬盘

这里说下，为什么不用fdisk。parted命令可以划分单个分区大于2T的GPT格式的分区，也可以划分普通的MBR分区，fdisk命令对于大于2T的分区无法划分，所以用fdisk无法看到parted划分的GPT格式的分区。


执行 parted -l 的返回结果
这里需要强调的是，在windows下格式化GPT硬盘，往往会产生一个Microsoft Reserved Partition分区（MSR,大概几百MB）和一个Basic Data Partition（真正存储数据的地方）分区，这有助于windows管理和操作GPT硬盘。但是将Windows格式化的GPT硬盘放到Linux服务器，Linux是不会识别这两个分区的。假设硬盘设备是 /dev/sdd，直接使用 mount /dev/sdd /mnt/data1 命令会报错，用fdisk -l命令查看硬盘设备，只会显示MSR分区，而不管Basic Data Partition分区，这也是使用 parted 而不是 fdisk 的原因。

所以，以下为正确的挂载命令，这里有一点要注意，一定要先在 /mnt 这里创建一个 data 目录，不然会报错的 - -

`mount -t ntfs-3g /dev/sdd2 /mnt/data1`
>Tips: 因为看不到硬盘标签，可以使用 blkid 命令查看下标签再继续操作，还是很方便的。


第6、7行的"1080P_B"、"1080P_A"就是我之前在Windows系统中对硬盘设置的标签
#####Step.3 针对 /etc/fstab 文件设置开机启动加载分区

首先做下备份：

```
cp /etc/fstab  /etc/fstab.bak
进入fstab文件并加上挂载代码

dev/sdd2 /mnt/data1 ntfs-3g defaults 0 0

```

请看倒数第二行
到此，挂载的部分就搞定了。以下为格式化部分。

因为 ext4 格式不知道比 ntfs 强多少倍，所以我对硬盘进行了格式化。

这里注意，直接格式化会报错，需要先取消挂载（umount）再执行格式化（mkfs）

```
umount /dev/ssd2

mkfs -t ext4 /dev/sdd2
```

之后再重新挂载就ok了，这里还有个需要注意的地方！因为改变了硬盘格式，所以需要回到 Step.3 修改下 fstab 文件的格式，不然重新之后就GG了。

>Tips: 格式化之后查看硬盘空间大小请使用 df -h 命令

