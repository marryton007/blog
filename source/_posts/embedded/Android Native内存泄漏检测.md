---
title: "Android Native内存泄漏检测"
date: 2022-02-22 15:34:00
tags:
  - Android
  - linux
  - memory leak
  - hook
categories:
  - embedded
---

## 1. 硬件平台

RK3568开发板， Android11

Android Studio 2020.3.1 patch 3 

## 2. 背景

一次例行检查中发现公司产品Bugly平台上报了一个java频繁调用JNI，最终出现OOM，导致系统崩溃重启的问题，为了将此类问题扼杀在开发阶段，特记录本次测试过程，防止后续重复踩坑。在调查过程中发现，Android针对JAVA层的内存泄漏检查相对比较成熟，但对于native层的泄漏问题支持得不是很好。

## 3. 技术方案选择

* [Android 使用 valgrind][valgrind]
* [Address Sanitizer (aka ASan)][asan]
* [Malloc Debug][malloc_debug]
* [字节跳动raphael][raphael]

## 4. 结论

经过几天尝试，最终选用[字节跳动raphael][raphael]方案，这也是现在唯一能找到比较好的开源方案。其工作原理请参阅参考资料，使用说明请见Github网站。

## 5. 参考资料

[西瓜视频稳定性治理体系建设二：Raphael 原理及实践](https://mp.weixin.qq.com/s/RF3m9_v5bYTYbwY-d1RloQ)

[全民K歌Android端Native内存分析与监控方案实践总结](https://zhuanlan.zhihu.com/p/371527210)

[爱奇艺xHook](https://github.com/iqiyi/xHook)



[asan]: https://developer.android.com/ndk/guides/asan
[valgrind]: https://source.android.google.cn/devices/tech/debug/valgrind?hl=zh-cn
[malloc_debug]: https://android.googlesource.com/platform/bionic/+/master/libc/malloc_debug/README.md
[raphael]: https://github.com/bytedance/memory-leak-detector

