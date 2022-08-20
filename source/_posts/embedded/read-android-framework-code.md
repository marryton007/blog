---
title: "阅读Android框架源码"
date: 2022-05-20 14:17:00
tags:
  - Android
  - Android studio
  - linux
categories:
  - embedded
---



## 1. 背景

之前一直都是看Kernel源码，网上有很多好用的工具，如：

* [elixir][elixir]
* [sourcegraph+ github][sourcegraph]
* [vim + ccls][vim]

最近，因为要跟踪Android框架某个流程，须查看Android的框架源码，尝试过几种方案后，最终选择了[rclone][rclone]+Android Studio，先说一下工作环境。

## 2. 环境

Android 源码存放在Linux服务器上，之所以放在服务器上，而不是本地PC，主要是利用服务器性能和存储空间，毕竟Android框架代码太大了，压缩过后大约50G，解压出来100G，编译过后200G。平时工作时，使用 [windterm][windterm]连到服务器，使用[vim+ccls][vim]就可以完成日常工作了。

## 3. 尝试

因为要阅读Java代码，这里先后使用了

* vscode(remote ssh)+java
* rclone + source insight
* Android stuido(linux版) + vcxSrv
* rclone + Android studio



其中，vscode不能跳转，估计是不能直接把某个目录加入vscode中，source insight搜索文件倒是好用，但跳转函数定义不是很好用。这些支持得最好的还是Android studio，开始的时候尝试在Linux上运行Android studio，再通过vcxSrv将图形界面展示到Window上，但一些快捷键不能使用，只能通过鼠标点击，效率大大降低。这里使用[rclone][rclone]的作用是将Linux上某个目录直接映射到Windows上的虚拟磁盘上，因为我们编译的出来的Android固件比较大，不想每次都拷贝到本地，再烧录到开发上，使用[rclone][rclone]后，就可以直接指定从虚拟磁盘上下载并烧录Android固件了。

## 4. 虚拟磁盘rclone

在使用[rclone][rclone]前，使用过

* [sshfs-win-manager][sshfs]
* [SFTP drive][sftp]

[sshfs-win-manager][sshfs]最大的问题是不能直接下载比较大的文件，比如2G

[SFTP drive][sftp]倒是可以下载大文件，但免费版每次只能影射一个虚拟磁盘，这让有多台服务器的同学怎么办？

[rclone][rclone] 解决了上面的问题，并且没有广告，这么好的东东哪里去找，果断替换。

### 4.1 安装

```shell
scoop install rclone
```

首次使用需要配置一下

```shell
rclone config
```

配置文件内容如下：

```json
[252]
type = sftp
host = 192.168.200.252
user = example
key_file = ~/.ssh/id_rsa.pem
md5sum_command = md5sum
sha1sum_command = sha1sum

[253]
type = sftp
host = 192.168.200.253
user = example
key_file = ~/.ssh/id_rsa.pem
md5sum_command = md5sum
sha1sum_command = sha1sum
```

这里的配置看起来比较简单，主要用意是使用keyfile+sftp登录服务器。最后，将服务器目录影射为虚拟磁盘。

### 4.2 影射虚拟磁盘

```shell
// 将252服务器根目录影射到X盘，并启用缓存
rclone mount 252:/ x: --cache-dir D:\cache\252 --vfs-cache-mode writes &
```

## 5. 将Android源码导入Android Studio

### 5.1 准备

```shell
cd /Android/Source
ls -l |grep ^d |grep -v frameworks |grep -v system |awk '{print $9}'
```

这一步是将Android源码下的目录截取出来，以便后面排除不关心的目录，我这里仅保留了frameworks和system

```shell
art
bionic
bootable
build
compatibility
cts
dalvik
developers
development
device
EHomeApp
EHomePreinstall
external
gen
hardware
kernel
libcore
libnativehelper
mkcombinedroot
out
packages
pdk
platform_testing
prebuilts
rkbin
RKDocs
rkst
RKTools
rockdev
sdk
test
toolchain
tools
u-boot
vendor
```

将上面的结果附加到development/tools/idegen/excluded-paths文件末尾

```shell
ls -l |grep ^d |grep -v frameworks |grep -v system |awk '{print "^"$9}' >> development/tools/idegen/excluded-paths 
```

### 5.2 生成Android项目

