---
layout: post
categories: Docker
title: Docker device or resource busy 问题
date: 2018-04-10 20:09:12 +0800
description: docker
keywords: docker
---

## 现象描述
- 查看容器发现 Dead 状态容器

```
[root@node ~]# docker ps -a |grep Dead
e0fdd4b75b7c        store.jevic.com/fileinject-agents/tcs:a35104a    "/fileinject-agent..."   28 hours ago        Dead                                             r-fileinject-agents-tcs-16-81cfe394
7d7841c49c7b        store.jevic.com/fileinject-agents/ups:ebb7241    "/fileinject-agent..."   28 hours ago        Dead                                                                                       

```

- 查看系统挂载 
    - 此时有很多Dead 容器依然被挂载中 

```
[root@node ~]# df -h|grep docker
overlay         415G   18G  398G   5% /var/lib/docker/overlay/93b1805f06dda8ae47b762465ca51ec27463fd715d08f91b25ca21ac074d8010/merged
shm              64M     0   64M   0% /var/lib/docker/containers/420d6805eede457b88f0781f50d8744647d1cdcc535f40207259abdd68f6e31e/shm
overlay         415G   18G  398G   5% /var/lib/docker/overlay/
....... 此次省略......

```

## 删除容器

```
[root@node ~]# docker rm -f 7d7841c49c7b
Error response from daemon: driver "overlay" failed to remove root filesystem for 7d7841c49c7b4109756274020d93776ce615f0f512c4f21e05b3ac94371b0604: remove /var/lib/docker/overlay/80299696071d4181254a31052dfecea062201905f0b3187ae241575fbc7a059e/merged: device or resource busy

提示: 设备正忙

卸载重新删除
[root@node ~]# umount var/lib/docker/overlay/80299696071d4181254a31052dfecea062201905f0b3187ae241575fbc7a059e/merged
[root@node ~]# docker rm -f 7d7841c49c7b
....
```

>最后 重新docker ps 发现 容器已经被删除
>>对于有些应用容器 可能上述步骤依然无法删除掉容器;
>>直接重启宿主机也可以恢复正常,当然根据实际情况来判断是否需要此操作毕竟会导致服务中断；
>>这种Dead 状态的并不影响docker正常使用，所以可视情况执行操作；

---

## 问题原因

>挂载点泄露实际上是 RHEL/CentOS 内核的一个 bug，目前预计会在 RHEL7.4 kernel 中修复。
因此该问题理论上在 Fedora,ubuntu  等发行版本中是不存在的。
目前在RHEL/CentOS下需要在 docker.service中增加MountFlags=slave来保证 mount namespace 是私有的，不会造成挂载点泄露。
另外，挂载点泄露在各种 storage driver 中都存在，比如最常见的 devicemeppaer 和 overlay。

>>但是，目前 RHEL/CentOS 版本下 MountFlags=slave 和 --live-restore 两个给力的参数不能同时存在。
>>因为 MountFlags=slave 会导致 docker daemon 每次重启时私有挂载命名空间都会发生变化，而 --live-restore 又相当于使得容器持有了变化之前的旧的挂载点信息，因此，当重启 docker daemon 之后，执行 docker exec 试图进入容器时会报错值得一提的是，虽然无法进入容器，但是容器依旧工作正常并且 docker logs 也没问题。

## 总结
>对于生产环境，遇到 Device is busy 问题时，使用 docker rm -f [container] 后重新启动新的容器即可；
对于新的机器，当前版本下为 docker.service 加上 MountFlags=slave 即可。


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
