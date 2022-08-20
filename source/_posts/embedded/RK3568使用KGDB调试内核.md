---
title: "RK3568使用KGDB调试内核"
date: 2022-03-15 16:24:00
tags:
  - Android
  - linux
  - kgdb
categories:
  - embedded
---

## 1. 开发环境

开发板： RK3568 evb1
内核版本： 4.19.172
开发机： Windows 10 + VirtualBox VM ubuntu20.04
编译服务器： ubuntu18.04 

## 2. 背景

在浏览内核代码、开发内核功能模块的过程，有时需要跟踪内核执行流程，常用的手段有：

- printk/pr_xxx  添加打印语句，需要重新编译模块
- dynamic print， 动态调试，在运行时启用或停用模块的调试功能，无须重新编译模块
- kgdb调试，可以对内核设置断点，或进行单步跟踪，使用方式比较友好

本文主要介绍最后一种，kgdb 是Linux内核提供一个gdb server，远程gdb可以通过串口或网络来连接，大致过程如下：

- 编译支持kgdb的内核，并烧录到开发板上
- 开发机通过串口连接到开发板
- 开发机上通过gdb连接到开发板上kgdb，开始调试

## 3. 目的

记录开发过程，为后续开发和使用提供资料，防止重复踩坑

## 4. 开发板配置

### 4.1 配置内核支持kgdb

在内核配置文件.config中， 需要确认如下选项：

| CONFIG_KGDB                | 加入KGDB支持                                                 |
| -------------------------- | ------------------------------------------------------------ |
| CONFIG_KGDB_SERIAL_CONSOLE | 使KGDB通过串口与主机通信(打开这个选项，默认会打开CONFIG_CONSOLE_POLL和CONFIG_MAGIC_SYSRQ) |
| CONFIG_KGDB_KDB            | 加入KDB支持                                                  |
| CONFIG_DEBUG_KERNEL        | 包含驱动调试信息                                             |
| CONFIG_DEBUG_INFO          | 使内核包含基本调试信息                                       |
| CONFIG_STRICT_KERNEL_RWX=n | 可选，关闭这个，能在只读区域设置断点                         |

可选选项：

| CONFIG_FRAME_POINTER | 使KDB能够打印更多的栈信息                                    |
| -------------------- | ------------------------------------------------------------ |
| CONFIG_KALLSYMS      | 加入符号信息                                                 |
| CONFIG_KDB_KEYBOARD  | 如果是通过目标版的键盘与KDB通信，需要把这个打开，且键盘不能是USB接口 |
| CONFIG_KGDB_TESTS    |                                                              |
| CONFIG_GDB_SCRIPTS=y | 内核提供的GDB脚本                                            |

### 4.2 完善kgdb

远端的 gdb 连上 linux 的 kgdb 之后，在断点处执行单步调式（step/next）的时候，调式器并不是执行断点处的语句，而是每次都陷入到下面的代码段：

```c
arch/arm64/kernel/entry.S：356
el1_irq:
        kernel_entry 1
        enable_dbg
```

社区也有人碰到这个问题，并提交了如下 patch 来修复这个问题。这个问题并不是在所有的 arm64 平台上都会碰到，patch 还在讨论并没有合进 upstream，Patch 如下:

> https://patchwork.kernel.org/project/linux-arm-kernel/patch/20170523043058.5463-3-takahiro.akashi@linaro.org/

上面的网页里可以下载补丁文件，不过需要稍做修改才能在Linux-4.19.172工作。

