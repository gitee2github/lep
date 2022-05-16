 Copyright (C) 2022. Huawei Technologies Co., Ltd. All rights reserved.

This program is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License version 2 and only version 2 as published by the Free Software Foundation.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

----------------------------------------

## 本文档章节如下

一、补丁说明

二、特性说明

三、补丁使用和验证方法———内核、文件系统制作和部署示例

----------------------------------------

## 一、补丁说明

相关特性配套使用Linux 2.6.34.13版本内核，在内核跟目录通过patch -p1 < xxx.patch打入补丁使用

| 补丁名 | 特性名 | 配置选项 |
| ------- | ------- | ------- |
|0001-kbox.patch	| 内核黑匣子(kbox)		|CONFIG_KBOX		|
|0002-tinysleep.patch	| 超短时间睡眠(Tinysleep)	|CONFIG_MWAIT_TINYSLEEP	|
|0003-tasklock.patch 	|用户态禁止抢占(Tasklock)	|CONFIG_RTOS_TASKLOCK	|

----------------------------------------

## 二、特性说明

### kbox内核黑匣子

#### 1 特性描述

用户在使用Linux操作系统过程中，当Linux内核在运行中出现异常时，会导致系统重启。重启时内核日志会消失，从而很难定位问题。

另外，用户在开发驱动程序时，需要记录程序运行信息和错误信息，但用户日志存储空间有限，同时不希望与内核日志混淆。

为了便于用户记录日志的同时保留内核日志，RTOS提供了内核黑匣子特性。

内核黑匣子是一种记录内核日志的机制，当系统重启或出现异常时，内核黑匣子会将异常信息写入非易失性设备上，从而支撑问题定位。

内核配置选项：CONFIG_KBOX

#### 2 限制与约束

2.1 掉电后，kbox region管理的非易失内存信息会丢失。

2.2 配置的kbox管理的内存地址不与其它模块或内核重叠。

2.3 当kbox擦除后，所有依赖kbox的特性需要重新配置region。

2.4 如果kbox分配的region ID达到64，内核中记录的regionID溢出，信息将记入panic region。

2.5 kbox用于记录OS层次的数据信息，用于定位系统异常问题。kbox同时给产品提供内核模块和用户态程序记录日志，

不涉及上层业务敏感数据；产品应根据安全要求，不能将敏感信息记录到日志。

2.6 dump_path文件在mem配置之后再配置存在数据丢失风险，如果没有配置dump_path则无kbox异常转储功能；

cmdline中配置保留内存大小场景，kbox异常转储功能直到配置dump_path才生效。

#### 3 特性应用

使能kbox存在两种方式，分别介绍如下：

3.1 使能内核黑匣子，配置内核启动参数，参数形式为：kbox_mem=size[K|M|G]@address[K|M|G] 或 kbox_mem=size[K|M|G]@address[K|M|G],def_region_size 或 kbox_mem=size[K|M|G]@address。

3.2 系统启动后配置kbox proc接口：echo address[K|M|G] size[K|M|G]> /proc/kbox/mem

#### 4 用户态接口

4.1 deviceinfo

/proc/kbox/deviceinfo：显示物理设备基本信息。

字段说明：

| 字段	| 说明 |
| ------- | ------- |
|driver name	|物理设备驱动名	|
|total		|物理设备总空间	|
|used		|已使用的空间（头控制信息+region总空间）	|
|free		|尚未使用的空间	|
|region		|region模块名	|
|start		|region在物理设备中的起始位置	|
|region		|regionsize所占空间		|
|datasize	|region中实际数据长度		|
|permission	|权限<br>read-write: 普通分区, 可读可写<br>readonly: 历史分区, 只读<br>creating: 分区创建过程中|
						
4.2 dump_path

/sys/module/kbox/parameters/dump_path：kbox格式异常时设置内存dump的路径。

当检查到kbox检测到kbox region数据异常时，将该region的信息转储到文件中，存在 dump_path设置的路径下。文件名:kbox_dump.region.region名；

该接口用于配置和查看异常转储的路径dump_path。当未配置dump_path时, kbox不保存dump的文件当配置的dump_path非有效路径时, 无法保存dump的文件。

4.3 erasure

/proc/kbox/erasure：擦除region。将所有region分区清除，包括缓冲区和物理设备，即通过执行此命令，再使能对应的region模块可实现对物理设备重新分区。

有效输入：

输入数据长度为2，第一个字符为 "1"，其他输入无效，返回错误码。

4.4 kbox_default_reg_size

/sys/module/kbox/parameters/kbox_default_reg_size：配置和查看 kbox_default_reg_size大小( 即 kbox panic 分区的大小)。

接收一个无符号数值设置kbox_default_reg_size大小

内部采用按页向上对齐, 即实际生效大小为 kbox_default_reg_size 按页向上对齐后的值；

