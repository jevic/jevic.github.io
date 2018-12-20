---
layout: post
categories: Linux
title: CentOS 内核 kernel升级
date: 2017-02-18 19:56:06 +0800
description: kernel,linux,centos
keywords: centos
---

CentOS 7.2 默认内核版本为3.10

## install
```
# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# yum --enablerepo=elrepo-kernel install kernel-ml-devel kernel-ml -y
```

## grub

```
# vim /etc/default/grub
修改
GRUB_DEFAULT=0
保存退出

# grub2-mkconfig -o /boot/grub2/grub.cfg
```

## reboot
查看内核版本

```
# uname -r
4.09.1-1.el7.elrepo.x86_64
```