```diff
diff --git a/kernel/arch/arm64/kernel/kgdb.c b/kernel/arch/arm64/kernel/kgdb.c
index 8815b5457d..e3c11e3d0f 100644
--- a/kernel/arch/arm64/kernel/kgdb.c
+++ b/kernel/arch/arm64/kernel/kgdb.c
@@ -28,6 +28,7 @@
 
 #include <asm/debug-monitors.h>
 #include <asm/insn.h>
+#include <asm/ptrace.h>
 #include <asm/traps.h>
 
 struct dbg_reg_def_t dbg_reg_def[DBG_MAX_REG_NUM] = {
@@ -111,6 +112,8 @@ struct dbg_reg_def_t dbg_reg_def[DBG_MAX_REG_NUM] = {
        { "fpcr", 4, -1 },
 };
 
+static DEFINE_PER_CPU(unsigned int, kgdb_pstate);
+
 char *dbg_get_reg(int regno, void *mem, struct pt_regs *regs)
 {
        if (regno >= DBG_MAX_REG_NUM || regno < 0)
@@ -217,6 +220,10 @@ int kgdb_arch_handle_exception(int exception_vector, int signo,
                err = 0;
                break;
        case 's':
+               /* mask interrupts while single stepping */
+               __this_cpu_write(kgdb_pstate, linux_regs->pstate);
+               linux_regs->pstate |= PSR_I_BIT;
+
                /*
                 * Update step address value with address passed
                 * with step packet.
@@ -266,11 +273,22 @@ NOKPROBE_SYMBOL(kgdb_compiled_brk_fn);
 
 static int kgdb_step_brk_fn(struct pt_regs *regs, unsigned int esr)
 {
-       if (user_mode(regs) || !kgdb_single_step)
-               return DBG_HOOK_ERROR;
+  unsigned int pstate;
 
-       kgdb_handle_exception(0, SIGTRAP, 0, regs);
-       return DBG_HOOK_HANDLED;
+  if (user_mode(regs) || !kgdb_single_step)
+    return DBG_HOOK_ERROR;
+
+  kernel_disable_single_step();
+  /* restore interrupt mask status */
+  pstate = __this_cpu_read(kgdb_pstate);
+  if (pstate & PSR_I_BIT)
+    regs->pstate |= PSR_I_BIT;
+  else
+    regs->pstate &= ~PSR_I_BIT;
+
+
+  kgdb_handle_exception(0, SIGTRAP, 0, regs);
+  return DBG_HOOK_HANDLED;
 }
 NOKPROBE_SYMBOL(kgdb_step_brk_fn);
```

### 4.3 启动开发板

将编译好的内核烧录至开发板，通过ADB连接或控制台输入如下命令

```c
echo ttyFIQ0 > /sys/module/kgdboc/parameters/kgdboc
echo g > /proc/sysrq-trigger
```

其中ttyFIQ0是开发机和开发板相连的串口。

sysrq-trigger是Linux魔术键，发送不同的键值可引发内核不同操作，g就是进行kgdb模式。执行完最后一句后，开发板进入kgdb模块，这里按任何键都没有反应，内核等待接收gdb调试请求。

## 5. 开发机配置

接下来的操作需要使用Linux环境，但由于平时使用window10工作，这里配置一台VirtualBox虚拟机，运行是Ubuntu20.04，后续工作都在虚拟机里进行。

### 5.1 影射USB串口到虚拟机

![image-20211022152722533](_media/image-20211022152722533.png)

### 5.2 安装工具

在x86平台上调试arm64文件不能使用平时的gdb，要安装gdb-multiarch。

```shell
sudo apt install gdb-multiarch
```

所有的内核源码，编译出来的内核镜像文件都在编译服务器上，要将服务器上的目录影射到虚拟机中来，可以使用NFS，这里选择使用sshfs

```shell
sudo apt install sshfs
mkdir /opt/sdk/rongping/android-20210804
# mount remote filesystem with sshfs
sshfs xxx@192.168.60.252:/opt/sdk/rongping/android-20210804 /opt/sdk/rongping/android-20210804
```

### 5.3 开始调试

```shell
cd /opt/sdk/rongping/android-20210804/kernel
# vmlinux是带调试信息的内核镜像
gdb-multiarch ./vmlinux
# 设备目标平台
set architecture aarch64
# 设置串口波特率
set serial baud 115200
# 通过串口连接到开发上的kgdb
target remote /dev/ttyUSB0
```

一切正常的话，会有如下提示：

```shell
vagrant@vagrant:/opt/sdk/rongping/android-20210804/kernel$ gdb-multiarch ./vmlinux
GNU gdb (Ubuntu 9.2-0ubuntu1~20.04) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./vmlinux...
warning: File "/opt/sdk/rongping/android-20210804/kernel/scripts/gdb/vmlinux-gdb.py" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
        add-auto-load-safe-path /opt/sdk/rongping/android-20210804/kernel/scripts/gdb/vmlinux-gdb.py
line to your configuration file "/home/vagrant/.gdbinit".
To completely disable this security protection add
        set auto-load safe-path /
line to your configuration file "/home/vagrant/.gdbinit".
For more information about this security protection see the
"Auto-loading safe path" section in the GDB manual.  E.g., run from the shell:
        info "(gdb)Auto-loading safe path"
(gdb) set architecture aarch64
The target architecture is assumed to be aarch64
(gdb) set serial baud 115200
(gdb) target remote /dev/ttyUSB0
Remote debugging using /dev/ttyUSB0
arch_kgdb_breakpoint () at ./arch/arm64/include/asm/kgdb.h:32
32              asm ("brk %0" : : "I" (KGDB_COMPILED_DBG_BRK_IMM));
```

设置断点，并继续运行

```
(gdb) b rockchip_drm_summary_show
Breakpoint 1 at 0xffffff8008647ba0: file drivers/gpu/drm/rockchip/rockchip_drm_drv.c, line 1346.
(gdb) c
Continuing.
```