默认值为65536

4.5 mem

/proc/kbox/mem：配置和读取保留内存地址和大小。

输入格式为“echo address[K|M|G] size[K|M|G] > /proc/kbox/mem”。

4.6 panic region

/proc/kbox/regions/panic：配置了kbox内存之后，系统默认会创建panic分区，但未指定分区时，内核日志会打到panic分区中。

通过/proc/kbox/regions/panic接口可以清空或读取panic分区记录日志。

清空命令为“echo "" > /proc/kbox/regions/panic”。

4.7 time

/sys/module/kbox/parameters/time：控制 kbox 时间戳记录功能的开启关闭。

时间格式：[20010109004536]，分别表示年月日时分秒。

输入格式为“echo 【Y/N】 > /sys/module/kbox/parameters/time”。

#### 5 内核态接口

5.1 kbox_register_region

该接口用于在内核中创建kbox的region。kbox分区数据结构如下：其中包含分区名，分区大小以及注册分区的模块，注册成功后会将分区ID记录在rfd中。

```
struct kbox_region
 {
         char name[KBOX_REGION_NAME_LEN];
         size_t size;
         struct module *mod;
         int rfd;
 };
 ```
 
5.2 kbox_write

int kbox_write(int fd, const char * buff,unsigned len)

这里传入kbox分区id（fd）即可向特定分区写入日志。


### 用户态禁止抢占 Tasklock

#### 1 简介

该特性支持在用户态禁止或者允许任务抢占的功能，用户可以借助该特性在内核的支持，实现用户态接口，禁止或允许当前任务在running状态下被抢占调度。

例如当用户需要确保一个低优先级的任务尽快完成一临界区代码，而不会由于时间片或有更改优先级任务而发生切换，则可以使用该特性。

#### 2 内核功能

配置选项：CONFIG_RTOS_TASKLOCK

实现方式：进程初始化时，内核申请进程所需共享内存；用户通过写入共享内存的标志位控制本任务的抢占；内核调度流程中，读取共享内存的标志位进行判断，选择拒绝或允许调度；进程退出时，内核释放相应共享内存。

#### 3 对外接口

  (1) /proc/sys/kernel/sched_preempt_disable
  
    功能：配置全局禁止或允许用户态禁止抢占
	
    使用：echo [输入] > /proc/sys/kernel/sched_preempt_disable
	
     - 输入0，则支持tasklock功能
	 
     - 输入非0，则不支持tasklock功能
	 
  (2) /proc/sys/kernel/sched_preempt_disable_timeout
  
    功能：配置任务禁止抢占的时间阈值（单位ms），该阈值对系统中所有使用tasklock的任务都有效
	
    使用：echo [输入] > /proc/sys/kernel/sched_preempt_disable_timeout
	
     - 输入时间阈值的数字，单位ms，例如设置为12ms则输入 12
	 
     - 当系统CPU的HZ值小于等于1000时，取值范围：[0,ULONG_MAX]
	 
     - 当系统CPU的HZ值大于1000时，取值范围为：[0,ULONG_MAX/HZ*1000]
	 
  (3) /dev/sched_ctrl
  
    功能：提供内核地址映射的接口，用户通过写入共享内存，向内核传递信息，配置当前用户任务的禁止/允许抢占状态。
	
    具体使用方法请见开发指导。

#### 4 用户开发指导

  (1) 初始化：
  
    1) 打开dev文件，例：int fd = open("/dev/sched_ctrl", O_NDELAY | O_RDWR);
	
    2) 映射一个页的共享内存，例：void *va_addr = mmap(NULL, 1024 * 4, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
	
    3) 关闭dev文件，例：close(fd);
	
    4) 为提高安全性，可以将共享内存的地址 va_addr 通过pthread_key_create、pthread_setspecific方式配置为线程私有数据
	
  (2) 实现禁止抢占：
  
    1) 在共享内存 va_addr + 8 的地址（记录禁止抢占时间戳的地址偏移）存入当前时间戳
	
    2) 在共享内存 va_addr 首地址存入1
	
  (3) 实现重新允许抢占：
  
    1) 在共享内存 va_addr 首地址存入0
	
    2) 将共享内存 va_addr + 8 的地址（记录禁止抢占时间戳的地址偏移）清为0
	
  (4) 使用：
  
    通过上述用户开发的禁止抢占和重新允许抢占的功能，标记用户程序需要禁止抢占的临界区。
	
    程序运行前注意配置 /proc/sys/kernel/sched_preempt_disable、/proc/sys/kernel/sched_preempt_disable_timeout
	