```shell
cd Android/Source
source build/envsetup.sh
lunch xxxxx
mmm development/tools/idegen
development/tools/idegen/idegen.sh
```

以上命令序列将在Android/Source目录下生成android.ipr和Android.iml，Android studio可直接打开android.ipr，先修改Android.iml文件，排除不关心的目录

```shell
ls -l |grep ^d |grep -v frameworks |grep -v system |awk '{print "<excludeFolder url=\"file://$MODULE_DIR$/"$9"\" />"}'
```

得到如下内容：

```shell
<excludeFolder url="file://$MODULE_DIR$/art" />
<excludeFolder url="file://$MODULE_DIR$/bionic" />
<excludeFolder url="file://$MODULE_DIR$/bootable" />
<excludeFolder url="file://$MODULE_DIR$/build" />
<excludeFolder url="file://$MODULE_DIR$/compatibility" />
<excludeFolder url="file://$MODULE_DIR$/cts" />
<excludeFolder url="file://$MODULE_DIR$/dalvik" />
<excludeFolder url="file://$MODULE_DIR$/developers" />
<excludeFolder url="file://$MODULE_DIR$/development" />
<excludeFolder url="file://$MODULE_DIR$/device" />
<excludeFolder url="file://$MODULE_DIR$/EHomeApp" />
<excludeFolder url="file://$MODULE_DIR$/EHomePreinstall" />
<excludeFolder url="file://$MODULE_DIR$/external" />
<excludeFolder url="file://$MODULE_DIR$/gen" />
<excludeFolder url="file://$MODULE_DIR$/hardware" />
<excludeFolder url="file://$MODULE_DIR$/kernel" />
<excludeFolder url="file://$MODULE_DIR$/libcore" />
<excludeFolder url="file://$MODULE_DIR$/libnativehelper" />
<excludeFolder url="file://$MODULE_DIR$/mkcombinedroot" />
<excludeFolder url="file://$MODULE_DIR$/out" />
<excludeFolder url="file://$MODULE_DIR$/packages" />
<excludeFolder url="file://$MODULE_DIR$/pdk" />
<excludeFolder url="file://$MODULE_DIR$/platform_testing" />
<excludeFolder url="file://$MODULE_DIR$/prebuilts" />
<excludeFolder url="file://$MODULE_DIR$/rkbin" />
<excludeFolder url="file://$MODULE_DIR$/RKDocs" />
<excludeFolder url="file://$MODULE_DIR$/rkst" />
<excludeFolder url="file://$MODULE_DIR$/RKTools" />
<excludeFolder url="file://$MODULE_DIR$/rockdev" />
<excludeFolder url="file://$MODULE_DIR$/sdk" />
<excludeFolder url="file://$MODULE_DIR$/test" />
<excludeFolder url="file://$MODULE_DIR$/toolchain" />
<excludeFolder url="file://$MODULE_DIR$/tools" />
<excludeFolder url="file://$MODULE_DIR$/u-boot" />
<excludeFolder url="file://$MODULE_DIR$/vendor" />
```

将上面的内容放到android.iml文件的合适位置，并稍做修改，删除重复项即可

### 5.3 导入Android工程

现在将可以使用Windows上Android Studio打开虚拟磁盘上的Android.ipr文件，等待Android studio索引完成，大功告成。

### 6. 参考资料：

[kernel代码在线阅读][elixir]

[使用github+sourcegraph在线阅读内核代码][sourcegraph]

[2022 年 vim 的 C/C++ 配置][vim]

[Windows平台下使用 Rclone 挂载 OneDrive 为本地硬盘][rclone]

[使用 Android Studio 阅读 AOSP 源码][aosp]



[aosp]:https://blog.devwu.com/2019/06/23/how-to-reading-AOSP-with-Android-Studio/
[elixir]:https://elixir.bootlin.com/linux/latest/source
[sourcegraph]:https://sourcegraph.com/github.com/torvalds/linux
[vim]:https://martins3.github.io/My-Linux-Config/docs/nvim.html
[rclone]:https://zhuanlan.zhihu.com/p/139200172
[windterm]: https://github.com/kingToolbox/WindTerm
[sshfs]:  https://github.com/evsar3/sshfs-win-manager
[sftp]:https://www.nsoftware.com/sftp/drive/