回到开发板，这时发现又可以输入命令了

```shell
rk3568_r:/ # cat /d/dri/0/summary
```

再次回到虚拟机，发现刚才设置的断点已经触发了，并可以使用`n,s,p,c`等命令进行调试

```shell
[New Thread 1896]
[New Thread 1897]
[New Thread 1898]
--Type <RET> for more, q to quit, c to continue without paging--
[Switching to Thread 1919]

Thread 1058 hit Breakpoint 1, rockchip_drm_summary_show (s=0xffffffc03a927000, data=0x1) at drivers/gpu/drm/rockchip/rockchip_drm_drv.c:1346
1346    {
(gdb) n
1348            struct drm_minor *minor = node->minor;
(gdb) n
1349            struct drm_device *drm_dev = minor->dev;
(gdb) 
1353            drm_for_each_crtc(crtc, drm_dev) {
(gdb) s
1350            struct rockchip_drm_private *priv = drm_dev->dev_private;
(gdb) n
1353            drm_for_each_crtc(crtc, drm_dev) {
(gdb) l
1348            struct drm_minor *minor = node->minor;
1349            struct drm_device *drm_dev = minor->dev;
1350            struct rockchip_drm_private *priv = drm_dev->dev_private;
1351            struct drm_crtc *crtc;
1352
1353            drm_for_each_crtc(crtc, drm_dev) {
1354                    int pipe = drm_crtc_index(crtc);
1355
1356                    if (priv->crtc_funcs[pipe] &&
1357                        priv->crtc_funcs[pipe]->debugfs_dump)
(gdb) n
1356                    if (priv->crtc_funcs[pipe] &&
(gdb) 
1358                            priv->crtc_funcs[pipe]->debugfs_dump(crtc, s);
(gdb) 
1356                    if (priv->crtc_funcs[pipe] &&
(gdb) 
1357                        priv->crtc_funcs[pipe]->debugfs_dump)
(gdb) 
1356                    if (priv->crtc_funcs[pipe] &&
(gdb) 
1358                            priv->crtc_funcs[pipe]->debugfs_dump(crtc, s);
(gdb) 
1353            drm_for_each_crtc(crtc, drm_dev) {
(gdb) 
1356                    if (priv->crtc_funcs[pipe] &&
(gdb) p crtc
$1 = (struct drm_crtc *) 0xffffffc07a650948
(gdb) p *crtc
$2 = {dev = 0xffffffc07a532000, port = 0xffffffc07ff96618, head = {next = 0xffffffc07a651208, prev = 0xffffffc07a6500a8}, name = 0xffffffc07a63e180 "video_port1", mutex = {mutex = {base = {owner = {counter = 0}, wait_lock = {{
            rlock = {raw_lock = {{val = {counter = 0}, {locked = 0 '\000', pending = 0 '\000'}, {locked_pending = 0, tail = 0}}}}}}, osq = {tail = {counter = 0}}, wait_list = {next = 0xffffffc07a650980, prev = 0xffffffc07a650980}}, 
      ctx = 0x0}, head = {next = 0xffffffc07a650998, prev = 0xffffffc07a650998}}, base = {id = 85, type = 3435973836, properties = 0xffffffc07a650bf0, refcount = {refcount = {refs = {counter = 0}}}, free_cb = 0x0}, 
  primary = 0xffffffc07a657278, cursor = 0x0, index = 1, cursor_x = 0, cursor_y = 0, enabled = true, mode = {head = {next = 0x0, prev = 0x0}, base = {id = 0, type = 0, properties = 0x0, refcount = {refcount = {refs = {counter = 0}}}, 
      free_cb = 0x0}, name = "800x1280", '\000' <repeats 23 times>, status = MODE_OK, type = 72, clock = 67730, hdisplay = 800, hsync_start = 818, hsync_end = 836, htotal = 854, hskew = 0, vdisplay = 1280, vsync_start = 1300, 
    vsync_end = 1304, vtotal = 1314, vscan = 0, flags = 10, width_mm = 0, height_mm = 0, crtc_clock = 67730, crtc_hdisplay = 800, crtc_hblank_start = 800, crtc_hblank_end = 854, crtc_hsync_start = 818, crtc_hsync_end = 836, 
    crtc_htotal = 854, crtc_hskew = 0, crtc_vdisplay = 1280, crtc_vblank_start = 1280, crtc_vblank_end = 1314, crtc_vsync_start = 1300, crtc_vsync_end = 1304, crtc_vtotal = 1314, private = 0x0, private_flags = 0, vrefresh = 60, 
    hsync = 0, picture_aspect_ratio = HDMI_PICTURE_ASPECT_NONE, export_head = {next = 0x0, prev = 0x0}}, hwmode = {head = {next = 0x0, prev = 0x0}, base = {id = 0, type = 0, properties = 0x0, refcount = {refcount = {refs = {
            counter = 0}}}, free_cb = 0x0}, name = '\000' <repeats 31 times>, status = MODE_OK, type = 0, clock = 0, hdisplay = 0, hsync_start = 0, hsync_end = 0, htotal = 0, hskew = 0, vdisplay = 0, vsync_start = 0, vsync_end = 0, 
    vtotal = 0, vscan = 0, flags = 0, width_mm = 0, height_mm = 0, crtc_clock = 0, crtc_hdisplay = 0, crtc_hblank_start = 0, crtc_hblank_end = 0, crtc_hsync_start = 0, crtc_hsync_end = 0, crtc_htotal = 0, crtc_hskew = 0, 
    crtc_vdisplay = 0, crtc_vblank_start = 0, crtc_vblank_end = 0, crtc_vsync_start = 0, crtc_vsync_end = 0, crtc_vtotal = 0, private = 0x0, private_flags = 0, vrefresh = 0, hsync = 0, picture_aspect_ratio = HDMI_PICTURE_ASPECT_NONE, 
    export_head = {next = 0x0, prev = 0x0}}, x = 0, y = 0, funcs = 0xffffff8009371420 <vop2_crtc_funcs>, gamma_size = 1024, gamma_store = 0xffffffc07a6a4000, helper_private = 0xffffff80093714d8 <vop2_crtc_helper_funcs>, properties = {
    count = 13, properties = {0xffffffc07a636e00, 0xffffffc07a636f00, 0xffffffc07a636c00, 0xffffffc07a638f80, 0xffffffc07a63a080, 0xffffffc07a63a180, 0xffffffc07a638400, 0xffffffc07a638500, 0xffffffc07a638600, 0xffffffc07a638700, 
      0xffffffc07a63e280, 0xffffffc07a637180, 0xffffffc07a637200, 0x0 <repeats 51 times>}, values = {0, 0, 0, 218762, 1, 0, 100, 100, 100, 100, 63, 0, 1024, 0 <repeats 51 times>}}, state = 0xffffffc036a34000, commit_list = {
    next = 0xffffffc07a651000, prev = 0xffffffc07a651000}, commit_lock = {{rlock = {raw_lock = {{val = {counter = 0}, {locked = 0 '\000', pending = 0 '\000'}, {locked_pending = 0, tail = 0}}}}}}, debugfs_entry = 0xffffffc079ac25b0, 
  crc = {lock = {{rlock = {raw_lock = {{val = {counter = 0}, {locked = 0 '\000', pending = 0 '\000'}, {locked_pending = 0, tail = 0}}}}}}, source = 0xffffffc07a63e200 "auto", opened = false, overflow = false, entries = 0x0, head = 0, 
    tail = 0, values_cnt = 0, wq = {lock = {{rlock = {raw_lock = {{val = {counter = 0}, {locked = 0 '\000', pending = 0 '\000'}, {locked_pending = 0, tail = 0}}}}}}, head = {next = 0xffffffc07a651058, prev = 0xffffffc07a651058}}}, 
  fence_context = 5, fence_lock = {{rlock = {raw_lock = {{val = {counter = 0}, {locked = 0 '\000', pending = 0 '\000'}, {locked_pending = 0, tail = 0}}}}}}, fence_seqno = 0, 
  timeline_name = "CRTC:85-video_port1", '\000' <repeats 12 times>}
(gdb) finish
Run till exit from #0  rockchip_drm_summary_show (s=0xffffffc03a927000, data=<optimized out>) at drivers/gpu/drm/rockchip/rockchip_drm_drv.c:1356
[New Thread 1920]
0xffffff8008292718 in seq_read (file=0xffffffc034f81100, buf=0x5eca018818 <error: Cannot access memory at address 0x5eca018818>, size=4096, ppos=0xffffff801094be60) at fs/seq_file.c:229
229                     err = m->op->show(m, p);
Value returned is $3 = 0
(gdb) c
Continuing.
```



## 6. 参考资料

[用 kGDB 调试 Linux 内核](http://tinylab.org/kgdb-debugging-kernel/)

[如何使用Kgdb调试内核源码？](https://www.ivdone.top/article/872.html#1-2)

[如何在x86架构Linux上使用qemu+gdb调试aarch64的内核](https://zhuanlan.zhihu.com/p/47783910)

[kgdb调试aarch64内核模块](https://zhuanlan.zhihu.com/p/197545583)

