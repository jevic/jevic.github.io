---
layout: post
categories: Linux Shell
title: 磁盘分区fdisk/parted
date: 2017-06-04 15:39:53 +0800
description: 硬盘分区
keywords: fdisk parted
---

## fdisk
```
# cat fomat.sh 
#!/bin/bash
disks="sdb sdc sdd sde sdf sdg sdh"
for disk in $disks
do
  fdisk /dev/$disk <<EOF
d
n
p



w
EOF

mkfs -t xfs -f -i size=512 /dev/${disk}1
done

----

#!/bin/bash
devs="sdb sdc sdd sde sdf sdg sdh"

declare -A m_list
m_list=([sdb]=/data1 [sdc]=/data2 [sdd]=/data3 [sde]=/data4 [sdf]=/data5 [sdg]=/data6 [sdh]=/data7)


for i in $devs;do
    echo "mount /dev/$i ${m_list[$i]}"
done
```

## parted
>GUID分区表(简称GPT。使用GUID分区表的磁盘称为GPT磁盘)是源自EFI标准的一种较新的磁盘分区表结构的标准。与普遍使用的主引导记录(MBR)分区方案相比，GPT提供了更加灵活的磁盘分区机制

fdisk不支持[GPT](https://wiki.archlinux.org/index.php/Partitioning_(简体中文))(当硬盘容量大于2TB时候无法使用fdisk进行分区的管理)

```
~ # parted /dev/sdc
GNU Parted 3.1
Using /dev/sdc
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mktable
New disk label type? gpt
(parted) mkpart
Partition name?  []? data
File system type?  [ext2]? ext4
Start? 0
End? 4000G
Warning: The resulting partition is not properly aligned for best performance.
Ignore/Cancel? i
(parted) p
Model: ATA ST4000NM0033-9ZM (scsi)
Disk /dev/sdc: 4001GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name   Flags
 1      17.4kB  4000GB  4000GB               data2
(parted) q
Information: You may need to update /etc/fstab.
~ # mkfs.ext4 /dev/sdc1
```
