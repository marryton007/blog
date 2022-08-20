---
title: "自制Android启动动画"
date: 2021-12-22 10:28:00
tags:
  - Android
  - linux
categories:
  - embedded
---

## 1. 背景

RK3568开发板，  Android 11

## 2. 目的

记录开发过程中遇到的问题，防止重复踩坑

## 3. 素材准备

* 新建工作目录，目录下建立如下内容

* 多个子目录
    用于存放图片，如part01，part02等
    part01，一般放开机动画的第一张图片，目录只放一张图片
    part02， 放循环的图片，多张图片形成视频的效果
* desc.txt，是动画的规则

* desc.txt文件格式说明

```bash
1280 800 25
P   1  0 part01
P   0  0 part02   
```

第一行，1280   800  是分辨率，25是帧数
第二行，1 是循环一次，0是间隔时间，part01是对应目录
第二行，0 是无限循环，0是间隔时间，part02是对应目录

## 3. 打包

选择工作目录中的所有内容，使用压缩工具打包

![image-20211013104739133](_media/image-20211013104739133.png)

注意打包时的选项

![image-20211013105005744](_media/image-20211013105005744.png)

这里**压缩文件的名称必须为`bootanimation.zip`，压缩等级必须选`存储`，**不然开机动画无法正常工作。

## 5. 测试

在开发板上执行如下命令进行测试：

```bash
//进入ADB，建立/odm/media目录，且保证/odm可写入；如果已做过这一步，可跳过
adb shell
mount -o remount,rw /odm
mkdir -p /odm/media
exit

//上传制作好的压缩包到/odm/media目录
adb push bootanimation.zip /odm/media

//快速测试，查看开机动画
adb shell setprop service.bootanim.exit 0
adb shell setprop ctl.start bootanim

//停止动画
adb shell setprop service.bootanim.exit 1
```

## 6. 提交

测试通过后，替换RK3568 Android SDK中的同名文件，文件位于：

```bash
device/rockchip/common/bootanimation.zip
```