#### 5 限制与约束

  (1) 如果长时间关抢占，由于系统其它的关键任务得不到足够多的时间运行，系统的稳定性和可靠性无法保证。 
  
  (2) 关抢占超时会发送48号实时信号，进程收到该信号的默认操作是杀死进程，所以在使用tasklock的时如果收到该信号不是杀死任务则必须重新定义对48号信号的处理方式。
  
  (3) 该特性存在以下的使用限制：
  
     - 父进程使用本特性后，使用fork创建子进程，然后父进程和子进程不能同时使用本特性
	 
     - 父进程使用本特性后，使用fork创建子进程，并且使用exec刷新进程空间，此时父子进程间没有使用限制
	 
     - 父进程和线程之间，没有使用限制
	 
     - 禁止在修改系统时间的场景下使用关抢占超时功能


### 超短时间睡眠 Tinysleep

#### 1 简介

Linux操作系统中，用户态程序进行切换时，与内核的交互过程中会造成调度的开销较大，即时使用usleep(1)也可能出现开销时间远远大于1us。

该特性提供超短时间睡眠功能，通过改变用户态和内核态的通信方式，降低开销时间，提高性能。

####  2 内核功能

配置选项：CONFIG_MWAIT_TINYSLEEP

实现方式：添加新的系统调用接口，调用硬件指令 monitor/mwait 实现超短时间睡眠和唤醒，将时延降到最低。

####  3 对外接口

  (1) 启动参数 tinysleep_scheduler_interval
  
    功能：配置ktinysched内核线程调度间隔
	
    使用：配置启动参数 bootargs="tinysleep_scheduler_interval=15" 则ktinysched内核线程调度间隔为15ms
	
     - ktinysched线程负责在规定时间强制创造一个调度时机，解决可能由tinysleep引起的RCU死锁问题
	 
     - 不配置默认为10ms
	 
     - 传0则不启用ktinysched内核线程
	 
  (2) 内核接口 tinysleep_wakeup
  
    功能：从内核唤醒tinysleep，可用在内核驱动等
	
    使用：内核模块直接调用 tinysleep_wakeup(cpuid);
	
     - 输入：cpuid代表的物理核
	 
     - cpuid为unsigned int数据类型；最小值为0,最大值为系统中online的cpu的编号最大值
	 
     - 如果指定cpu不是online状态，则唤醒cpu不成功
	 
  (3) 系统调用 mwait
  
    功能：系统调用使当前核进入tinysleep睡眠状态
	
	具体使用方法请见开发指导

####  4 用户开发指导

  (1) 用户态程序：
  
  为达到最大的性能，可以通过嵌入式汇编直接进行mwait系统调用，绕开glibc的时延和系统开销。
  
  例：（以 x86_64 为例）

```
    #define TINYSLEEP_SYSCALL_NR_MWAIT 300

    inline int tinysleep(void)
    {
            int ret = 0;
            __asm__ volatile(
                    "syscall"
                    : "=a"(ret)
                    : "a" (TINYSLEEP_SYSCALL_NR_MWAIT)
            );
            return ret;
    }

    int main(void)
    {
        int ret;
        ...
        ret = tinysleep(); // 用户态进入超短时间睡眠
        ...
        return 0;
    }
```

  (2) 内核态程序：

  负责唤醒的驱动模块代码中调用唤醒接口，主动唤醒已睡眠的cpu。

  例：

```
    #include <linux/rtos/tinysleep.h>
    ...
    tinysleep_wakeup(3); //唤醒3号cpu
```

####  5 限制与约束

  (1) CPU架构应当是X86并且必须支持monitor/mwait指令

  (2) 进入睡眠的用户态进程和负责唤醒的内核线程需要绑定在不同的cpu

  (3) 对于使用tinysleep的任务，perf top查看的cpu占用率是不准确的
  
----------------------------------------

## 三、补丁使用和验证方法———内核、文件系统制作和部署示例

相关特性配套使用Linux 2.6.34.13版本内核，本文档所有的构建都基于64位ubuntu 14.04，并在x86_64的qemu平台环境下部署运行验证。

注意：由于内核版本较老，仅提供上述构建环境验证示例。

#### 1 环境准备

a. 配套内核源码下载：
```
https://git.kernel.org/scm/linux/kernel/git/stable/linux.git/snapshot/linux-2.6.34.13.tar.gz
```
b. 配套验证使用的busybox源码下载：
```
https://busybox.net/downloads/busybox-1.30.1.tar.bz2
```
c. 编译/构建环境准备：ubuntu 14.04（示例使用amd64/x86_64的ubuntu镜像，ubuntu 14.04环境自带gcc 4.8.x、glibc 2.19，兼容lliunx 2.6.34.13）

d. 编译和验证软件准备：在ubuntu 14.04下，执行：
```
sudo apt-get install build-essential libncurses5-dev qemu
```

#### 2 内核构建

