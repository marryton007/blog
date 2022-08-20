---
title: "Android环境中bcc/bpftrace环境搭建"
date: 2022-04-29 11:54:00
tags:
  - Android
  - adeb
  - linux
categories:
  - embedded
---

## 1. 背景

最近想研究一下Android中的性能分析，特别是关于使用[bpftrace][bpftrace],[bcc][bcc]等工具对Linux内核进行动态跟踪的方法，因为Android本身工具并不丰富，网上大部分文章都是推荐使用[adeb][adeb]这个工具，本篇文章就是关于如何搭建[adeb][adeb]环境。

## 2. 环境

Ubuntu18.04 + RK3568开发板 Android11

## 3. 原理

简单来讲，adeb使用qemu-debootstrap工具生成了一个文件系统，并将这个文件系统通过adb push到Android开发板上，最后chroot到这个文件系统中运行shell。

## 4. 使用

### 4.1 快速使用

```shell
// 下载预先制作好的文件系统
cd ~/download
curl -O https://github.com/joelagnel/adeb/releases/download/v0.99h/androdeb-fs.tgz.zip
unzip androdeb-fs.tgz.zip

// 下载adeb
cd ~/git
git clone  https://github.com/marryton007/adeb.git
cd adeb
sudo ln -s $(pwd)/adeb /usr/bin/adeb
// 准备adeb环境
adeb prepare --archive ~/download/androdeb-fs.tgz
// 登录进入开发板，这里需要adb能访问开发板
adeb shell
```

### 4.2 自制文件系统

也许前面的文件系统不能完全满足你的要求，你可以修改adeb源码，添加自己的软件包，并制作自己的文件系统

```shell
// 制作文件系统，并打包成androdeb-fs.tgz
adeb prepare --full --buildtar
cp androdeb-fs* ~/download
```

### 4.3 通过NFS加载

即便你生成了androdeb-fs.tgz,但每次执行下面的指令都要花费很长的时间，因为它会上传压缩文件，并解压，花费的时间与文件系统大小有关，另外，如果开发板存储空间不够的话，也会有问题。

```shell
adeb prepare --archive ~/download/androdeb-fs.tgz
```

生命有限，不要将时间浪费在无谓的重复上，解决办法：使用NFS加载文件系统

### 4.4 准备NFS服务器

```shell
sudo apt install nfs-kernel-server
sudo mkdir /opt/androdeb
echo "/opt/androdeb  *(rw,sync,no_subtree_check,no_root_squash,insecure)" |sudo tee /etc/exports
sudo systemctl restart nfs-server
sudo exportfs -rv
```

### 4.5 填充文件系统内容

```shell
sudo cp ~/git/adeb/addons/* /opt/androdeb
sudo tar xf ~/download/androdeb-fs.tgz -C /opt/androdeb
```

### 4.5 Android开发板准备

Android系统要加载NFS文件系统，首先要内核支持NFS，请确保以下Kernel选项是选中的。

```shell
CONFIG_NETWORK_FILESYSTEMS=y
CONFIG_NFS_FS=y
CONFIG_NFS_V2=y
CONFIG_NFS_V3=y
CONFIG_NFS_COMMON=y
CONFIG_SUNRPC=y
```

### 4.6 加载NFS文件系统

```shell
adb root && adb remount && adb tcpip 5555
adb shell
// 如果不关闭 selinux，mount 时会报：failed: I/O error
setenforce 0
mkdir /data/androdeb
// 请将nfsserver用真实IP地址替换
busybox mount -t nfs -o nolock  nfsserver:/opt/androdeb  /data/androdeb
```

### 4.7 通过NFS文件系统运行bcc，bpftrace

通过NFS加载文件系统后，即可以切换到NFS文件系统

```shell
adb shell
cd /data/androdeb
source run.common
do_mounts
// chroot到NFS文件系统
./run
opensnoop-bpfcc
```

```shell
<built-in>:1:10: fatal error: './include/linux/kconfig.h' file not found
#include "./include/linux/kconfig.h"
         ^~~~~~~~~~~~~~~~~~~~~~~~~~~
1 error generated.
Traceback (most recent call last):
  File "/usr/sbin/opensnoop-bpfcc", line 181, in <module>
    b = BPF(text=bpf_text)
  File "/usr/lib/python2.7/dist-packages/bcc/__init__.py", line 320, in __init__
    raise Exception("Failed to compile BPF text")
Exception: Failed to compile BPF text
```

这里是由于缺少Kernel头文件，我直接通过sshfs(你可以尝试NFS)将kernel源码影射到/lib/modules/`uname -r`/build目录下。

```shell
// 以下命令是在NFS文件系统中执行
mkdir -p /lib/modules/`uname -r`/build
// 由于build服务器只能使用密钥登录，这里将私钥将传到Android开发板上
chmod 600 /data/local/id_rsa
sshfs xxx@build-server:/opt/sdk/rock/rk-android11/kernel /lib/modules/`uname -r`/build -o IdentityFile=/data/local/id_rsa
```

如果上面的sshfs使用有问题，可以尝添加如下参数查看DEBUG信息

```shell
sshfs xxx@build-server:/opt/sdk/rock/rk-android11/kernel /lib/modules/`uname -r`/build -o IdentityFile=/data/local/id_rsa -o debug -o sshfs_debug
```

## 总结

整个过程看起来还是比较复杂的，既要编译内核，又用到了2种网络文件系统(NFS，SSHFS)，这也与Android系统自身有关系，毕竟与通用的Linux还是有不少差别，这里也是做个记录，怕忘记了。

## 参考

[原始的adeb仓库][adeb-origin]
[我的adeb仓库][adeb]
[nfs server 过防火墙][jianshu]
[通过wifi使用nfs把ubuntu挂载到android][blog]

[adeb-origin]: https://github.com/joelagnel/adeb
[adeb]: https://github.com/marryton007/adeb.git
[bcc]: https://github.com/iovisor/bcc
[bpftrace]: https://github.com/iovisor/bpftrace
[jianshu]: https://www.jianshu.com/p/b6f00754ead6
[blog]: https://blog.csdn.net/u010164190/article/details/100142755
