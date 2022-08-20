---
title: "交叉编译trace-cmd"
date: 2021-11-29 09:32:00
tags:
  - Android
  - ftrace
  - linux
categories:
  - embedded
---

## 1. 背景

在查看瑞芯微《Thermal开发指南》时，看到了可以使用trace-cmd来进行调试，出于对于类似调试工具的好奇，尝试在本地机器上交叉编译trace-cmd并上传到Android设备上使用，中间过程比较折腾，最终出了本文。同时，本文也可以作为类似工作的一个参考。



## 2. 编译过程

### 2.1 依赖

根据官方文档，trace-cmd用到了2个库，它们需要被先编译

* libtracceevent
* libtracefs



### 2.2 下载源码并编译

```sh
# 创建工作目录
mkdir ~/git/

# 下载libtraceevent并交叉编译libtraceevent
git clone https://git.kernel.org/pub/scm/libs/libtrace/libtraceevent.git/
cd libtraceevent
CC=aarch64-linux-gnu-gcc CROSS_COMPILE=aarch64-linux-gnu- make DESTDIR=../build install
# 重要，替换libtraceevent.pc中的prefix
sed -i -e 's@prefix=/usr/local@prefix=/home/jiaxi/git/build/usr/local@g' ../build/usr/local/lib/x86_64-linux-gnu/pkgconfig/libtraceevent.pc

# 下载libtracefs并交叉编译libtracefs
cd ../
git clone https://git.kernel.org/pub/scm/libs/libtrace/libtracefs.git/
cd libtracefs
PKG_CONFIG_PATH=../build/usr/local/lib/x86_64-linux-gnu/pkgconfig CC=aarch64-linux-gnu-gcc CROSS_COMPILE=aarch64-linux-gnu- make DESTDIR=../build install
# 重要，替换libtracefs.pc中的prefix
sed -i -e 's@prefix=/usr/local@prefix=/home/jiaxi/git/build/usr/local@g' ../build/usr/local/lib/x86_64-linux-gnu/pkgconfig/libtracefs.pc

# 下载trace-cmd并交叉编译
cd ../
git clone git://git.kernel.org/pub/scm/utils/trace-cmd/trace-cmd.git
cd trace-cmd
PKG_CONFIG_PATH=../build/usr/local/lib/x86_64-linux-gnu/pkgconfig CC=aarch64-linux-gnu-gcc CROSS_COMPILE=aarch64-linux-gnu-  LDFLAGS=-static make DESTDIR=../build install

# 上传到Android设备
cd ../build/usr/local/bin
adb push trace-cmd /system/bin
```

关键注意点：

* **替换.pc文件中的prefix**，不然pkg-config无法找到依赖库正确的路径
* 最终编译trace-cmd时，要加上**LDFLAGS=-static**，使用静态链接生成trace-cmd，这样，只需要上传trace-cmd这一个文件就可以了，不然在执行时会报"No such file or directory"



### 2.3 关键编译参数

```sh
# 指定交叉编译时使用的gcc
CC=aarch64-linux-gnu-gcc
# 指定交叉编译工具链前缀，与上面CC是一对固定搭配
CROSS_COMPILE=aarch64-linux-gnu-
# 指定安装位置，一般本地编译时默认安装位置是/usr/local，这里的意思是安装位置将会变成../build/usr/local；主要作用有3点，一是防止弄乱本地环境； 二是后面作用pkg-config的搜索路径，三是拷贝的时候方便，直接打包../build目录即可
DESTDIR=../build
# pkg-config工具用于管理依赖关系，PKG_CONIFG_PATH用于指定pkg-config搜索的路径。
PKG_CONFIG_PATH=../build/usr/local/lib/x86_64-linux-gnu/pkgconfig
# 使用静态链接方式编译trace-cmd
LDFLAGS=-static
```



tips: 使用pkg-config 获取libtraceevent的头文件和库文件位置

```sh
~ PKG_CONFIG_PATH=../build/usr/local/lib/x86_64-linux-gnu/pkgconfig pkg-config --cflags libtraceevent 
-I/home/jiaxi/git/build/usr/local/include/traceevent
~ PKG_CONFIG_PATH=../build/usr/local/lib/x86_64-linux-gnu/pkgconfig pkg-config --libs libtraceevent 
-L/home/jiaxi/git/build/usr/local/lib64 -ltraceevent
```



## 3. 运行效果

```sh
# 这一步在Android设备上运行, 会在当前目录生trace.dat文件
rk3568_r:/data/local # trace-cmd record -e thermal -e thermal_power_allocator -b 102400
Hit Ctrl^C to stop recording
^C
CPU0 data recorded at offset=0x507000
    0 bytes in size
CPU1 data recorded at offset=0x507000
    0 bytes in size
CPU2 data recorded at offset=0x507000
    4096 bytes in size
CPU3 data recorded at offset=0x508000
    0 bytes in size
rk3568_r:/data/local # ls
trace.dat

# 下面的步骤在PC上进行
# 下载trace.dat文件
adb pull /data/local/trace.dat  /tmp
# 转换成txt文件，即可进行分析了
trace-cmd report /tmp/trace.dat > /tmp/trace.txt
```



## 4. 扩展资料

trace-cmd 作为ftrace的前端，是一个易于使用，且特性众多、可用来追踪内核函数的命令。

[使用 trace-cmd 追踪内核](https://linux.cn/article-13852-1.html)

[通过 ftrace 来分析 Linux 内核](https://linux.cn/article-13752-1.html)