a. 解压1.a中的内核源码：
```
tar xf linux-2.6.34.13.tar.gz
```
b. 到内核源码根目录，打上相关特性补丁，假设某特性补丁名为xxx_feature.patch，内核宏控名为CONFIG_XXX_FEATURE（参见补丁说明）
```
cd linux-2.6.34.13
patch -p1 < XXXXX.patch
```
c. 制作内核配置文件（基线使用内核自带默认i586配置即可），打开对应特性宏配置，根据2.b，假设为CONFIG_XXX_FEATURE
```
make i586_defconfig
make menuconfig
#进入菜单后，按"/"键入CONFIG_XXX_FEATURE进行搜索，按照提示找到对应特性开关位置，然后键入"y"使能特性，保存退出菜单
```
d. 检查.config文件，搜索CONFIG_XXX_FEATURE=y，确认已打开想要特性

e. 编译内核（有输出件arch/x86/boot/bzImage即可）：
```
make
```

#### 3 文件系统制作

a. 解压1.b中的buxybox源码：

```
tar xf busybox-1.30.1.tar.bz2
```

b. 到busybox源码目录，并成busybox默认配置（默认使用shared链接）

```
cd busybox-1.30.1
make defconfig
```

c. 使用默认配置编译安装busybox（默认输出件在_install目录）：

```
make install
```

d. 到_install目录下，进行dev配置，并从构建主机拷贝文件系统需要的相关动态库：

```
cd _install
mkdir -p ./dev
sudo mknod dev/console c 5 1
sudo mknod dev/ram b 1 0
mkdir ./lib
mkdir ./usr/lib
mkdir ./lib64
cp /lib/x86_64-linux-gnu/libresolv.so.2 ./lib
cp /usr/lib/x86_64-linux-gnu/libresolv.so ./usr/lib
cp -a /lib/x86_64-linux-gnu/libc-2.19.so ./lib
cp -a /lib/x86_64-linux-gnu/libm-2.19.so ./lib
cp -a /lib/x86_64-linux-gnu/libc.so.6 ./lib
cp -a /usr/lib/x86_64-linux-gnu/libc.so ./usr/lib
cp -a /lib/x86_64-linux-gnu/libm.so.6 ./lib
cp -a /usr/lib/x86_64-linux-gnu/libm.so ./usr/lib
cp -a /lib/x86_64-linux-gnu/ld-2.19.so ./lib
cp -a /usr/lib/x86_64-linux-gnu/libcrypt.so ./usr/lib
cp -a /lib/x86_64-linux-gnu/libcrypt.so.1 ./lib
cp -a /lib/x86_64-linux-gnu/libcrypt-2.19.so ./lib
cp -a /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 ./lib64
cp -a /lib/x86_64-linux-gnu/ld-2.19.so ./lib64
cp -a /lib/x86_64-linux-gnu/libdl-2.19.so ./lib
cp -a /lib/x86_64-linux-gnu/libdl.so.2 ./lib
cp -a /lib/x86_64-linux-gnu/libpthread.so.0 ./lib
cp -a /lib/x86_64-linux-gnu/libpthread-2.19.so ./lib
cp -a /lib/x86_64-linux-gnu/libutil.so.1 ./lib
cp -a /lib/x86_64-linux-gnu/libutil-2.19.so ./lib
cp -a /lib/x86_64-linux-gnu/libgcc_s.so.1 ./lib
cp -a /usr/lib/x86_64-linux-gnu/libssl* ./usr/lib
cp -a /lib/x86_64-linux-gnu/libssl.so.1.0.0 ./lib
cp -a /usr/lib/x86_64-linux-gnu/libseccomp.so* ./usr/lib
cp -a /usr/lib/x86_64-linux-gnu/libevent-2.0.so* ./usr/lib
cp -a /lib/x86_64-linux-gnu/libz.so* ./lib
cp -a /usr/lib/x86_64-linux-gnu/librt.so ./usr/lib
cp -a /lib/x86_64-linux-gnu/librt-2.19.so ./lib
cp -a /usr/lib/x86_64-linux-gnu/librt.so ./usr/lib
```

e. 最后制作简易启动脚本，在_install目录下，创建文件init，键入如下内容:

```
#!/bin/sh
echo "########Ready########"
mkdir /proc /sys /tmp
mount -t proc none /proc
mount -t sysfs none /sys
mount -t tmpfs none /tmp
echo -e "\nThis boot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
exec /bin/sh
```
注意使用如下命令赋于可执行权限:
```
chmod +x ./init
```
d. 将其他你需要的内容放入_install后，进行打包：
```
find -print0 | cpio -0oH newc | gzip -9 > ~/initramfs.cpio.gz
```

#### 4 QEMU部署
```
qemu-system-x86_64 -kernel linux-2.6.34.13/arch/x86/boot/bzImage -initrd ~/initramfs.cpio.gz --append "console=ttyS0 root=/dev/ram init=/init" -nographic
```
